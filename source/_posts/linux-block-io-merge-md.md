---
title: Linux Block 模块之 I/O 合并代码解析
date: 2024-07-27 13:20:59
tags: block
categories: Linux Block
description: 详细分析 Linux Block 层 I/O 合并的代码
---

## 1. I/O 路径

从内核角度看，进程产生的 I/O 路径主要有三条：

- 缓存 I/O：系统绝大部分 I/O 走的这种形式，充分利用文件系统层的 page cache 所带来的优势。应用程序产生的 I/O 经系统调用落入 page cache 之后便可以直接返回，page cache 中的缓存数据由内核回写线程在适当时机同步到底层的存储介质上，当然应用程序也可以主动发起回写过程（如 fsync 系统调用）来确保数据尽快同步打扫存储介质上，从而避免系统崩溃或者掉电带来的数据不一致性；
- 非缓存 I/O（带蓄流）：这种 I/O 绕过文件系统层的 cache。用户在打开要读写的文件的时候需要加上 `O_DIRECT` 标记，意为直接 I/O，不让文件系统的 page cache 介入。从用户角度而言，应用程序能直接控制的 I/O 形式除了上面提到的缓存 I/O，剩下的 I/O 都走这种形式，就算打开文件时加上了 `O_SYNC` 标记，最终产生的 I/O 也会进入蓄流链表。如果应用程序在用户空间自己做了缓存，那么就可以使用这种 I/O 方式，常见的如数据库应用；
- 非缓存 I/O（不带蓄流）：内核通用快层的蓄流机制只给内核空间提供了接口来控制 I/O 请求是否蓄流，用户空间进程无法控制提交的 I/O 请求进入通用快层的时候是否蓄流。严格的说，用户空间直接产生的 I/O 都会走蓄流路径，哪怕是提交 I/O 的时候加上了 `O_DIRECT` 和 `O_SYNC` 标记。用户空间间接产生的 I/O，如文件系统日志数据、元数据，有的不会走蓄流路径而是直接进入调度队列尽快得到调度。注意一点，通用块层的蓄流只是提供机制和接口而不是策略，也就是需不需要蓄流、何时蓄流完全由内核中的 I/O 派发者决定；

## 2. 合并方式

应用程序无论使用图中哪条 I/O 路径，内核都会想方设法对 I/O 进行合并。内核为促进这种合并，在 I/O 栈上设置了三种合并方式以对应以上三种 I/O 路径：

- Cache（页高速缓存）
- Plug list（蓄流链表）
- Elevator Queue（调度队列）

其中 Cache 属于文件系统层的内容，本文中不讨论，我们主要分析另外两种 I/O 合并方式。

### 2.1. 逻辑入口

`blk_queue_io()` 函数是通用块层进行 I/O 合并的入口函数，该函数内先尝试 Plug 合并，合并成功返回，合并失败则继续尝试 Elevator 合并，合并成功返回，合并失败则为 bio 创建一个新的 request 插入到调度队列中，尝试 I/O 合并部分的代码如下：

```C++
void blk_queue_io(struct request_queue *q, struct bio *bio)
{
        const bool sync = !!(bio->bi_rw & REQ_SYNC);
        struct blk_plug *plug;
        int el_ret, rw_flags, where = ELEVATOR_INSERT_SORT;
        struct request *req, *free;
        unsigned int request_count = 0;

        /*
         * low level driver can indicate that it wants pages above a
         * certain limit bounced to low memory (ie for highmem, or even
         * ISA dma in theory)
         *
         * 做DMA时的相关地址限制，可能该bio只能访问低端内存，
         * 因此需要将高端内存中的bio数据拷贝到低端内存中
         */
        blk_queue_bounce(q, &bio);

        /* 完整性校验 */
        if (bio_integrity_enabled(bio) && bio_integrity_prep(bio)) {
                bio_endio(bio, -EIO);
                return;
        }

        /* FLUSH和FUA直接生成新的请求处理 */
        if (bio->bi_rw & (REQ_FLUSH | REQ_FUA)) {
                spin_lock_irq(q->queue_lock);
                where = ELEVATOR_INSERT_FLUSH;
                goto get_rq;
        }

        /*
         * Check if we can merge with the plugged list before grabbing
         * any locks.
         *
         * 先尝试plug合并，plug中为当前进程的req链表，合并成功直接返回
         */
        if (!blk_queue_nomerges(q)) {
                if (blk_attempt_plug_merge(q, bio, &request_count, NULL))
                        return;
        } else
                request_count = blk_plug_queued_count(q);

        /* 不管是与队列中的请求合并还是插入新的请求都需要加锁 */
        spin_lock_irq(q->queue_lock);

        /* 这里记录下电梯哈希和调度算法调度队列
         * bio生成新的请求后，会插入两个地方
         * 1.电梯的通用哈希表，以请求的结束为止为哈希值进行哈希，
         *   便于查找可以向后合并的请求
         *   q->elevator->hash
         * 2.具体调度算法的调度队列，用于调度算法进行调度，以deadline为例
         *   q->elevator->elevator_data->sort_list/fifo_expire
         *
         * 调度算法派发请求后，请求会进入q的派发队列，
         *   同时从哈希和调度队列中移除
         */

        /* 在请求队列的哈希表中查找可以向后合并的请求，在调度算法
         * 的调度队列中查找可以合并的请求（deadline算法中只有向前合并）
         */
        el_ret = elv_merge(q, &req, bio);
        if (el_ret == ELEVATOR_BACK_MERGE) {
                /* 尝试进行bio与req的合并 */
                if (bio_attempt_back_merge(q, req, bio)) {
                        /* 合并成功的话，调用具体调度算法的后续处理（deadline中并没有实现接口） */
                        elv_bio_merged(q, req, bio);
                        /* 这次合并的bio可能会弥补两个bio之间的空隙，所以这里查找根据
                         * 给定req后边的一个请求，判断能否进行合并
                         */
                        free = attempt_back_merge(q, req);
                        if (!free)
                                /* 如果上边没有检测到可以合并的request，则调用接口处理
                                 * bio合并到request之后的处理：
                                 *   1.调用调度算法的回调函数处理调度算法需要执行的操作
                                 *   2.如果是向后合并还需要更新电梯哈希
                                 */
                                elv_merged_request(q, req, el_ret);
                        else
                                __blk_put_request(q, free);
                        goto out_unlock;
                }
        } else if (el_ret == ELEVATOR_FRONT_MERGE) {
                if (bio_attempt_front_merge(q, req, bio)) {
                        elv_bio_merged(q, req, bio);
                        free = attempt_front_merge(q, req);
                        if (!free)
                                elv_merged_request(q, req, el_ret);
                        else
                                __blk_put_request(q, free);
                        goto out_unlock;
                }
        }

get_rq:
        /* bio无法合并，为其申请一个新的request */
        /*
         * This sync check and mask will be re-done in init_request_from_bio(),
         * but we need to set it earlier to expose the sync flag to the
         * rq allocator and io schedulers.
         */
        rw_flags = bio_data_dir(bio);
        if (sync)
                rw_flags |= REQ_SYNC;

        /*
         * Grab a free request. This is might sleep but can not fail.
         * Returns with the queue unlocked.
         */
        blk_queue_enter_live(q);
        /* 获取空闲的请求 */
        req = get_request(q, rw_flags, bio, 0);
        if (IS_ERR(req)) {
                blk_queue_exit(q);
                bio_endio(bio, PTR_ERR(req));        /* @q is dead */
                goto out_unlock;
        }

        /*
         * After dropping the lock and possibly sleeping here, our request
         * may now be mergeable after it had proven unmergeable (above).
         * We don't worry about that case for efficiency. It won't happen
         * often, and the elevators are able to handle it.
         *
         * 根据bio初始化request
         */
        init_request_from_bio(req, bio);

        if (test_bit(QUEUE_FLAG_SAME_COMP, &q->queue_flags))
                req->cpu = raw_smp_processor_id();

        /* 如果当前进程有plug_list，首先插入到plug_list中 */
        plug = current->plug;
        if (plug) {
                /*
                 * If this is the first request added after a plug, fire
                 * of a plug trace.
                 *
                 * @request_count may become stale because of schedule
                 * out, so check plug list again.
                 */
                if (!request_count || list_empty(&plug->list))
                        trace_block_plug(q);
                else {
                        struct request *last = list_entry_rq(plug->list.prev);
                        if (request_count >= BLK_MAX_REQUEST_COUNT ||
                            blk_rq_bytes(last) >= BLK_PLUG_FLUSH_SIZE) {
                                blk_flush_plug_list(plug, false);
                                trace_block_plug(q);
                        }
                }
                /* 插入到plug链表 */
                list_add_tail(&req->queuelist, &plug->list);
                blk_account_io_start(req, true);
        } else {
                spin_lock_irq(q->queue_lock);
                /* 插入到电梯及调度算法中 */
                add_acct_request(q, req, where);
                /* 执行q->request_fn(q)，调用底层驱动的策略例程处理请求。
                 * 以SCSI为例，request_fn初始化为scsi_request_fn
                 * scsi_request_fn->blk_peek_request->__elv_next_request
                 * __elv_next_request中会使用调度具体算法的
                 * dispatch回调函数取出请求进行处理
                 */
                __blk_run_queue(q);
out_unlock:
                spin_unlock_irq(q->queue_lock);
        }
}
```

### 2.2. Plug 合并

Plug 合并是内核通用块层提供的机制，如果需要开启则在提交 I/O 前执行 `blk_start_plug()` 函数启动蓄流，IO 全部提交完成后执行 `blk_finish_plug()` 函数进行泄流。启动蓄流主要是为当前进程（ `current` 宏）分配 plug 链表，提交 I/O 时 plug 链表存在则 I/O 先缓存到该链表中，执行泄流时将该链表中缓存的 I/O 下发。执行 plug 合并的函数为 `blk_attempt_plug_merge()` ，该函数的主要逻辑为遍历 `current->plug->list` 中的所有请求，将属于同一个底层设备队列且位置相邻的请求合并在一起，代码逻辑如下：

```C++
bool blk_attempt_plug_merge(struct request_queue *q, struct bio *bio,
                            unsigned int *request_count,
                            struct request **same_queue_rq)
{
        struct blk_plug *plug;
        struct request *rq;
        bool ret = false;
        struct list_head *plug_list;

        plug = current->plug;
        if (!plug)
                goto out;
        *request_count = 0;

        if (q->mq_ops)
                plug_list = &plug->mq_list;
        else
                plug_list = &plug->list;

        list_for_each_entry_reverse(rq, plug_list, queuelist) {
                int el_ret;

                /* 给plug_list中同一个设备的请求计数 */
                if (rq->q == q) {
                        (*request_count)++;
                        /*
                         * Only blk-mq multiple hardware queues case checks the
                         * rq in the same queue, there should be only one such
                         * rq in a queue
                         **/
                        if (same_queue_rq)
                                *same_queue_rq = rq;
                }

                /* 同一队列，满足合并的才能合并 */
                if (rq->q != q || !blk_rq_merge_ok(rq, bio))
                        continue;

                /* 判断能进行合并的位置 */
                el_ret = blk_try_merge(rq, bio);
                if (el_ret == ELEVATOR_BACK_MERGE) {
                        ret = bio_attempt_back_merge(q, rq, bio);
                        if (ret)
                                break;
                } else if (el_ret == ELEVATOR_FRONT_MERGE) {
                        ret = bio_attempt_front_merge(q, rq, bio);
                        if (ret)
                                break;
                }
        }
out:
        return ret;
}
```

该逻辑中有两点需要注意。一是 `request_count` 计数，该变量统计 plug 链表中与当前 I/O 属于同一队列的请求个数，当数量超过 `BLK_MAX_REQUEST_COUNT` （当前定义为 16）时则主动下刷 plug 中的请求；二是函数 `blk_rq_merge_ok()` 与函数 `blk_try_merge()` ，其中 `blk_rq_merge_ok()` 检查 req 及 bio 属性是否支持合并， `blk_try_merge()` 则是根据 req 及 bio 的起始位置与长度来检查是否相邻，以判断能否合并以及合并的类型（前向/后向），代码如下：

```C++
bool blk_rq_merge_ok(struct request *rq, struct bio *bio)
{
        /* rq_mergeable - 判断rq是否有不允许合并的标记
         * bio_mergeable - 判断bio是否有不允许合并的标记
         */
        if (!rq_mergeable(rq) || !bio_mergeable(bio))
                return false;

        /* 判断rq和bio是否是特殊的请求，
         * 如果是特殊请求并且两者相同则不允许合并
         */
        if (!blk_check_merge_flags(rq->cmd_flags, bio->bi_rw))
                return false;

        /* different data direction or already started, don't merge */
        /* 请求方向不同，不能合并 */
        if (bio_data_dir(bio) != rq_data_dir(rq))
                return false;

        /* must be same device and not a special request */
        /* 指向的通用磁盘不同或者rq是特殊请求，不能合并 */
        if (rq->rq_disk != bio->bi_bdev->bd_disk || req_no_special_merge(rq))
                return false;

        /* only merge integrity protected bio into ditto rq */
        /* 完整性保护不同的请求和bio，不能合并 */
        if (bio_integrity(bio) != blk_integrity_rq(rq))
                return false;

        /* must be using the same buffer */
        /* REQ_WRITE_SAME请求，来自不同的page，不能合并 */
        if (rq->cmd_flags & REQ_WRITE_SAME &&
            !blk_write_same_mergeable(rq->bio, bio))
                return false;

        return true;
}

int blk_try_merge(struct request *rq, struct bio *bio)
{
        /* rq结束位置等于bio起始位置返回后向合并 */
        if (blk_rq_pos(rq) + blk_rq_sectors(rq) == bio->bi_sector)
                return ELEVATOR_BACK_MERGE;
        /* bio结束位置等于rq起始位置返回前向合并 */
        else if (blk_rq_pos(rq) - bio_sectors(bio) == bio->bi_sector)
                return ELEVATOR_FRONT_MERGE;
        return ELEVATOR_NO_MERGE;
}
```

执行泄流操作的函数是 `blk_flush_plug_list()` ，代码逻辑如下：

```C++
void blk_flush_plug_list(struct blk_plug *plug, bool from_schedule)
{
        struct request_queue *q;
        unsigned long flags;
        struct request *rq;
        LIST_HEAD(list);
        unsigned int depth;

        BUG_ON(plug->magic != PLUG_MAGIC);

        flush_plug_callbacks(plug, from_schedule);

        if (!list_empty(&plug->mq_list))
                blk_mq_flush_plug_list(plug, from_schedule);

        if (list_empty(&plug->list))
                return;

        list_splice_init(&plug->list, &list);

        /* 1.根据请求所在的q排序，这样可以优先处理同一设备的请求
         * 2.同一设备的请求根据请求的起始位置排序，这样可以优化处理性能
         */
        list_sort(NULL, &list, plug_rq_cmp);

        q = NULL;
        depth = 0;

        /*
         * Save and disable interrupts here, to avoid doing it for every
         * queue lock we have to take.
         */
        local_irq_save(flags);
        while (!list_empty(&list)) {
                rq = list_entry_rq(list.next);
                list_del_init(&rq->queuelist);
                BUG_ON(!rq->q);
                /* 当 rq->q != q 时，说明一个队列的请求都已经取出，排查第一次，
                 * 当q为真时调用q->request_fn处理请求，q->request_fn会调用
                 * 具体的调度算法取出请求进行处理
                 */
                if (rq->q != q) {
                        /*
                         * This drops the queue lock
                         */
                        if (q)
                                queue_unplugged(q, depth, from_schedule);
                        q = rq->q;
                        depth = 0;
                        spin_lock(q->queue_lock);
                }

                /*
                 * Short-circuit if @q is dead
                 */
                if (unlikely(blk_queue_dying(q))) {
                        __blk_end_request_all(rq, -ENODEV);
                        continue;
                }

                /*
                 * rq is already accounted, so use raw insert
                 *
                 * 在这里处理同一个队列的请求时，先将请求插入到调度队列和电梯
                 * 哈希中，同一队列全部插入完成后，在上边会一次性处理
                 */
                if (rq->cmd_flags & (REQ_FLUSH | REQ_FUA))
                        __elv_add_request(q, rq, ELEVATOR_INSERT_FLUSH);
                else
                        __elv_add_request(q, rq, ELEVATOR_INSERT_SORT_MERGE);

                depth++;
        }

        /*
         * This drops the queue lock
         * 处理最后一个设备的请求
         */
        if (q)
                queue_unplugged(q, depth, from_schedule);

        local_irq_restore(flags);
}
```

### 2.3. Elevator 合并

bio 生成新的 request 后，会插入到两个位置。一是电梯调度算法的哈希表中，以 request 的结束位置为哈希值进行哈希，便于查找可以后向合并的请求，取值为 `q->elevator->hash` ；二是具体调度算法的调度队列，用于调度算法进行 I/O 调度，以 deadline 调度算法为例，取值为 `q->elevator->elevator_data->sort_list/fifo_expire` 。调度算法派发请求后，请求会进入 `q` 的派发队列并同时从哈希和调度队列中移除。执行 Elevator 合并的函数为 `elv_merge()` ，主要包含电梯调度算法的后向合并以及具体 I/O 调度算法的前向合并，并且为了提高合并效率，函数在最开始先检查最近一次合并成功的请求能否与 bio 进行合并。代码逻辑如下：

```C++
int elv_merge(struct request_queue *q, struct request **req, struct bio *bio)
{
        struct elevator_queue *e = q->elevator;
        struct request *__rq;
        int ret;

        /*
         * Levels of merges:
         *         nomerges:  No merges at all attempted
         *         noxmerges: Only simple one-hit cache try
         *         merges:           All merge tries attempted
         *
         * 检查队列是否设置禁止合并的标记
         */
        if (blk_queue_nomerges(q))
                return ELEVATOR_NO_MERGE;

        /*
         * First try one-hit cache.
         * 首先检测上次进行了合并的请求能否再次合并
         */
        if (q->last_merge && elv_bio_merge_ok(q->last_merge, bio)) {
                ret = blk_try_merge(q->last_merge, bio);
                if (ret != ELEVATOR_NO_MERGE) {
                        *req = q->last_merge;
                        return ret;
                }
        }

        if (blk_queue_noxmerges(q))
                return ELEVATOR_NO_MERGE;

        /*
         * See if our hash lookup can find a potential backmerge.
         * 在哈希中查找可能后向合并的请求
         */
        __rq = elv_rqhash_find(q, bio->bi_sector);
        if (__rq && elv_bio_merge_ok(__rq, bio)) {
                *req = __rq;
                return ELEVATOR_BACK_MERGE;
        }

        /* 调用具体调度算法的接口判断能否进行前向合并 */
        if (e->uses_mq && e->aux->ops.mq.request_merge)
                return e->aux->ops.mq.request_merge(q, req, bio);
        else if (!e->uses_mq && e->aux->ops.sq.elevator_merge_fn)
                return e->aux->ops.sq.elevator_merge_fn(q, req, bio);

        return ELEVATOR_NO_MERGE;
}
```

### 2.4. 前/后向合并

在结构体 `struct request` 中有两个成员分别是 `struct bio *bio` 和 `struct bio *biotail` ，成员 `bio` 表示还没有完成传输的第一个 bio，成员 `biotail` 指向最后一个需要处理的 bio。在结构体 `struct bio` 中有一个成员 `struct bio *bi_next` ，该成员指向下一个需要处理的 bio，多个需要连续处理的 bio 通过该成员链接。

前向合并指的是将 bio 合并到 req 的前边，故基本逻辑为：

```C++
/* 将bio插入到req前边 */
bio->bi_next = req->bio;
req->bio = bio;
```

后向合并指的是将 bio 合并到 req 的后边，故基本逻辑为：

```C++
/* 将bio插入到req尾部 */
req->biotail->bi_next = bio;
req->biotail = bio;
```
