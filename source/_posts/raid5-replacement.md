---
title: RAID5 I/O 处理之 replace 代码详解
date: 2024-07-26 15:40:54
tags: raid5
categories: md
description: 详细分析 RAID5 I/O 处理中磁盘替换处理部分的代码
---

## 1. 作用

从字面意思理解，replacement 即是替换。我们知道硬盘都有一定的使用寿命，可以在硬盘失效之前通过该功能将就盘的数据迁移至新盘。因为 replacement 的流程是从旧盘中读出数据直接写入新盘，因此比重构少很多读和校验值计算的操作，效率更高。

另外在 raid2.0 中，由于硬盘切片的使用方式，当系统只添加一块新盘时无法直接给 raid 扩容，需要先进行资源均衡，使得各盘空闲空间一致后再扩容，所以 replacement 同样适用于均衡场景中切片回收替换的逻辑。

## 2. 代码解析

### 2.1. 需替换设置

通过命令 `echo want_replacement > /sys/block/md0/md/dev-sdb/state` 设置硬盘标记为"需替换"状态，该 sys 命令会执行如下代码：

```C++
state_store()
    /* Replacement标记表明成员磁盘新盘，不能被设置为需替换 */
 \_ if (rdev->raid_disk >= 0 && !test_bit(Replacement, &rdev->flags))
        /* 给旧盘设置标记表明该成员磁盘是需要替换的 */
        set_bit(WantReplacement, &rdev->flags);
    /* 设置md为不要需要重构状态 */
 \_ set_bit(MD_RECOVERY_NEEDED, &rdev->mddev->recovery);
    /* 唤醒raid5d */
 \_ md_wakeup_thread(rdev->mddev->thread);
     \_ raid5d()
            /* 检查是否需要同步
             * 此时在sys接口的调用栈中，try_lock失败直接退出未创建同步线程
             */
         \_ md_check_recovery()  
```

### 2.2. 加入新盘

通过命令 `mdadm --manage -a /dev/md0 /dev/sde` 给块设备加入新盘，新盘加入后自动开始同步。

函数调用关系如下：

```C++
md_ioctl()
 \_ add_new_disk()

raid5d()
 \_ md_check_recovery()
     \_ remove_and_add_spares()
         \_ raid5_add_disk()
     \_ md_register_thread()
     \_ md_wakeup_thread(mddev->sync_thread)

md_do_sync()
```

这里关键函数为 `raid5_add_disk()` ，在函数内设置了相关 rdev 的各项标记，这里只说明该函数的相关逻辑，如下：

```C++
static int raid5_add_disk(struct mddev *mddev, struct md_rdev *rdev)
{
    struct r5conf *conf = mddev->private;
    struct disk_info *p;
    int first = 0;
    int last = conf->raid_disk - 1;
    
    /* 遍历所有磁盘 */
    for (disk = first; disk <= last; disk++) {
        p = conf->disk + disk;
        /* 如果设置了需替换标记且尚未指定新盘 */
        if (test_bit(WantReplacement, &rdev->flags) &&
             p->replacement == NULL) {
            /* 设置磁盘状态为未同步 */
            clear_bit(In_sync, &rdev->flags);
            /* 设置新盘在md中的磁盘索引 */
            rdev->raid_disk = disk;
            /* 设置md需要全盘同步 */
            config->fullsync = 1;
            /* 给replacement指针赋值使其指向新盘 */
            rcu_assign_pointer(p->replacement, rdev);
            break;
        }
    }
}
```

加入新盘后调用 `md_do_sync()` 会发起同步

### 2.3. 条带处理

在同步函数中，循环调用 `sync_request()` ，该函数主要逻辑如下：

```C++
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

#### 2.3.1. 下发读请求

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
                if (!test_bit(STRIPE_DISCARD, &sh->state) &&
                    /* 清掉STRIPE_SYNC_REQUESTED标记 */
                    test_and_clear_bit(STRIPE_SYNC_REQUESTED, &sh->state)) {
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

        /* s.replacing为真进入handle_stripe_fill */
        if (s.to_read || s.non_overwrite
            || (conf->level == 6 && s.to_write && s.failed)
            || (s.syncing && (s.uptodate + s.compute < disks))
            || s.replacing
            || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /* 此时 s.locked == 0 条件不成立不会进入该if分支 */
        if (s.replacing && s.locked == 0
            && !test_bit(STRIPE_INSYNC, &sh->state)) {
                /* Write out to replacement devices where possible */
                for (i = 0; i < conf->raid_disks; i++)
                        if (test_bit(R5_UPTODATE, &sh->dev[i].flags) &&
                            test_bit(R5_NeedReplace, &sh->dev[i].flags)) {
                                set_bit(R5_WantReplace, &sh->dev[i].flags);
                                set_bit(R5_LOCKED, &sh->dev[i].flags);
                                s.locked++;
                        }
                set_bit(STRIPE_INSYNC, &sh->state);
        }
        /* 此时 s.locked == 0 条件不成立不会进入该if分支 */
        if ((s.syncing || s.replacing) && s.locked == 0 &&
            test_bit(STRIPE_INSYNC, &sh->state)) {
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
                /* 加入新盘的成员磁盘replacement存在但不满足
                 * rdev->recovery_offset >= sh->sector + STRIPE_SECTORS（同步时同步进度小于sh->sector）
                 * 走到else分支
                 */
                rdev = rcu_dereference(conf->disks[i].replacement);
                if (rdev && !test_bit(Faulty, &rdev->flags) &&
                    rdev->recovery_offset >= sh->sector + STRIPE_SECTORS &&
                    !is_badblock(rdev, sh->sector, STRIPE_SECTORS,
                                 &first_bad, &bad_sectors))
                        set_bit(R5_ReadRepl, &dev->flags);
                else {
                        if (rdev)
                                /* 设置R5_NeedReplace标记 */
                                set_bit(R5_NeedReplace, &dev->flags);
                        rdev = rcu_dereference(conf->disks[i].rdev);
                        clear_bit(R5_ReadRepl, &dev->flags);
                }

                /* 在replacement处理中所有硬盘都是正常的，do_recovery为0，s->failed也为0 */
                if (!test_bit(R5_Insync, &dev->flags)) {
                        if (s->failed < 2)
                                s->failed_num[s->failed] = i;
                        s->failed++;
                        if (rdev && !test_bit(Faulty, &rdev->flags))
                                do_recovery = 1;
                }
        }

        /* 在handle_stripe中设置了该标记 */
        if (test_bit(STRIPE_SYNCING, &sh->state)) {
                /* 条件都未成立走else分支 */
                if (do_recovery ||
                    sh->sector >= conf->mddev->recovery_cp ||
                    test_bit(MD_RECOVERY_REQUESTED, &(conf->mddev->recovery)))
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

        /* 此时所有条带/设备都未发起请求且未包含最新数据 */
        if (!test_bit(R5_LOCKED, &dev->flags) &&
            !test_bit(R5_UPTODATE, &dev->flags) &&
            (dev->toread ||
             (dev->towrite && !test_bit(R5_OVERWRITE, &dev->flags)) ||
             s->syncing || s->expanding ||
             /* want_replace()函数中会判断disk_idx对应的成员磁盘是否有replacemenet且
              * 条带起始位置大于等于replacement重构位置返回1
              * 在replacing过程中设置了replacement的成员磁盘进入if
              */
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
                /* 对设置了replacement的成员磁盘下发读请求 */
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

#### 2.3.2. 下发写请求

函数调用关系：

```C++
handle_stripe()
 \_ analyse_stripe()
 \_ ops_run_io()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* 在fetch_block中设置了replacement的条带已经是最新数据
         * R5_UPTODATE条件为真，所以不会设置任何新的需要读取
         */
        if (s.to_read || s.non_overwrite
                 || (conf->level == 6 && s.to_write && s.failed)
                 || (s.syncing && (s.uptodate + s.compute < disks))
                 || s.replacing
                 || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /* replacing在analyse_stripe设置
         * 未下发新的读请求locked为真
         * 条带同步状态再第一轮中清除
         * 进入if
         */
        if (s.replacing && s.locked == 0
                 && !test_bit(STRIPE_INSYNC, &sh->state)) {
                for (i = 0; i < conf->raid_disks; i++)
                        /* 读成功后R5_UPTODATE为真，R5_WantReplace在analyse_stripe中设置 */
                        if (test_bit(R5_UPTODATE, &sh->dev[i].flags) &&
                                 test_bit(R5_NeedReplace, &sh->dev[i].flags)) {
                                /* 设置R5_WantReplace标记在下发写请求时判断 */
                                set_bit(R5_WantReplace, &sh->dev[i].flags);
                                /* 设置R5_LOCKED标记表明条带/设备准备调度IO */
                                set_bit(R5_LOCKED, &sh->dev[i].flags);
                                /* 自增locked计数 */
                                s.locked++;
                        }
                /* 将条带设置为同步状态 */
                set_bit(STRIPE_INSYNC, &sh->state);
        }
        /* 上个if中locked自增不进入if */
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

/* 和上轮一轮不在赘述 */
analyse_stripe()

static void ops_run_io(struct stripe_head *sh, struct stripe_head_state *s)
{
        struct r5conf *conf = sh->raid_conf;
        int i, disks = sh->disks;

        might_sleep();

        /* 遍历所有条带/设备 */
        for (i = disks; i--; ) {
                if (test_and_clear_bit(R5_WantReplace, &sh->dev[i].flags)) {
                        rw = WRITE;
                        replace_only = 1;
                }

                bi = &sh->dev[i].req;
                rbi = &sh->dev[i].rreq; /* For writing to replacement */

                rcu_read_lock();
                rrdev = rcu_dereference(conf->disks[i].replacement);
                smp_mb(); /* Ensure that if rrdev is NULL, rdev won't be */
                rdev = rcu_dereference(conf->disks[i].rdev);

                if (rw & WRITE) {
                        /* 将rdev标空后只能使用rrdev */
                        if (replace_only)
                                rdev = NULL;
                }
                rcu_read_unlock();

                /* 使用rbi向rrdev下发写请求 */
                if (rrdev) {
                        set_bit(STRIPE_IO_STARTED, &sh->state);

                        bio_reset(rbi);
                        rbi->bi_bdev = rrdev->bdev;
                        rbi->bi_rw = rw;
                        BUG_ON(!(rw & WRITE));
                        rbi->bi_end_io = raid5_end_write_request;
                        rbi->bi_private = sh;

                        atomic_inc(&sh->count);
                        if (use_new_offset(conf, sh))
                                rbi->bi_sector = (sh->sector + rrdev->new_data_offset);
                        else
                                rbi->bi_sector = (sh->sector + rrdev->data_offset);
                        rbi->bi_vcnt = 1;
                        rbi->bi_io_vec[0].bv_len = STRIPE_SIZE;
                        rbi->bi_io_vec[0].bv_offset = 0;
                        rbi->bi_size = STRIPE_SIZE;

                        generic_make_request(rbi);
                }
        }
}
```

#### 2.3.3. 完成

函数调用关系：

```C++
handle_stripe()
 \_ md_done_sync()
```

代码逻辑如下：

```C++
static void handle_stripe(struct stripe_head *sh)
{
        /* 解析条带状态 */
        analyse_stripe(sh, &s);

        /* 在fetch_block中设置了replacement的条带已经是最新数据
         * R5_UPTODATE条件为真，所以不会设置任何新的需要读取
         */
        if (s.to_read || s.non_overwrite
                 || (conf->level == 6 && s.to_write && s.failed)
                 || (s.syncing && (s.uptodate + s.compute < disks))
                 || s.replacing
                 || s.expanding)
                handle_stripe_fill(sh, &s, disks);

        /* replacing在analyse_stripe设置
         * 未下发新的读请求locked为真
         * STRIPE_INSYNC在上轮处理中设置，本轮不在进入if
         */
        if (s.replacing && s.locked == 0
            && !test_bit(STRIPE_INSYNC, &sh->state)) {
                /* Write out to replacement devices where possible */
                for (i = 0; i < conf->raid_disks; i++)
                        if (test_bit(R5_UPTODATE, &sh->dev[i].flags) &&
                            test_bit(R5_NeedReplace, &sh->dev[i].flags)) {
                                set_bit(R5_WantReplace, &sh->dev[i].flags);
                                set_bit(R5_LOCKED, &sh->dev[i].flags);
                                s.locked++;
                        }
                set_bit(STRIPE_INSYNC, &sh->state);
        }

        /* replacing、locked、STRIPE_INSYNC条件都成立 */
        if ((s.syncing || s.replacing) && s.locked == 0 &&
                 test_bit(STRIPE_INSYNC, &sh->state)) {
                md_done_sync(conf->mddev, STRIPE_SECTORS, 1);
                clear_bit(STRIPE_SYNCING, &sh->state);
        }
}

void md_done_sync(struct mddev *mddev, int blocks, int ok)
{
        /* 自增完成计数 */
        atomic_sub(blocks, &mddev->recovery_active);
        /* 通知同步前程继续进行 */
        wake_up(&mddev->recovery_wait);
        if (!ok) {
                set_bit(MD_RECOVERY_INTR, &mddev->recovery);
                set_bit(MD_RECOVERY_ERROR, &mddev->recovery);
                md_wakeup_thread(mddev->thread);
        }
}
```

至此一个条带的同步流程分析完毕
