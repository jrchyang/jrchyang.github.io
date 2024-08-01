---
title: RAID5 I/O 处理之重构代码详解
date: 2024-07-26 15:38:43
tags: raid5
categories: md
description: 详细分析 RAID5 I/O 处理中磁盘重构处理部分的代码
---

## 1. 作用

当阵列降级时，可以添加一块新盘进行重构，以恢复阵列的冗余。

## 2. 发起重构

可以通过以下命令 md 并发起重构：

```sh
mdadm -C /dev/md0 --force --run -l 5 -n 3 -c 128K /dev/sd[b-d] --assume-clean
mdadm --manage -f /dev/md0 /dev/sdb
mdadm --manage -a /dev/md0 /dev/sde
```

相关代码逻辑如下：

### 2.1. 设置磁盘异常

函数调用关系：

```C++
md_ioctl() /* SET_DISK_FAULTY */
 \_ set_disk_faulty()
     \_ md_error()
         \_ error() /* raid5.c */
         \_ md_wakeup_thread() /* raid5d */

raid5d()
 \_ md_check_recovery()
     \_ remove_and_add_spares()
```

这里主要是设置成员磁盘异常的逻辑，代码逻辑如下：

```C++
static void error(struct mddev *mddev, struct md_rdev *rdev)
{
        spin_lock_irqsave(&conf->device_lock, flags);
        /* 清除成员磁盘“同步”状态标记 */
        clear_bit(In_sync, &rdev->flags);
        /* 重新计算md降级状态 */
        mddev->degraded = calc_degraded(conf);
        spin_unlock_irqrestore(&conf->device_lock, flags);
        /* 打断同步 */
        set_bit(MD_RECOVERY_INTR, &mddev->recovery);

        /* 设置成员磁盘为异常状态 */
        set_bit(Blocked, &rdev->flags);
        set_bit(Faulty, &rdev->flags);
        /* 设置md发生磁盘状态改变 */
        set_bit(MD_CHANGE_DEVS, &mddev->flags);
}
```

### 2.2. 添加新盘

函数调用关系：

```C++
md_ioctl() /* ADD_NEW_DISK */
 \_ add_new_disk()
     \_ md_wakeup_thread() /* raid5d */

raid5d()
 \_ md_check_recovery()
     \_ remove_and_add_spares()
     \_ md_register_thread() /* md_do_sync */

md_do_sync()
 \_ sync_request() /* raid5d */
```

这里需要注意，在加盘时没有像前文 replacement 中描述的那样设置磁盘为 WantReplacement 状态，所以不会将新的磁盘赋值给旧盘的 replacement 指针。主要为设置重构相关标记，逻辑如下：

```C++
static int remove_and_add_spares(struct mddev *mddev,
                                 struct md_rdev *this)
{
        int spares = 0;

        rdev_for_each(rdev, mddev) {
                /* 新添加的磁盘未设置相关标记自增spares */
                if (rdev->raid_disk >= 0 &&
                    !test_bit(In_sync, &rdev->flags) &&
                    !test_bit(Journal, &rdev->flags) &&
                    !test_bit(Faulty, &rdev->flags))
                        spares++;
        }

        return spares;
}

void md_check_recovery(struct mddev *mddev)
{
        if (mddev_trylock(mddev)) {
                int spares = 0;

                /* 如上描述，在remove_and_add_spares返回spares为1 */
                if ((spares = remove_and_add_spares(mddev, NULL))) {
                        /* 设置md为重构状态 */
                        clear_bit(MD_RECOVERY_SYNC, &mddev->recovery);
                        clear_bit(MD_RECOVERY_CHECK, &mddev->recovery);
                        clear_bit(MD_RECOVERY_REQUESTED, &mddev->recovery);
                        set_bit(MD_RECOVERY_RECOVER, &mddev->recovery);
                }

                if (mddev->pers->sync_request) {
                        if (spares) {
                                /* We are adding a device or devices to an array
                                 * which has the bitmap stored on all devices.
                                 * So make sure all bitmap pages get written
                                 */
                                bitmap_write_all(mddev->bitmap);
                        }
                        /* 创建重构线程 */
                        mddev->sync_thread = md_register_thread(md_do_sync,
                                                                mddev,
                                                                "resync");
                        /* 创建失败则清除相关标记 */
                        if (!mddev->sync_thread) {
                                printk(KERN_ERR "%s: could not start resync"
                                        " thread...\n", 
                                        mdname(mddev));
                                /* leave the spares where they are, it shouldn't hurt */
                                clear_bit(MD_RECOVERY_RUNNING, &mddev->recovery);
                                clear_bit(MD_RECOVERY_SYNC, &mddev->recovery);
                                clear_bit(MD_RECOVERY_RESHAPE, &mddev->recovery);
                                clear_bit(MD_RECOVERY_REQUESTED, &mddev->recovery);
                                clear_bit(MD_RECOVERY_CHECK, &mddev->recovery);
                        } else {
                                /* 成功则唤醒线程开始执行 */
                                md_wakeup_thread(mddev->sync_thread);
                        }
                }
                mddev_unlock(mddev);
        }
}

static inline sector_t sync_request(struct mddev *mddev, sector_t sector_nr, int *skipped, int go_faster)
{
        /* 获取一个空闲条带 */
        sh = get_active_stripe(conf, sector_nr, 0, 1, 0);
        if (sh == NULL) {
                sh = get_active_stripe(conf, sector_nr, 0, 0, 0);
                schedule_timeout_uninterruptible(1);
        }

        /* 设置同步标记 */
        set_bit(STRIPE_SYNC_REQUESTED, &sh->state);

        /* 将条带推入条带状态机处理 */
        handle_stripe(sh);
        release_stripe(sh);

        return STRIPE_SECTORS;
}
```

## 3. 条带处理

我们依旧通过分析条带各轮次的处理来解析重构过程中代码执行流程及 IO 发生的情况。

### 3.1. 下发读请求

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_stripe_fill()
     \_ fetch_block()
 \_ ops_run_io()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 在sync_request中设置了该标记 */
        if (test_bit(STRIPE_SYNC_REQUESTED, &sh->state)) {
                spin_lock(&sh->stripe_lock);
                /* 此时条带不是处理DISCARD请求 */
                if (!test_bit(STRIPE_DISCARD, &sh->state)
                    /* 清掉STRIPE_SYNC_REQUESTED标记 */
                    && test_and_clear_bit(STRIPE_SYNC_REQUESTED, &sh->state)) {
                        /* 设置条带同步中标记 */
                        set_bit(STRIPE_SYNCING, &sh->state);
                        /* 清除条带一致状态的标记 */
                        clear_bit(STRIPE_INSYNC, &sh->state);
                }
                spin_unlock(&sh->stripe_lock);
        }
        clear_bit(STRIPE_DELAYED, &sh->state);

        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* s.syncing为真且第一轮条带处理时s.uptodate + s.compute等于0条件满足进入handle_stripe_fill */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /* 此时 s.locked == 0 条件不成立不会进入该if分支 */
        if ((s.syncing || s.replacing) && s.locked == 0
            && test_bit(STRIPE_INSYNC, &sh->state)) {
                md_done_sync(conf->mddev, STRIPE_SECTORS, 1);
                clear_bit(STRIPE_SYNCING, &sh->state);
                if (test_and_clear_bit(R5_Overlap, &sh->dev[sh->pd_idx].flags))
                        wake_up(&conf->wait_for_overlap);
        }

        /* 下发读请求 */
        ops_run_io(sh, &s);
}

static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        int do_recovery = 0;

        /* 遍历所有条带/设备 */
        rcu_read_lock();
        for (i=disks; i--; ) {
                /* 新加入的成员磁盘重构完成之前不处于同步状态，满足if条件 */
                if (!test_bit(R5_Insync, &dev->flags)) {
                        /* 加上raid6在内最大支持坏2块磁盘 */
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        /* 自增failed */
                        s->failed++;
                        /* rdev指向新盘且新盘不是Faulty状态（旧盘是），满足if条件设置do_recovery */
                        if (rdev && !test_bit(Faulty, &rdev->flags))
                                do_recovery = 1;
                }
        }

        /* 在handle_stripe中设置了该标记 */
        if (test_bit(STRIPE_SYNCING, &sh->state)) {
                /* do_recovery条件满足，设置 s->syncing = 1 表明条带在做重构 */
                if (do_recovery
                    || sh->sector >= conf->mddev->recovery_cp
                    || test_bit(MD_RECOVERY_REQUESTED, &(conf->mddev->recovery)))
                        s->syncing = 1;
                else
                        s->replacing = 1;
        }
        rcu_read_unlock();
}


static void handle_stripe_fill(struct stripe_head *sh,
                               struct stripe_head_state *s,
                               int disks)
{
        int i;

        /* 未设置条带状态进入fetch_block */
        if (!test_bit(STRIPE_COMPUTE_RUN, &sh->state) && !sh->check_state
            && !sh->reconstruct_state)
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

        /* 此时所有条带/设备都未发起请求且未包含最新数据 */
        /* 满足s->syncing条件进入第一层if */
        if (!test_bit(R5_LOCKED, &dev->flags)
            && !test_bit(R5_UPTODATE, &dev->flags)
            && (dev->toread
                || (dev->towrite && !test_bit(R5_OVERWRITE, &dev->flags))
                || s->syncing || s->expanding
                || (s->replacing && want_replace(sh, disk_idx))
                || (s->failed >= 1 && fdev[0]->toread)
                || (s->failed >= 2 && fdev[1]->toread)
                || (sh->raid_conf->level <= 5 && s->failed && fdev[0]->towrite
                    && !test_bit(R5_OVERWRITE, &fdev[0]->flags))
                || (sh->raid_conf->level == 6 && s->failed && s->to_write))) {
                /* we would like to get this block, possibly by computing it,
                 * otherwise read it if the backing disk is insync
                 */
                BUG_ON(test_bit(R5_Wantcompute, &dev->flags));
                BUG_ON(test_bit(R5_Wantread, &dev->flags));
                /*
                 * 对所有正常可读的成员磁盘下发读请求
                 * 需要注意的是，如果是raid5，因为只有一个冗余，因此重构是需要向所有其他磁盘下发读的
                 * 但是如果是raid6，因为有两个冗余，在只有一个成员磁盘异常的情况下
                 * 可以少读一块盘，但是实际没有这么做还是都读了，在后续处理中会用
                 * 计算出来的值和读出来的值进行比较如果不相等则重新写一次进行修复
                 */
                if (test_bit(R5_Insync, &dev->flags)) {
                        set_bit(R5_LOCKED, &dev->flags);
                        set_bit(R5_Wantread, &dev->flags);
                        /* 自增locked计数 */
                        s->locked++;
                }
        }

        return 0;
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

### 3.2. 计算校验

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_stripe_fill()
     \_ fetch_block()
 \_ raid_run_ops()
     \_ ops_run_compute5()
         \_ ops_complete_compute()
             \_ mark_target_uptodate()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* s.syncing为真，此时s.compute为0，s.uptodate == disks-1，满足if条件 */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /* 在fetch_block中设置了STRIPE_COMPUTE_RUN不进入if */
        if (sh->check_state ||
            (s.syncing && s.locked == 0 &&
             !test_bit(STRIPE_COMPUTE_RUN, &sh->state) &&
             !test_bit(STRIPE_INSYNC, &sh->state))) {
                if (conf->level == 6)
                        handle_parity_checks6(conf, sh, &s, disks);
                else
                        handle_parity_checks5(conf, sh, &s, disks);
        }

        /* 计算校验值 */
        if (s.ops_request)
                raid_run_ops(sh, s.ops_request);
}

static void analyse_stripe(struct stripe_head *sh, struct stripe_head_state *s)
{
        /* 遍历成员磁盘 */
        rcu_read_lock();
        for (i=disks; i--; ) {
                dev = &sh->dev[i];

                /* 在raid5_end_read_request中清除了R5_LOCKED标记，所以此时不自增locked */
                if (test_bit(R5_LOCKED, &dev->flags))
                        s->locked++;
                /* 读成功自增uptodate */
                if (test_bit(R5_UPTODATE, &dev->flags))
                        s->uptodate++;
                if (!test_bit(R5_Insync, &dev->flags)) {
                        /* 保存异常磁盘索引 */
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        s->failed++;
                        if (rdev && !test_bit(Faulty, &rdev->flags))
                                do_recovery = 1;
                }
        }
        rcu_read_unlock();
}

static int fetch_block(struct stripe_head *sh, struct stripe_head_state *s,
                       int disk_idx, int disks)
{
        /* 此时只有异常磁盘没有下发读所以未设置R5_UPTODATE标记，只有遍历到该成员磁盘时进入if */
        if (!test_bit(R5_LOCKED, &dev->flags) &&
            !test_bit(R5_UPTODATE, &dev->flags) &&
            (dev->toread ||
             (dev->towrite && !test_bit(R5_OVERWRITE, &dev->flags)) ||
             s->syncing || s->expanding ||
             (s->replacing && want_replace(sh, disk_idx)) ||
             (s->failed >= 1 && fdev[0]->toread) ||
             (s->failed >= 2 && fdev[1]->toread) ||
             (sh->raid_conf->level <= 5 && s->failed && fdev[0]->towrite &&
              !test_bit(R5_OVERWRITE, &fdev[0]->flags)) ||
             (sh->raid_conf->level == 6 && s->failed && s->to_write))) {
                /* we would like to get this block, possibly by computing it,
                 * otherwise read it if the backing disk is insync
                 */
                BUG_ON(test_bit(R5_Wantcompute, &dev->flags));
                BUG_ON(test_bit(R5_Wantread, &dev->flags));
                /* raid5重构时除热备盘外其他都读成功时满足第一个条件 */
                /* 该成员磁盘是统计的异常（未同步）磁盘时满足第二个条件 */
                if ((s->uptodate == disks - 1) &&
                    (s->failed && (disk_idx == s->failed_num[0] ||
                                   disk_idx == s->failed_num[1]))) {
                        /* 设置条带状态为开始计算流程后续不会再进入fetch_block */
                        set_bit(STRIPE_COMPUTE_RUN, &sh->state);
                        /* 设置需要进行计算操作 */
                        set_bit(STRIPE_OP_COMPUTE_BLK, &s->ops_request);
                        /* 设置该条带/设备为需要计算 */
                        set_bit(R5_Wantcompute, &dev->flags);
                        /* 保存需要计算的条带/设备索引 */
                        sh->ops.target = disk_idx;
                        sh->ops.target2 = -1; /* no 2nd target */
                        s->req_compute = 1;
                        /* 自增uptodate，自增之后不会再进入handle_stripe_fill函数 */
                        s->uptodate++;
                        return 1;
                }
        }

        return 0;
}

static void raid_run_ops(struct stripe_head *sh, unsigned long ops_request)
{
        cpu = get_cpu();
        percpu = per_cpu_ptr(conf->percpu, cpu);

        if (test_bit(STRIPE_OP_COMPUTE_BLK, &ops_request)) {
                /* 进行重构计算 */
                tx = ops_run_compute5(sh, percpu);
                /* terminate the chain if reconstruct is not set to be run */
                if (tx && !test_bit(STRIPE_OP_RECONSTRUCT, &ops_request))
                        async_tx_ack(tx);
        }
        put_cpu();
}

static struct dma_async_tx_descriptor *
ops_run_compute5(struct stripe_head *sh, struct raid5_percpu *percpu)
{
        int disks = sh->disks;
        struct page **xor_srcs = percpu->scribble;
        int target = sh->ops.target;
        struct r5dev *tgt = &sh->dev[target];
        struct page *xor_dest = tgt->page;
        int count = 0;
        struct dma_async_tx_descriptor *tx;
        struct async_submit_ctl submit;
        int i;

        pr_debug("%s: stripe %llu block: %d\n",
                __func__, (unsigned long long)sh->sector, target);
        BUG_ON(!test_bit(R5_Wantcompute, &tgt->flags));

        /* 统计源数据个数，重构流程中只设置了热备盘为target其他均为源，所以count大于1 */
        for (i = disks; i--; )
                if (i != target)
                        xor_srcs[count++] = sh->dev[i].page;

        atomic_inc(&sh->count);

        /* 设置回调函数 */
        init_async_submit(&submit, ASYNC_TX_FENCE|ASYNC_TX_XOR_ZERO_DST, NULL,
                          ops_complete_compute, sh, to_addr_conv(sh, percpu));
        
        /* count大于1执行async_xor异步进行异或运算 */
        if (unlikely(count == 1))
                tx = async_memcpy(xor_dest, xor_srcs[0], 0, 0, STRIPE_SIZE, &submit);
        else
                tx = async_xor(xor_dest, xor_srcs, 0, count, STRIPE_SIZE, &submit);

        return tx;
}

static void ops_complete_compute(void *stripe_head_ref)
{
        struct stripe_head *sh = stripe_head_ref;

        /* 设置target执行dev的标记 */
        mark_target_uptodate(sh, sh->ops.target);
        mark_target_uptodate(sh, sh->ops.target2);

        /* 计算完成清除条带计算状态 */
        clear_bit(STRIPE_COMPUTE_RUN, &sh->state);
        /* 此时没有进行check流程，不满足if判断 */
        if (sh->check_state == check_state_compute_run)
                sh->check_state = check_state_compute_result;
        /* 设置条带状态推入状态机 */
        set_bit(STRIPE_HANDLE, &sh->state);
        release_stripe(sh);
}

static void mark_target_uptodate(struct stripe_head *sh, int target)
{
        /* 参数检查 */
        if (target < 0)
                return;

        tgt = &sh->dev[target];
        /* 设置dev包含最新数据状态 */
        set_bit(R5_UPTODATE, &tgt->flags);
        BUG_ON(!test_bit(R5_Wantcompute, &tgt->flags));
        /* 清除dev需要标记 */
        clear_bit(R5_Wantcompute, &tgt->flags);
}
```

### 3.3. 下发写请求

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ handle_parity_checks5()
 \_ ops_run_io()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* 此时uptodate等于disks不再进入if */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /*
         * s.syncing为真，s.locked为0，STRIPE_COMPUTE_RUN被清除，
         * STRIPE_INSYNC尚未设置，进入handle_parity_checks5
         */
        if (sh->check_state ||
            (s.syncing && s.locked == 0 &&
             !test_bit(STRIPE_COMPUTE_RUN, &sh->state) &&
             !test_bit(STRIPE_INSYNC, &sh->state))) {
                if (conf->level == 6)
                        handle_parity_checks6(conf, sh, &s, disks);
                else
                        handle_parity_checks5(conf, sh, &s, disks);
        }

        /* 在handle_parity_checks5中自增了locked未进入if */
        if ((s.syncing || s.replacing) && s.locked == 0 &&
            test_bit(STRIPE_INSYNC, &sh->state)) {
                md_done_sync(conf->mddev, STRIPE_SECTORS, 1);
                clear_bit(STRIPE_SYNCING, &sh->state);
                if (test_and_clear_bit(R5_Overlap, &sh->dev[sh->pd_idx].flags))
                        wake_up(&conf->wait_for_overlap);
        }

        /* 下发写请求 */
        ops_run_io(sh, &s);
}

static void handle_parity_checks5(struct r5conf *conf, struct stripe_head *sh,
                                struct stripe_head_state *s, int disks)
{
        struct r5dev *dev = NULL;

        set_bit(STRIPE_HANDLE, &sh->state);

        switch (sh->check_state) {
        case check_state_idle:
                /* 在analyse_stripe中dev仍处于未同步状态所以failed不为0 */
                if (s->failed == 0) {
                        BUG_ON(s->uptodate != disks);
                        sh->check_state = check_state_run;
                        set_bit(STRIPE_OP_CHECK, &s->ops_request);
                        clear_bit(R5_UPTODATE, &sh->dev[sh->pd_idx].flags);
                        s->uptodate--;
                        break;
                }
                /* 将异常磁盘赋值给dev */
                dev = &sh->dev[s->failed_num[0]];
                /* fall through */
        case check_state_compute_result:
                sh->check_state = check_state_idle;

                /* 条带已设置同步状态标记则退出 */
                if (test_bit(STRIPE_INSYNC, &sh->state))
                        break;

                /* 此时已经重构完，所以failed的dev必须包含最新数据且uptodate与磁盘数相等 */
                BUG_ON(!test_bit(R5_UPTODATE, &dev->flags));
                BUG_ON(s->uptodate != disks);

                /* 给需要下发写请求的dev上锁并设置需要写的标记 */
                set_bit(R5_LOCKED, &dev->flags);
                s->locked++;
                set_bit(R5_Wantwrite, &dev->flags);
                /* 清除条带降级标记 */
                clear_bit(STRIPE_DEGRADED, &sh->state);
                /* 设置条带处于同步状态标记 */
                set_bit(STRIPE_INSYNC, &sh->state);
                break;
        }
}

static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        /* 遍历条带/设备 */
        for (i = disks; i--; ) {
                /* 设置IO标记为写 */
                if (test_and_clear_bit(R5_Wantwrite, &sh->dev[i].flags)) {
                        if (test_and_clear_bit(R5_WantFUA, &sh->dev[i].flags))
                                rw = WRITE_FUA;
                        else
                                rw = WRITE;
                        if (test_bit(R5_Discard, &sh->dev[i].flags))
                                rw |= REQ_DISCARD;
                }

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

                        /* 提交bio */
                        generic_make_request(bi);
                }
        }
}
```

### 3.4. 同步结束

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ md_done_sync()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带 */
        analyse_stripe(sh, &s);

        /*
         * 在raid5_end_write_request中会清除R5_LOCKED标记，此时locked为0
         * 在上轮次处理中设置了STRIPE_INSYNC标记进入if
         */
        if ((s.syncing || s.replacing) && s.locked == 0 &&
            test_bit(STRIPE_INSYNC, &sh->state)) {
                /* 执行条带完成后的相关处理 */
                md_done_sync(conf->mddev, STRIPE_SECTORS, 1);
                /* 清除条带的同步中标记 */
                clear_bit(STRIPE_SYNCING, &sh->state);
        }
}

void md_done_sync(struct mddev *mddev, int blocks, int ok)
{
        /*
         * 重构流程中，提交的重构数量会累加到recovery_active中，
         * 这里每个条带完成后再减去相应的值
         */
        atomic_sub(blocks, &mddev->recovery_active);
        /*
         * 重构时会控制重构速度，当提交的请求过多时重构线程会进入到recovery_wait等待
         * 这里唤醒在等待的重构线程
         */
        wake_up(&mddev->recovery_wait);
        /* ok等于1不进入if */
        if (!ok) {
                /* 如果重构出现异常则打断重构 */
                set_bit(MD_RECOVERY_INTR, &mddev->recovery);
                set_bit(MD_RECOVERY_ERROR, &mddev->recovery);
                md_wakeup_thread(mddev->thread);
                // stop recovery, signal do_sync ....
        }
}
```

## 4. 激活热备盘

重构完成后需要将热备盘设置为正常状态开始使用，函数调用关系如下：

```C++
md_do_sync()
 \_ md_wakeup_thread() /* raid5d */
 \_ raid5d()
     \_ md_check_recovery()
         \_ md_reap_sync_thread()
             \_ raid5_spare_active()
```

代码逻辑如下：

```C++
static int raid5_spare_active(struct mddev *mddev)
{
        int i;
        struct r5conf *conf = mddev->private;
        struct disk_info *tmp;
        int count = 0;
        unsigned long flags;

        for (i = 0; i < conf->raid_disks; i++) {
                tmp = conf->disks + i;
                /* test_and_set_bit中设置成员磁盘状态为In_sync，即处于同步状态 */
                if (tmp->rdev
                    && tmp->rdev->recovery_offset == MaxSector
                    && !test_bit(Faulty, &tmp->rdev->flags)
                    && !test_and_set_bit(In_sync, &tmp->rdev->flags)) {
                        count++;
                        sysfs_notify_dirent_safe(tmp->rdev->sysfs_state);
                }
        }

        /* 计算md降级状态 */
        spin_lock_irqsave(&conf->device_lock, flags);
        mddev->degraded = calc_degraded(conf);
        spin_unlock_irqrestore(&conf->device_lock, flags);

        return count;
}
```
