---
title: RAID5 I/O 处理之条带读代码详解
date: 2024-07-26 15:14:12
tags: raid5
categories: md
description: 详细分析 RAID5 I/O 处理中条带读部分的代码
---

除了对齐读流程中读失败通过条带重试的场景会进入到条带读，当 I/O 覆盖范围超过一个 chunk 时也会进入条带读（如向 chunk 为 4K 的 RAID 下发起始位置为 1K 大小为 4K 的 I/O），接下来我们就这部分逻辑进行分析。

## 1. I/O 加入链表

首先 bio 通过 `add_stripe_bio()` 函数被挂载到条带头指向成员磁盘设备的 `toread` 上，代码如下所示：

```C++
/* 只保留读请求相关处理逻辑 */
static int add_stripe_bio(struct stripe_head *sh, struct bio *bi, int dd_idx, int forwrite)
{
        struct bio **bip;
        struct r5conf *conf = sh->raid_conf;
 
        spin_lock_irq(&sh->stripe_lock);
        /* 
         * 获取bio所在dev的用于保存toread的地址
         * 后续bio插入时会根据其起始位置进行排序
         * 这里使用二级指针便于后续的插入操作
         */
        bip = &sh->dev[dd_idx].toread;
 
        /* 
         * 遍历当前需要读的bio，判断是否存在bio覆盖范围重叠的场景
         * 如果有重叠则跳转到overlap设置标记后返回0退出
         * 需要等待已存在的导致重叠的bio执行完毕后才能再次执行
         */
        while (*bip && (*bip)->bi_sector < bi->bi_sector) {
                if (bio_end_sector(*bip) > bi->bi_sector)
                        goto overlap;
                bip = & (*bip)->bi_next;
        }
        if (*bip && (*bip)->bi_sector < bio_end_sector(bi))
                goto overlap;
        BUG_ON(*bip && bi->bi_next && (*bip) != bi->bi_next);
 
        /* 将bio根据起始位置顺序插入到toread的bio链表中等待处理 */
        if (*bip)
                bi->bi_next = *bip;
        *bip = bi;
        /* 增加bio的bi_phys_segments计数 */
        raid5_inc_bi_active_stripes(bi);
        spin_unlock_irq(&sh->stripe_lock);
 
        return 1;
 
overlap:
        set_bit(R5_Overlap, &sh->dev[dd_idx].flags);
        spin_unlock_irq(&sh->stripe_lock);
        return 0;
}
```

## 2. 条带处理

条带处理的函数入口为 `handle_active_stripes()` ，代码如下所示：

```C++
#define MAX_STRIPE_BATCH 8
static int handle_active_stripes(struct r5conf *conf)
{
        struct stripe_head *batch[MAX_STRIPE_BATCH], *sh;
        int i, batch_size = 0;
 
        while (batch_size < MAX_STRIPE_BATCH &&
                 /* 根据优先级获取一个待处理条带 */
                 (sh = __get_priority_stripe(conf)) != NULL)
                batch[batch_size++] = sh;
 
        if (batch_size == 0)
                return batch_size;
        spin_unlock_irq(&conf->device_lock);
 
        /* 调用handle_stripe函数处理条带 */
        for (i = 0; i < batch_size; i++)
                handle_stripe(batch[i]);
 
        cond_resched();
 
        spin_lock_irq(&conf->device_lock);
        for (i = 0; i < batch_size; i++)
                __release_stripe(conf, batch[i]);
        return batch_size;
}
```

`handle_stripe()` 是条带处理的主要函数。一个条带从开始到结束需要调用几次 `handle_stripe()` 及相关函数。本文讨论如下四种场景：

- 读成功
- I/O 所在磁盘异常
- 读 I/O 报错
- 阵列超冗余

接下来根据不同场景下每轮处理的内容进行代码分析，**贴出的代码只包含当前处理的相关内容**。

### 2.1. 读成功

正常的条带读会经过以下三轮的条带处理，读取成功后将数据返回给调用者。

#### 2.1.1. 下发读请求

函数调用关系入下：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_stripe_fill()
     \_ fetch_block()
 \_ ops_run_io()
```

各函数执行的代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 调用analyse_stripe解析条带状态 */
        analyse_stripe(sh, &s);
 
        /* s.to_read条件为真进入handle_stripe_fill */
        if (s.to_read || s.non_overwrite
                || (conf->level == 6 && s.to_write && s.failed)
                || (s.syncing && (s.uptodate + s.compute < disks))
                || s.replacing
                || s.expanding)
                handle_stripe_fill(sh, &s, disks);
 
        /* 调用ops_run_io检查是否有需要调度的请求 */
        ops_run_io(sh, &s);
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                /* 统计读请求 */
                if (dev->toread)
                        s->to_read++;
                /* 条带/设备状态正常 */
                if (test_bit(In_sync, &rdev->flags))
                        set_bit(R5_Insync, &dev->flags);
        }
        rcu_read_unlock();
}
 
static void handle_stripe_fill(struct stripe_head *sh,
                               struct stripe_head_state *s,
                               int disks)
{
        int i;
 
        /* 当前条带状态没有设置标记，满足条件判断进入if */
        if (!test_bit(STRIPE_COMPUTE_RUN, &sh->state) && !sh->check_state &&
                 !sh->reconstruct_state)
                for (i = disks; i--; )
                        if (fetch_block(sh, s, i, disks))
                                break;
        set_bit(STRIPE_HANDLE, &sh->state);
}
 
static int fetch_block(struct stripe_head *sh, struct stripe_head_state *s,
                       int disk_idx, int disks)
{
        struct r5dev *dev = &sh->dev[disk_idx];
        struct r5dev *fdev[2] = { &sh->dev[s->failed_num[0]],
                                  &sh->dev[s->failed_num[1]] };
 
        /* dev尚未下发IO所以未设置R5_LOCKED和R5_UPTODATE标记 */
        if (!test_bit(R5_LOCKED, &dev->flags) &&
                 !test_bit(R5_UPTODATE, &dev->flags) &&
                 /* dev->toread条件为真，进入最外层if判断 */
                 dev->toread) {
                /* 在analyse_stripe中设置了R5_Insync */
                if (test_bit(R5_Insync, &dev->flags)) {
                        /* 设置R5_LOCKED标记表明对应磁盘正在进行IO处理 */
                        set_bit(R5_LOCKED, &dev->flags);
                        /* 设置R5_Wantread标记表明需要下发读请求 */
                        set_bit(R5_Wantread, &dev->flags);
                        /* 统计在进行IO操作的dev的计数 */
                        s->locked++;
                }
        }
 
        return 0;
}
 
static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        struct r5conf *conf = sh->raid_conf;
        int i, disks = sh->disks;
 
        might_sleep();
 
        for (i = disks; i--; ) {
                bi = &sh->dev[i].req;
                rbi = &sh->dev[i].rreq; /* For writing to replacement */
 
                rcu_read_lock();
                rrdev = rcu_dereference(conf->disks[i].replacement);
                smp_mb(); /* Ensure that if rrdev is NULL, rdev won't be */
                rdev = rcu_dereference(conf->disks[i].rdev);
 
                if (test_and_clear_bit(R5_Wantwrite, &sh->dev[i].flags)) {
                        if (test_and_clear_bit(R5_WantFUA, &sh->dev[i].flags))
                                rw = WRITE_FUA;
                        else
                                rw = WRITE;
                        if (test_bit(R5_Discard, &sh->dev[i].flags))
                                rw |= REQ_DISCARD;
                } else if (test_and_clear_bit(R5_Wantread, &sh->dev[i].flags))
                        /* 设置为请求类型为读 */
                        rw = READ;
                else if (test_and_clear_bit(R5_WantReplace,
                                                &sh->dev[i].flags)) {
                        rw = WRITE;
                        replace_only = 1;
                } else
                        /* 其余跳过 */
                        continue;
 
                if (rdev) {
                        set_bit(STRIPE_IO_STARTED, &sh->state);
 
                        /* 
                         * 设置bio参数
                         * 包括重新设置bio指向的块设备，起始位置，IO完成回调函数
                         */
                        bio_reset(bi);
                        bi->bi_bdev = rdev->bdev;
                        bi->bi_rw = rw;
                        bi->bi_end_io = (rw & WRITE)
                                ? raid5_end_write_request
                                : raid5_end_read_request;
                        bi->bi_private = sh;
 
                        atomic_inc(&sh->count);
                        if (use_new_offset(conf, sh))
                                bi->bi_sector = (sh->sector + rdev->new_data_offset);
                        else
                                bi->bi_sector = (sh->sector + rdev->data_offset);
 
                        if (test_bit(R5_ReadNoMerge, &sh->dev[i].flags))
                                bi->bi_rw |= REQ_FLUSH;
 
                        bi->bi_vcnt = 1;
                        bi->bi_io_vec[0].bv_len = STRIPE_SIZE;
                        bi->bi_io_vec[0].bv_offset = 0;
                        bi->bi_size = STRIPE_SIZE;
                        if (rrdev)
                                set_bit(R5_DOUBLE_LOCKED, &sh->dev[i].flags);
 
                        /* 调用generic_make_request向底层块设备提交请求 */
                        generic_make_request(bi);
                }
        }
}
```

#### 2.1.2. 拷贝数据到 BIO

函数调用关系如下：

```C++
raid5_end_read_request()
 \_ handle_stripe()
     \_ analyse_stripe()
     \_ raid_run_ops()
         \_ ops_run_biofill()
```

各函数执行的代码逻辑如下：

```C++
static void raid5_end_read_request(struct bio * bi, int error)
{
        /* 读成功设置BIO_UPTODATE标记 */
        int uptodate = test_bit(BIO_UPTODATE, &bi->bi_flags);
 
        if (uptodate) {
                /* 标记条带/设备上的数据为最新的 */
                set_bit(R5_UPTODATE, &sh->dev[i].flags);
        }
 
        /* 自减成员磁盘的pending io计数 */
        rdev_dec_pending(rdev, conf->mddev);
        /* 清除R5_LOCKED标记表明磁盘IO动作结束 */
        clear_bit(R5_LOCKED, &sh->dev[i].flags);
 
        /* 将条带推入状态机处理 */
        set_bit(STRIPE_HANDLE, &sh->state);
        release_stripe(sh);
}
 
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);
 
        /* 设置标记将条带中的数据填充到bio中 */
        if (s.to_fill && !test_bit(STRIPE_BIOFILL_RUN, &sh->state)) {
                set_bit(STRIPE_OP_BIOFILL, &s.ops_request);
                set_bit(STRIPE_BIOFILL_RUN, &sh->state);
        }
 
        /* 因前边设置了标记这里进入raid_run_ops函数 */
        if (s.ops_request)
                raid_run_ops(sh, s.ops_request);
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                /* 设置R5_Wantfill标记表明需要将dev->page中的数据填充到bio->page中 */
                if (test_bit(R5_UPTODATE, &dev->flags) && dev->toread &&
                         !test_bit(STRIPE_BIOFILL_RUN, &sh->state))
                        set_bit(R5_Wantfill, &dev->flags);
 
                /* 统计数据与磁盘数据一致的dev */
                if (test_bit(R5_UPTODATE, &dev->flags))
                        s->uptodate++;
 
                /* 统计需要进行填充数据的dev */
                if (test_bit(R5_Wantfill, &dev->flags))
                        s->to_fill++;
        }
        rcu_read_unlock();
}
 
static void raid_run_ops(struct stripe_head *sh, unsigned long ops_request)
{
        /* 在解析条带时设置了该标记，进入if调用ops_run_biofill */
        if (test_bit(STRIPE_OP_BIOFILL, &ops_request)) {
                ops_run_biofill(sh);
                overlap_clear++;
        }
}
 
static void ops_run_biofill(struct stripe_head *sh)
{
        struct dma_async_tx_descriptor *tx = NULL;
        struct async_submit_ctl submit;
        int i;
 
        for (i = sh->disks; i--; ) {
                struct r5dev *dev = &sh->dev[i];
                /* 设置了R5_Wantfill标记的dev需要调度拷贝数据 */
                if (test_bit(R5_Wantfill, &dev->flags)) {
                        struct bio *rbi;
                        spin_lock_irq(&sh->stripe_lock);
                        /* 将已完成读处理的请求从dev的toread移动到read */
                        dev->read = rbi = dev->toread;
                        /* toread置空 */
                        dev->toread = NULL;
                        spin_unlock_irq(&sh->stripe_lock);
                        while (rbi && rbi->bi_sector <
                                dev->sector + STRIPE_SECTORS) {
                                tx = async_copy_data(0, rbi, dev->page,
                                        dev->sector, tx);
                                rbi = r5_next_bio(rbi, dev->sector);
                        }
                }
        }
 
        atomic_inc(&sh->count);
 
        /* 通过异步引擎进行数据拷贝并设置回调函数为ops_complete_biofill */
        init_async_submit(&submit, ASYNC_TX_ACK, tx, ops_complete_biofill, sh, NULL);
        async_trigger_callback(&submit);
}
```

#### 2.1.3. 向上返回

函数调用关系如下：

```C++
ops_complete_biofill()
 \_ return_io()
```

各函数执行的代码逻辑如下：

```C++
static void ops_complete_biofill(void *stripe_head_ref)
{
        struct stripe_head *sh = stripe_head_ref;
        struct bio *return_bi = NULL;
        int i;
 
        /* clear completed biofills */
        for (i = sh->disks; i--; ) {
                struct r5dev *dev = &sh->dev[i];
 
                /* 判断并清除R5_Wantfill标记 */
                if (test_and_clear_bit(R5_Wantfill, &dev->flags)) {
                        struct bio *rbi, *rbi2;
 
                        BUG_ON(!dev->read);
                        /* 将完成拷贝的读请求赋值到rbi，dev的read置空 */
                        rbi = dev->read;
                        dev->read = NULL;
 
                        /* 遍历条带范围内所有的bio */
                        while (rbi && rbi->bi_sector <
                                dev->sector + STRIPE_SECTORS) {
                                /* 通过bi_next获取条带范围内的下一个bio */
                                rbi2 = r5_next_bio(rbi, dev->sector);
                                /* 
                                 * 如果bi_phy_segments为0表明bio处理完毕，此时将bio挂载到
                                 * return_bi上，多个bio通过bi_next链接。
                                 * 假设有2个bio执行完成，执行完这段代码后结果为
                                 * return_bi->bio2->bio1->NULL
                                 */
                                if (!raid5_dec_bi_active_stripes(rbi)) {
                                        rbi->bi_next = return_bi;
                                        return_bi = rbi;
                                }
                                rbi = rbi2;
                        }
                }
        }
        clear_bit(STRIPE_BIOFILL_RUN, &sh->state);
 
        return_io(return_bi);
 
        set_bit(STRIPE_HANDLE, &sh->state);
        /* 将条带推入状态机处理 */
        release_stripe(sh);
}
 
static void return_io(struct bio *return_bi)
{
        struct bio *bi = return_bi;
 
        /* 通过bi_next遍历bio调用bio_endio结束io并通知上层 */
        while (bi) {
                return_bi = bi->bi_next;
                bi->bi_next = NULL;
                bi->bi_size = 0;
                bio_endio(bi, 0);
                bi = return_bi;
        }
}
```

### 2.2. I/O 所在磁盘异常

这里指下发 I/O 之前磁盘异常。当 I/O 所在磁盘异常时无法从该磁盘上直接读取数据，需要通过读取同条带的其他磁盘数据然后经过异或运算还原出当前要读的磁盘的数据。该流程会经过以下四轮的条带处理，读取成功后将数据返回给调用者。

#### 2.2.1. 下发读请求

函数调用关系如下：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_stripe_fill()
     \_ fetch_block()
 \_ ops_run_io()
```

各函数执行的代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带 */
        analyse_stripe(sh, &s);
 
        /* 满足s.to_read条件进入handle_stripe_fill */
        if (s.to_read || s.non_overwrite
                || (conf->level == 6 && s.to_write && s.failed)
                || (s.syncing && (s.uptodate + s.compute < disks))
                || s.replacing
                || s.expanding)
                handle_stripe_fill(sh, &s, disks);
 
        /* 调用ops_run_io检查是否有请求需要下发 */
        ops_run_io(sh, &s);
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                dev = &sh->dev[i];
 
                /* 统计有读请求的dev */
                if (dev->toread)
                        s->to_read++;
 
                /* 磁盘异常rdev设置为NULL */
                if (rdev && test_bit(Faulty, &rdev->flags))
                        rdev = NULL;
                /* 清除条带/设备同步状态标记 */
                clear_bit(R5_Insync, &dev->flags);
 
                if (!test_bit(R5_Insync, &dev->flags)) {
                        /* 记录异常磁盘索引 */
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        /* 统计异常dev */
                        s->failed++;
                }
        }
        rcu_read_unlock();
}
 
static void handle_stripe_fill(struct stripe_head *sh,
                               struct stripe_head_state *s,
                               int disks)
{
        int i;
 
        /* 当前条带状态没有设置标记，满足条件判断进入if */
        if (!test_bit(STRIPE_COMPUTE_RUN, &sh->state) && !sh->check_state &&
                 !sh->reconstruct_state)
                for (i = disks; i--; )
                        if (fetch_block(sh, s, i, disks))
                                break;
        set_bit(STRIPE_HANDLE, &sh->state);
}
 
static int fetch_block(struct stripe_head *sh, struct stripe_head_state *s,
                       int disk_idx, int disks)
{
        /* 对于有读请求但磁盘异常的dev满足该条件进入到if */
        if (!test_bit(R5_LOCKED, &dev->flags) &&
                 !test_bit(R5_UPTODATE, &dev->flags) &&
                 (dev->toread ||
                  (s->failed >= 1 && fdev[0]->toread))) {
                /* 
                 * 此时s.uptodate为0所以只能进入到最后的else if
                 * 由于异常磁盘对应的dev无R5_Insync标记所以异常磁盘对应的dev什么都没做
                 * 其他磁盘设置R5_LOCKED和R5_Wantread标记准备下发读请求
                 */
                if (test_bit(R5_Insync, &dev->flags)) {
                        set_bit(R5_LOCKED, &dev->flags);
                        set_bit(R5_Wantread, &dev->flags);
                        s->locked++;
                        pr_debug("Reading block %d (sync=%d)\n",
                                disk_idx, s->syncing);
                }
        }
 
        return 0;
}
 
static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        struct r5conf *conf = sh->raid_conf;
        int i, disks = sh->disks;
 
        might_sleep();
 
        for (i = disks; i--; ) {
                bi = &sh->dev[i].req;
                rbi = &sh->dev[i].rreq; /* For writing to replacement */
 
                rcu_read_lock();
                rrdev = rcu_dereference(conf->disks[i].replacement);
                smp_mb(); /* Ensure that if rrdev is NULL, rdev won't be */
                rdev = rcu_dereference(conf->disks[i].rdev);
 
                if (test_and_clear_bit(R5_Wantwrite, &sh->dev[i].flags)) {
                        if (test_and_clear_bit(R5_WantFUA, &sh->dev[i].flags))
                                rw = WRITE_FUA;
                        else
                                rw = WRITE;
                        if (test_bit(R5_Discard, &sh->dev[i].flags))
                                rw |= REQ_DISCARD;
                } else if (test_and_clear_bit(R5_Wantread, &sh->dev[i].flags))
                        /* 设置为请求类型为读 */
                        rw = READ;
                else if (test_and_clear_bit(R5_WantReplace,
                                                &sh->dev[i].flags)) {
                        rw = WRITE;
                        replace_only = 1;
                } else
                        /* 其余跳过 */
                        continue;
 
                if (rdev) {
                        set_bit(STRIPE_IO_STARTED, &sh->state);
 
                        /* 
                         * 设置bio参数
                         * 包括重新设置bio指向的块设备，起始位置，IO完成回调函数
                         */
                        bio_reset(bi);
                        bi->bi_bdev = rdev->bdev;
                        bi->bi_rw = rw;
                        bi->bi_end_io = (rw & WRITE)
                                ? raid5_end_write_request
                                : raid5_end_read_request;
                        bi->bi_private = sh;
 
                        atomic_inc(&sh->count);
                        if (use_new_offset(conf, sh))
                                bi->bi_sector = (sh->sector + rdev->new_data_offset);
                        else
                                bi->bi_sector = (sh->sector + rdev->data_offset);
 
                        if (test_bit(R5_ReadNoMerge, &sh->dev[i].flags))
                                bi->bi_rw |= REQ_FLUSH;
 
                        bi->bi_vcnt = 1;
                        bi->bi_io_vec[0].bv_len = STRIPE_SIZE;
                        bi->bi_io_vec[0].bv_offset = 0;
                        bi->bi_size = STRIPE_SIZE;
                        if (rrdev)
                                set_bit(R5_DOUBLE_LOCKED, &sh->dev[i].flags);
 
                        /* 调用generic_make_request向底层块设备提交请求 */
                        generic_make_request(bi);
                }
        }
}
```

#### 2.2.2. 执行异或运算

函数调用关系如下：

```C++
raid5_end_read_request()
 \_ handle_stripe()
     \_ analyse_stripe()
     \_ handle_stripe_fill()
         \_ fetch_block()
     \_ raid_run_ops()
         \_ ops_run_compute5()
```

各函数执行的代码逻辑如下：

```C++
static void raid5_end_read_request(struct bio * bi, int error)
{
        int uptodate = test_bit(BIO_UPTODATE, &bi->bi_flags);
 
        if (uptodate) {
                /* 设置R5_UPTODATE标记给读成功的dev */
                set_bit(R5_UPTODATE, &sh->dev[i].flags);
        }
 
        rdev_dec_pending(rdev, conf->mddev);
        /* 清除已经完成IO的dev的R5_LOCKED标记 */
        clear_bit(R5_LOCKED, &sh->dev[i].flags);
 
        /* 将条带推入状态机处理 */
        set_bit(STRIPE_HANDLE, &sh->state);
        release_stripe(sh);
}
 
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);
 
        /* 满足s.to_read条件进入handle_stripe_fill */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);
 
        /* 在fetch_block中设置了s.ops_request进入raid_run_ops执行raid相关操作 */
        if (s.ops_request)
                raid_run_ops(sh, s.ops_request);
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                /* 统计读成功的dev */
                if (test_bit(R5_UPTODATE, &dev->flags))
                        s->uptodate++;
                /* 统计有读请求的dev */
                if (dev->toread)
                        s->to_read++;
 
                /* 清除条带/设备数据最新的标记 */
                clear_bit(R5_Insync, &dev->flags);
 
                if (!test_bit(R5_Insync, &dev->flags)) {
                        /* 记录异常磁盘索引 */
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        /* 统计异常的磁盘 */
                        s->failed++;
                }
        }
        rcu_read_unlock();
}
 
static void handle_stripe_fill(struct stripe_head *sh,
                               struct stripe_head_state *s,
                               int disks)
{
        int i;
 
        /* 当前条带状态没有设置标记，满足条件判断进入if */
        if (!test_bit(STRIPE_COMPUTE_RUN, &sh->state) && !sh->check_state &&
                 !sh->reconstruct_state)
                for (i = disks; i--; )
                        if (fetch_block(sh, s, i, disks))
                                break;
        set_bit(STRIPE_HANDLE, &sh->state);
}
 
static int fetch_block(struct stripe_head *sh, struct stripe_head_state *s,
                       int disk_idx, int disks)
{
        if (!test_bit(R5_LOCKED, &dev->flags) &&
                 /* 已经完成读请求的dev设置了R5_UPTODATE标记不会进入if */
                 !test_bit(R5_UPTODATE, &dev->flags) &&
                 /* 异常磁盘toread为真进入if */
                 dev->toread) {
                /* 除异常磁盘外其他都读成功，此时s.uptodate满足条件 */
                if ((s->uptodate == disks - 1) &&
                    /* disk_idx为异常磁盘索引时满足条件 */
                    (s->failed && (disk_idx == s->failed_num[0] ||
                     disk_idx == s->failed_num[1]))) {
                        /* 标记条带需要进行计算操作 */
                        set_bit(STRIPE_COMPUTE_RUN, &sh->state);
                        set_bit(STRIPE_OP_COMPUTE_BLK, &s->ops_request);
                        /* 标记需要计算的条带/设备 */
                        set_bit(R5_Wantcompute, &dev->flags);
                        /* 标记需要计算的条带/设备索引 */
                        sh->ops.target = disk_idx;
                        sh->ops.target2 = -1; /* no 2nd target */
                        s->req_compute = 1;
                        /* 增加同步计数 */
                        s->uptodate++;
                        return 1;
                }
        }
 
        return 0;
}
 
static void raid_run_ops(struct stripe_head *sh, unsigned long ops_request)
{
        /* 调用计算函数 */
        if (test_bit(STRIPE_OP_COMPUTE_BLK, &ops_request)) {
                if (level < 6)
                        tx = ops_run_compute5(sh, percpu);
        }
}
 
static struct dma_async_tx_descriptor *
ops_run_compute5(struct stripe_head *sh, struct raid5_percpu *percpu)
{
        /* 设置完成回调函数为Ops_complete_compute */
        init_async_submit(&submit, ASYNC_TX_FENCE|ASYNC_TX_XOR_ZERO_DST, NULL,
                          ops_complete_compute, sh, to_addr_conv(sh, percpu));
        /* 通过异步接口进行异或运算 */
        tx = async_xor(xor_dest, xor_srcs, 0, count, STRIPE_SIZE, &submit);
 
        return tx;
}
```

#### 2.2.3. 拷贝数据到 BIO

```C++
static void ops_complete_compute(void *stripe_head_ref)
{
        /* 调用mark_target_uptodate函数设置dev状态为uptodate */
        mark_target_uptodate(sh, sh->ops.target);
        mark_target_uptodate(sh, sh->ops.target2);
        /* 计算工作完成清除STRIPE_COMPUTE_RUN标记 */
        clear_bit(STRIPE_COMPUTE_RUN, &sh->state);
 
        /* 将条带推入状态机处理 */
        set_bit(STRIPE_HANDLE, &sh->state);
        release_stripe(sh);
}
```

之后进入到 BIOFILL 流程与前一小节的流程一致，这里不再赘述。

#### 2.2.4. 向上返回

该流程与读成功流程一致，这里不再赘述。

### 2.3. 读 I/O 报错

这里所说的读 I/O 报错指的是在下发 I/O 之前 RAID 成员磁盘状态正常，但在执行 I/O 过程中底层报告错误，进而将该成员磁盘标记为故障磁盘。条带的第一轮下发读请求的处理过程与读成功章节相同，这里不再赘述。在 I/O 完成的回调函数 `raid5_end_read_request()` 中处理不同，流程如下。

```C++
static void raid5_end_read_request(struct bio * bi, int error)
{
        int uptodate = test_bit(BIO_UPTODATE, &bi->bi_flags);
 
        if (!uptodate) {
                /* 清除R5_UPTODATE标记表明dev IO异常 */
                clear_bit(R5_UPTODATE, &sh->dev[i].flags);
                /* 阵列第一次出现读错误的情况下设置retry标记进入重试 */
                if (retry)
                        /* 
                         * 如果设置了R5_ReadNoMerge标记表明是正在重试的对齐读，也说明异常的是IO本身
                         * 所以此时设置R5_ReadError标记给dev并清除R5_ReadNoMerge标记
                         */
                        if (test_bit(R5_ReadNoMerge, &sh->dev[i].flags)) {
                                set_bit(R5_ReadError, &sh->dev[i].flags);
                                clear_bit(R5_ReadNoMerge, &sh->dev[i].flags);
                        } else
                                /* 
                                 * 如果未设置R5_ReadNoMerge标记，此次异常可能由于进行了IO合并其他IO异常导致返回错误，
                                 * 所以设置该标记再次下发重试，如果成功则按照“读成功”流程继续执行
                                 * 如果重试失败则执行1817行的内容
                                 */
                                set_bit(R5_ReadNoMerge, &sh->dev[i].flags);
        }
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                /* 未设置R5_UPTODATE标记所以不设置R5_Wantfill填充数据 */
                if (test_bit(R5_UPTODATE, &dev->flags) && dev->toread &&
                        !test_bit(STRIPE_BIOFILL_RUN, &sh->state))
                        set_bit(R5_Wantfill, &dev->flags);
 
                /* 统计读请求 */
                if (dev->toread)
                        s->to_read++;
 
                /* 磁盘未设置为Faulty所以依然设置dev为R5_Insync状态 */
                if (test_bit(In_sync, &rdev->flags))
                        set_bit(R5_Insync, &dev->flags);
 
                /* 因为设置了R5_ReadError故清除dev的R5_Insync标记 */
                if (test_bit(R5_ReadError, &dev->flags))
                        clear_bit(R5_Insync, &dev->flags);
 
                /* 记录异常磁盘索引并统计异常dev */
                if (!test_bit(R5_Insync, &dev->flags)) {
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        s->failed++;
                        if (rdev && !test_bit(Faulty, &rdev->flags))
                                do_recovery = 1;
                }
        }
        rcu_read_unlock();
}
```

后续执行与 I/O 所在磁盘异常 流程相同的 **读其他成员磁盘**、**计算校验**、**向上返回** 流程。

正常来讲，执行到这里上层已经获取到了想要的结果，我们可以结束流程了，但是成员磁盘还设置了 `R5_ReadError` 标记，我们是不是可以尝试进行修复呢？因为现在生产的硬盘都有这样的功能：在对硬盘的读/写过程中，如果发现一个坏扇区，则由内部管理程序自动分配一个备用扇区来替换该扇区。这样一来，少量的坏扇区有可能在使用过程中被自动替换掉了，我们还可以继续使用。基于这个想法，我们可以将计算好的数据再写回成员磁盘，我们不管他是否写到了一个替换过的扇区，只要能再成功读回该数据就可以了。

### 2.4. 阵列超冗余

阵列超冗余时 IO 直接返回错误，处理流程如下所示。

函数调用关系如下：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_failed_stripe()
```

各函数执行的代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带 */
        analyse_stripe(sh, &s);
 
        /* 异常磁盘数超过RAID冗余 */
        if (s.failed > conf->max_degraded) {
                sh->check_state = 0;
                sh->reconstruct_state = 0;
                /* s.to_read为真 */
                if (s.to_read+s.to_write+s.written)
                        /* 调用handle_failed_stripe处理条带中的请求 */
                        handle_failed_stripe(conf, sh, &s, disks, &s.return_bi);
        }
 
        /* s.to_read为真进入到handle_stripe_fill中，但是条带的toread设置为NULL
         * 因此在fetch_block中不会设置R5_Wantread标记
         */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);
 
        /* 无需要调度的请求 */
        ops_run_io(sh, &s);
 
        /* 向上层返回 */
        return_io(s.return_bi);
}
 
static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        for (i = disks; i--; ) {
                /* 统计读请求 */
                if (dev->toread)
                        s->to_read++;
 
                /* 磁盘异常rdev设置为NULL */
                if (rdev && test_bit(Faulty, &rdev->flags))
                        rdev = NULL;
 
                /* 清除dev的R5_Insync标记 */
                clear_bit(R5_Insync, &dev->flags);
 
                if (!test_bit(R5_Insync, &dev->flags)) {
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        /* 统计异常计数 */
                        s->failed++;
                        if (rdev && !test_bit(Faulty, &rdev->flags))
                                do_recovery = 1;
                }
        }
        rcu_read_unlock();
}
 
static void
handle_failed_stripe(struct r5conf *conf, struct stripe_head *sh,
                     struct stripe_head_state *s, int disks,
                     struct bio **return_bi)
{
        /* 遍历所有dev */
        for (i = disks; i--; ) {
 
                spin_lock_irq(&sh->stripe_lock);
                /* 
                 * R5_Wantfill标记表示读成功需要将数据从dev拷贝到bio
                 * R5_Insync表示dev对应磁盘是正常的，R5_ReadError表示读错误
                 * 因此这三个判断的是dev中没有也无法读取正确的数据
                 * 没有正确的数据在超冗余的情况下直接返回所有的读请求
                 */
                if (!test_bit(R5_Wantfill, &sh->dev[i].flags) &&
                        (!test_bit(R5_Insync, &sh->dev[i].flags) ||
                          test_bit(R5_ReadError, &sh->dev[i].flags))) {
                        spin_lock_irq(&sh->stripe_lock);
                        bi = sh->dev[i].toread;
                        /* 将条带的toread指针设置为NULL */
                        sh->dev[i].toread = NULL;
                        spin_unlock_irq(&sh->stripe_lock);
                        if (test_and_clear_bit(R5_Overlap, &sh->dev[i].flags))
                                wake_up(&conf->wait_for_overlap);
                        while (bi && bi->bi_sector <
                                   sh->dev[i].sector + STRIPE_SECTORS) {
                                struct bio *nextbi =
                                        r5_next_bio(bi, sh->dev[i].sector);
                                clear_bit(BIO_UPTODATE, &bi->bi_flags);
                                if (!raid5_dec_bi_active_stripes(bi)) {
                                        bi->bi_next = *return_bi;
                                        /* 将条带中的bio赋值给return_bi */
                                        *return_bi = bi;
                                }
                                bi = nextbi;
                        }
                }
 
                clear_bit(R5_LOCKED, &sh->dev[i].flags);
        }
}
```
