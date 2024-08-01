---
title: RAID5 I/O 处理之写请求代码详解
date: 2024-07-26 15:29:07
tags: raid5
categories: md
description: 详细分析 RAID5 I/O 处理中写请求处理部分的代码
---

我们知道 RAID5 一个条带上的数据是由 N 个数据块和 1 个校验块组成，其校验块由 N 个数据块通过异或运算得出，这样才能在任意一个成员磁盘失效时通过其他 N 个成员磁盘恢复出用户写入的数据。这也就要求 RAID5 条带上的数据是一致的、同步的。

## 1. 写入方式

当新数据写入时就需要重新计算校验值，计算方式由以下两种：

1. 将条带上没有写请求的位置的数据读出，然后使用新数据和旧数据两者重新计算校验
2. 将条带上将要写数据的位置的数据和校验数据读出，然后试用新数据、旧数据和旧校验三者重新计算校验

我们以 5 块盘创建的 RAID5 为例，其每个条带上共有 4 个数据块和 1 个校验块。假设某个条带上的数据块依次为 D<sub>1</sub>、D<sub>2</sub>、D<sub>3</sub>、D<sub>4</sub>，校验块为 P<sub>1</sub>，在条带一致的情况下：P<sub>1</sub> = D<sub>1</sub> ⊕ D<sub>2</sub> ⊕ D<sub>3</sub> ⊕ D<sub>4</sub>。现在新的数据 D<sub>1</sub><sup>'</sup>要覆盖原来的 D<sub>1</sub>，两种写入方式的操作如下：

> 第一种方式：先读出 D<sub>2</sub>、D<sub>3</sub>、D<sub>4</sub>，再加上要写入的 D<sub>1</sub><sup>'</sup> 计算出 P<sub>1</sub><sup>'</sup>，即：P<sub>1</sub><sup>'</sup> = D<sub>2</sub> ⊕ D<sub>3</sub> ⊕ D<sub>4</sub> ⊕ D<sub>1</sub><sup>'</sup>  
> 第二种方式：先读出 D<sub>1</sub>、P<sub>1</sub>，再加上要写入的 D<sub>1</sub><sup>'</sup> 计算出 P<sub>1</sub><sup>'</sup>，即：P<sub>1</sub><sup>'</sup> = D<sub>1</sub> ⊕ P<sub>1</sub> ⊕ D<sub>1</sub><sup>'</sup>  

从具体的例子我们可以看出两种方式的本质区别在于要读出旧数据的个数不同，读的越少性能越好。因此在处理逻辑中会先计算那种方式需要读出的数据更少，然后采用该种写方式。第一种读其他旧数据直接计算新校验的写方式称为 **重构写** ，第二种读旧数据和旧校验的方式称为 **读改写** 。

## 2. 读改写

我们以 5 块盘 chunk 为 4K 的 RAID5 为例。按照前文的分析，如果我们在 RAID5 的起始位置写入大小为 4K 的 IO，此时会采用读改写的方式下发请求。代码分析如下：

### 2.1. 接收请求

函数调用关系：

```C++
raid5_make_request()
 \_ add_stripe_bio()
 \_ raid5_release_stripe()
 \_ md_wakeup_thread()
```

接收到请求后，在 `raid5_make_request()` 按 4K 大小切分 bio，通过 `add_stripe_bio()` 将 bio 挂在到条带上，推入条带状态机进行处理。

### 2.2. 下发读请求

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_stripe_dirtying()
 \_ ops_run_io()
```

代码解析：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* to_write条件成立设置需要读数据的条带/设备 */
        if (s.to_write && !sh->reconstruct_state && !sh->check_state)
                handle_stripe_dirtying(conf, sh, &s, disks);

        /* 下发读请求 */
        ops_run_io(sh, &s);
}

static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        rcu_read_lock();
        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                dev = &sh->dev[i];

                /* 写请求挂到条带设备的towrite上此时为真 */
                if (dev->towrite) {
                        s->to_write++;
                        /*
                         * 条带头处理数据块的大小为4K，与内核page大小一致
                         * 因此如果写的数据不足4K时需要先将原来的4K数据读出来再用新的数据覆盖
                         * R5_OVERWRITE标记在挂在bio时进行判断并设置
                         */
                        if (!test_bit(R5_OVERWRITE, &dev->flags))
                                s->non_overwrite++;
                }

                /* 正常情况下成员磁盘状态正常 */
                clear_bit(R5_Insync, &dev->flags);
                if (test_bit(In_sync, &rdev->flags))
                        set_bit(R5_Insync, &dev->flags);
        }
        rcu_read_unlock();
}

static void handle_stripe_dirtying(struct r5conf *conf,
                                   struct stripe_head *sh,
                                   struct stripe_head_state *s,
                                   int disks)
{
        /*
         * 以下几种情况不能进行读改写只能使用重构写
         * 1.RAID级别为6，因为读改写利用的是异或运算的特性，因此RAID6不适用
         * 2.IO起始范围超过了同步的进度。因为读改写需要用旧的数据和旧的校验，
         *   如果旧数据见不是一致的，那么再进行异或运算数据也是不一致的，
         */
        if (conf->max_degraded == 2 ||
                (recovery_cp < MaxSector && sh->sector >= recovery_cp)) {
                /* Calculate the real rcw later - for now make it
                 * look like rcw is cheaper
                 */
                rcw = 1; rmw = 2;
                pr_debug("force RCW max_degraded=%u, recovery_cp=%llu sh->sector=%llu\n",
                         conf->max_degraded, (unsigned long long)recovery_cp,
                         (unsigned long long)sh->sector);
        } else for (i = disks; i--; ) {
                /* 如果dev有写请求或保存的是校验值那么该dev在读改写逻辑中是需要读的 */
                struct r5dev *dev = &sh->dev[i];
                if ((dev->towrite || i == sh->pd_idx) &&
                         !test_bit(R5_LOCKED, &dev->flags) &&
                         !(test_bit(R5_UPTODATE, &dev->flags) ||
                         test_bit(R5_Wantcompute, &dev->flags))) {
                        rmw++;
                }
                /* 如果dev非满写（为写满4K或无IO）并且不是校验那么该dev在重构写逻辑中是需要读的 */
                if (!test_bit(R5_OVERWRITE, &dev->flags) && i != sh->pd_idx &&
                         !test_bit(R5_LOCKED, &dev->flags) &&
                         !(test_bit(R5_UPTODATE, &dev->flags) ||
                         test_bit(R5_Wantcompute, &dev->flags))) {
                        rcw++;
                }
        }

        /* 进行读改写需要读的次数少 */
        if (rmw < rcw && rmw > 0) {
                /* 遍历所有条带/设备 */
                for (i = disks; i--; ) {
                        struct r5dev *dev = &sh->dev[i];
                        /*
                         * 以下几种情况该dev下发读请求
                         * 1.有写请求或保存的是校验值
                         * 2.并且dev没有下发请求（R5_LOCKED）
                         * 3.并且dev中的数据不是磁盘中实际保存的数据（R5_UPTODATE）
                         *   并且没有在进行计算（R5_Wantcompute）
                         * 4.并且dev对应成员磁盘状态正常（R5_Insync）
                         */
                        if ((dev->towrite || i == sh->pd_idx) &&
                                 !test_bit(R5_LOCKED, &dev->flags) &&
                                 !(test_bit(R5_UPTODATE, &dev->flags) ||
                                  test_bit(R5_Wantcompute, &dev->flags)) &&
                                 test_bit(R5_Insync, &dev->flags)) {
                                /* 给条带/设备上锁表明正在进行IO */
                                set_bit(R5_LOCKED, &dev->flags);
                                /* 表明该条带/设备要调度读请求 */
                                set_bit(R5_Wantread, &dev->flags);
                                /* locked增加计数 */
                                s->locked++;
                        }
                }
        }

        /* locked不等于0本轮不调度schedule_reconstruction */
        if ((s->req_compute || !test_bit(STRIPE_COMPUTE_RUN, &sh->state)) &&
                 (s->locked == 0 && (rcw == 0 || rmw == 0) &&
                  !test_bit(STRIPE_BIT_DELAY, &sh->state)))
                schedule_reconstruction(sh, s, rcw == 0, 0);
}

static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                /* 对设置了读标记的下发读请求 */
                if (test_and_clear_bit(R5_Wantread, &sh->dev[i].flags))
                        rw = READ;
                /* 跳过其他不需要读的设备 */
                else
                        continue;

                if (rdev) {
                        bio_reset(bi);
                        bi->bi_bdev = rdev->bdev;
                        bi->bi_rw = rw;
                        bi->bi_end_io = raid5_end_read_request;
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
                        
                        /* 提交bio */
                        generic_make_request(bi);
                }
        }
}
```

### 2.3. 计算新校验

函数调用关系：

```C++
raid5_end_read_request()
 \_ handle_stripe()
     \_ analyse_stripe()
     \_ handle_stripe_dirtying()
         \_ schedule_reconstruction()
     \_ raid_run_ops()
         \_ ops_run_prexor5()
         \_ ops_run_biodrain()
         \_ ops_run_reconstruct5()
```

由上轮次下发读请求的回调出发本轮次处理。

本轮次中，再次进入 `handle_stripe_dirtying()` 后因为读请求的完成，上轮次中需要读的条带/设备都设置了 **R5_UPTODATE** 标记，所以一方面 **rmw** 变量等于 0，另一方面在不需要下发请求 **s->locked** 等于 0，因此满足条件进入到  `schedule_reconstruction()` 中。

```C++
static void
schedule_reconstruction(struct stripe_head *sh, struct stripe_head_state *s,
                         int rcw, int expand)
{
        if (!rcw) {
                /* RAID6不支持读改写 */
                BUG_ON(level == 6);
                BUG_ON(!(test_bit(R5_UPTODATE, &sh->dev[pd_idx].flags) ||
                        test_bit(R5_Wantcompute, &sh->dev[pd_idx].flags)));

                /* 遍历所有条带/设备 */
                for (i = disks; i--; ) {
                        struct r5dev *dev = &sh->dev[i];
                        /* 跳过校验 */
                        if (i == pd_idx)
                                continue;

                        /* 有写请求的条带/设备设置相关标记 */
                        if (dev->towrite &&
                                 (test_bit(R5_UPTODATE, &dev->flags) ||
                                  test_bit(R5_Wantcompute, &dev->flags))) {
                                /* 将数据从bio中拷贝到dev->page中 */
                                set_bit(R5_Wantdrain, &dev->flags);
                                /* 给条带/设备上锁表明正在进行IO */
                                set_bit(R5_LOCKED, &dev->flags);
                                /* 清除标记表明当前条带/设备的page中的数据不可直接试用 */
                                clear_bit(R5_UPTODATE, &dev->flags);
                                /* locked计数 */
                                s->locked++;
                        }
                }

                /* 设置条带重构状态 */
                sh->reconstruct_state = reconstruct_state_prexor_drain_run;
                /* 设置条带需要进行异或运算 */
                set_bit(STRIPE_OP_PREXOR, &s->ops_request);
                /* 设置条带需要“抽干”数据 */
                set_bit(STRIPE_OP_BIODRAIN, &s->ops_request);
                /* 设置条带需要计算校验 */
                set_bit(STRIPE_OP_RECONSTRUCT, &s->ops_request);
        }

        /* 给校验值所在条带/设备上锁表明正在进行IO */
        set_bit(R5_LOCKED, &sh->dev[pd_idx].flags);
        /* 清除标记表明当前条带/设备的page中的数据不可直接试用 */
        clear_bit(R5_UPTODATE, &sh->dev[pd_idx].flags);
        /* locked计数 */
        s->locked++;
}

static void raid_run_ops(struct stripe_head *sh, unsigned long ops_request)
{
        /* 先使用旧数据和旧校验进行异或运算获得中间状态的校验 */
        if (test_bit(STRIPE_OP_PREXOR, &ops_request))
                tx = ops_run_prexor(sh, percpu, tx);

        /* 将新数据拷贝从bio中拷贝到dev中 */
        if (test_bit(STRIPE_OP_BIODRAIN, &ops_request))
                tx = ops_run_biodrain(sh, tx);

        /* 计算最终的校验值 */
        if (test_bit(STRIPE_OP_RECONSTRUCT, &ops_request)) {
                if (level < 6)
                        ops_run_reconstruct5(sh, percpu, tx);
                else
                        ops_run_reconstruct6(sh, percpu, tx);
        }
}

static struct dma_async_tx_descriptor *
ops_run_prexor(struct stripe_head *sh, struct raid5_percpu *percpu,
                   struct dma_async_tx_descriptor *tx)
{
        /* 将校验值的page设置为第一个源数据和目标数据 */
        struct page *xor_dest = xor_srcs[count++] = sh->dev[pd_idx].page;

        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                struct r5dev *dev = &sh->dev[i];
                /* 需要“抽干”数据的dev即包含新数据的dev，将其page依次设置为源数据 */
                if (test_bit(R5_Wantdrain, &dev->flags))
                        xor_srcs[count++] = dev->page;
        }

        /* 进行异或运算 */
        init_async_submit(&submit, ASYNC_TX_FENCE|ASYNC_TX_XOR_DROP_DST, tx,
                          ops_complete_prexor, sh, to_addr_conv(sh, percpu));
        tx = async_xor(xor_dest, xor_srcs, 0, count, STRIPE_SIZE, &submit);

        return tx;
}

static struct dma_async_tx_descriptor *
ops_run_biodrain(struct stripe_head *sh, struct dma_async_tx_descriptor *tx)
{
        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                /* 处理所有需要“抽干”数据的dev */
                if (test_and_clear_bit(R5_Wantdrain, &dev->flags)) {
                        struct bio *wbi;

                        spin_lock_irq(&sh->stripe_lock);
                        /* 将bio从towrite转移到written表明开始调度 */
                        chosen = dev->towrite;
                        dev->towrite = NULL;
                        BUG_ON(dev->written);
                        wbi = dev->written = chosen;
                        spin_unlock_irq(&sh->stripe_lock);

                        /* 将bio中本条带范围内的所有数据拷贝到dev的page中 */
                        while (wbi && wbi->bi_sector < dev->sector + STRIPE_SECTORS) {
                                tx = async_copy_data(1, wbi, dev->page, dev->sector, tx);
                                wbi = r5_next_bio(wbi, dev->sector);
                        }
                }
        }

        return tx;
}

static void
ops_run_reconstruct5(struct stripe_head *sh, struct raid5_percpu *percpu,
                         struct dma_async_tx_descriptor *tx)
{
        /* check if prexor is active which means only process blocks
         * that are part of a read-modify-write (written)
         */
        if (sh->reconstruct_state == reconstruct_state_prexor_drain_run) {
                prexor = 1;
                /* 将校验值的page设置为第一个源数据和目标数据 */
                xor_dest = xor_srcs[count++] = sh->dev[pd_idx].page;
                /* 所有包含需要写请求的条带/设备依次设置为源数据 */
                for (i = disks; i--; ) {
                        struct r5dev *dev = &sh->dev[i];
                        if (dev->written)
                                xor_srcs[count++] = dev->page;
                }
        }

        /* 1/ if we prexor'd then the dest is reused as a source
         * 2/ if we did not prexor then we are redoing the parity
         * set ASYNC_TX_XOR_DROP_DST and ASYNC_TX_XOR_ZERO_DST
         * for the synchronous xor case
         */
        flags = ASYNC_TX_ACK |
                (prexor ? ASYNC_TX_XOR_DROP_DST : ASYNC_TX_XOR_ZERO_DST);
        atomic_inc(&sh->count);

        /* 进行异步异或运算，完成后进入回调函数ops_complete_reconstruct */
        init_async_submit(&submit, flags, tx, ops_complete_reconstruct, sh, to_addr_conv(sh, percpu));
        tx = async_xor(xor_dest, xor_srcs, 0, count, STRIPE_SIZE, &submit);
}
```

### 2.4. 下发写请求

函数调用关系：

```C++
ops_complete_reconstruct()
 \_ handle_stripe()
     \_ analyse_stripe()
     \_ ops_run_io()
```

代码逻辑如下：

```C++
static void ops_complete_reconstruct(void *stripe_head_ref)
{
        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                /* 普通写请求没有如下标记 */
                fua |= test_bit(R5_WantFUA, &sh->dev[i].flags);
                sync |= test_bit(R5_SyncIO, &sh->dev[i].flags);
                discard |= test_bit(R5_Discard, &sh->dev[i].flags);
        }

        for (i = disks; i--; ) {
                struct r5dev *dev = &sh->dev[i];

                if (dev->written || i == pd_idx || i == qd_idx) {
                        if (!discard)
                                /* 设置R5_UPTODATE标记表明条带/设备中的数据为最新可用的 */
                                set_bit(R5_UPTODATE, &dev->flags);
                }
        }

        /* 设置条带重构状态 */
        if (sh->reconstruct_state == reconstruct_state_prexor_drain_run)
                sh->reconstruct_state = reconstruct_state_prexor_drain_result;

        /* 将条带推入状态机 */
        set_bit(STRIPE_HANDLE, &sh->state);
        release_stripe(sh);
}

static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        prexor = 0;
        /* 条件成立 */
        if (sh->reconstruct_state == reconstruct_state_prexor_drain_result)
                prexor = 1;
        if (sh->reconstruct_state == reconstruct_state_drain_result ||
                 sh->reconstruct_state == reconstruct_state_prexor_drain_result) {
                sh->reconstruct_state = reconstruct_state_idle;

                for (i = disks; i--; ) {
                        struct r5dev *dev = &sh->dev[i];
                        /* 所有设置R5_LOCKED的条带/设备如果其是校验或有写请求 */
                        if (test_bit(R5_LOCKED, &dev->flags) &&
                                 (i == sh->pd_idx || i == sh->qd_idx ||
                                  dev->written)) {
                                /* 为条带/设备设置R5_Wantwrite标记表明需要下发写请求 */
                                set_bit(R5_Wantwrite, &dev->flags);
                                /* 读改写跳过下面重构写的相关逻辑判断 */
                                if (prexor)
                                        continue;
                                /* 如果是重构写此时条带处于一致状态设置相关标记 */
                                if (!test_bit(R5_Insync, &dev->flags) ||
                                         ((i == sh->pd_idx || i == sh->qd_idx)  &&
                                          s.failed == 0))
                                        set_bit(STRIPE_INSYNC, &sh->state);
                        }
                }
        }

        /* 下发写请求 */
        ops_run_io(sh, &s);
}

static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                /* 所有设置了R5_Wantwrite标记的条带/设备下发写请求 */
                if (test_and_clear_bit(R5_Wantwrite, &sh->dev[i].flags)) {
                        rw = WRITE;
                } else
                        continue;


                rcu_read_lock();
                rdev = rcu_dereference(conf->disks[i].rdev);
                rcu_read_unlock();

                if (rdev) {
                        bio_reset(bi);
                        bi->bi_bdev = rdev->bdev;
                        bi->bi_rw = rw;
                        bi->bi_end_io = raid5_end_write_request;
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

                        /* 提交写请求 */
                        generic_make_request(bi);
                }
        }
}
```

### 2.5. 向上返回

函数调用关系：

```C++
raid5_end_write_request()
 \_ handle_stripe()
     \_ analyse_stripe()
     \_ handle_stripe_clean_event()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态，本轮次最主要设置s->written计数 */
        analyse_stripe(sh, &s);

        /* 有已经完成调度的写且校验值已经更新完成说明IO整体完成 */
        pdev = &sh->dev[sh->pd_idx];
        if (s.written &&
                (s.p_failed || ((test_bit(R5_Insync, &pdev->flags)
                                 && !test_bit(R5_LOCKED, &pdev->flags)
                                 && (test_bit(R5_UPTODATE, &pdev->flags) ||
                                 test_bit(R5_Discard, &pdev->flags))))))
                /* 将写完成的IO保存到return_bi中 */
                handle_stripe_clean_event(conf, sh, disks, &s.return_bi);

        /* 向上返回 */
        return_io(s.return_bi);
}

static void handle_stripe_clean_event(struct r5conf *conf,
        struct stripe_head *sh, int disks, struct bio **return_bi)
{
        /* 遍历所有条带/设备 */
        for (i = disks; i--; )
                /* 处理所有已调度写请求的条带/设备 */
                if (sh->dev[i].written) {
                        dev = &sh->dev[i];
                        /*
                         * 无R5_LOCKED标记表明IO处理结束
                         * 有R5_UPTODATE标记表明IO处理成功
                         */
                        if (!test_bit(R5_LOCKED, &dev->flags) &&
                                (test_bit(R5_UPTODATE, &dev->flags) ||
                                 test_bit(R5_Discard, &dev->flags))) {
                                /* We can return any write requests */
                                struct bio *wbi, *wbi2;

                                /* 将所有处理完成的bio挂载到return_bi中 */
                                wbi = dev->written;
                                dev->written = NULL;
                                while (wbi && wbi->bi_sector <
                                        dev->sector + STRIPE_SECTORS) {
                                        wbi2 = r5_next_bio(wbi, dev->sector);
                                        if (!raid5_dec_bi_active_stripes(wbi)) {
                                                md_write_end(conf->mddev);
                                                wbi->bi_next = *return_bi;
                                                *return_bi = wbi;
                                        }
                                        wbi = wbi2;
                                }
                        }
                }
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

以上，读改写处理结束。

## 3. 重构写

重构写整体逻辑与读改写相差不大，我们只将不相同的部分说明如下，整体逻辑大家可自行串联。

第一轮读数据：

```C++
static void handle_stripe_dirtying(struct r5conf *conf,
                                   struct stripe_head *sh,
                                   struct stripe_head_state *s,
                                   int disks)
{
        int rmw = 0, rcw = 0, i;
        sector_t recovery_cp = conf->mddev->recovery_cp;

        /*
         * 以下几种情况不能进行读改写只能使用重构写
         * 1.RAID级别为6，因为读改写利用的是异或运算的特性，因此RAID6不适用
         * 2.IO起始范围超过了同步的进度。因为读改写需要用旧的数据和旧的校验，
         *   如果旧数据见不是一致的，那么再进行异或运算数据也是不一致的，
         */
        if (conf->max_degraded == 2 ||
            (recovery_cp < MaxSector && sh->sector >= recovery_cp)) {
                /* Calculate the real rcw later - for now make it
                 * look like rcw is cheaper
                 */
                rcw = 1; rmw = 2;
                pr_debug("force RCW max_degraded=%u, recovery_cp=%llu sh->sector=%llu\n",
                         conf->max_degraded, (unsigned long long)recovery_cp,
                         (unsigned long long)sh->sector);
        } else for (i = disks; i--; ) {
                /* 如果dev有写请求或保存的是校验值那么该dev在读改写逻辑中是需要读的 */
                struct r5dev *dev = &sh->dev[i];
                if ((dev->towrite || i == sh->pd_idx) &&
                    !test_bit(R5_LOCKED, &dev->flags) &&
                    !(test_bit(R5_UPTODATE, &dev->flags) ||
                      test_bit(R5_Wantcompute, &dev->flags))) {
                        if (test_bit(R5_Insync, &dev->flags))
                                rmw++;
                        else
                                rmw += 2*disks;  /* cannot read it */
                }
                /* 如果dev非满写（为写满4K或无IO）并且不是校验那么该dev在重构写逻辑中是需要读的 */
                if (!test_bit(R5_OVERWRITE, &dev->flags) && i != sh->pd_idx &&
                    !test_bit(R5_LOCKED, &dev->flags) &&
                    !(test_bit(R5_UPTODATE, &dev->flags) ||
                    test_bit(R5_Wantcompute, &dev->flags))) {
                        if (test_bit(R5_Insync, &dev->flags)) rcw++;
                        else
                                rcw += 2*disks;
                }
        }

        /* 进行重构写需要读的次数少 */
        if (rcw <= rmw && rcw > 0) {
                /* want reconstruct write, but need to get some data */
                int qread =0;
                rcw = 0;
                for (i = disks; i--; ) {
                        struct r5dev *dev = &sh->dev[i];
                        /*
                         * 满足以下条件下发读请求
                         *   1. 非满写
                         *   2. 不是校验
                         *   3. 尚未下发请求
                         *   4. 当前page中不是最新数据且不需要计算
                         */
                        if (!test_bit(R5_OVERWRITE, &dev->flags) &&
                            i != sh->pd_idx && i != sh->qd_idx &&
                            !test_bit(R5_LOCKED, &dev->flags) &&
                            !(test_bit(R5_UPTODATE, &dev->flags) ||
                              test_bit(R5_Wantcompute, &dev->flags))) {
                                rcw++;
                                if (!test_bit(R5_Insync, &dev->flags))
                                        continue; /* it's a failed drive */

                                /* 给条带/设备上锁表明正在进行IO */
                                set_bit(R5_LOCKED, &dev->flags);
                                /* 表明该条带/设备要调度读请求 */
                                set_bit(R5_Wantread, &dev->flags);
                                s->locked++;
                                /* locked增加计数 */
                                qread++;
                        }
                }
        }
        /* locked不等于0本轮不调度schedule_reconstruction */
        if ((s->req_compute || !test_bit(STRIPE_COMPUTE_RUN, &sh->state)) &&
            (s->locked == 0 && (rcw == 0 || rmw == 0) &&
            !test_bit(STRIPE_BIT_DELAY, &sh->state)))
                schedule_reconstruction(sh, s, rcw == 0, 0);
}
```

在第二轮计算校验的条带处理中，因为直接使用数据段计算校验段，因此不需要做 `ops_run_prexor` ，后续下发写请求和向上返回与读改写相同。

满写是重构写的一种特殊情况，即所有数据段都是满写，此时不需要读旧数据，直接根据新数据计算校验然后一起写入，此时性能最好，我们在实际使用中，可以根据业务写 IO 的 buffer 大小调整数据盘个数和 chunk 大小，以达到满写的状态。
