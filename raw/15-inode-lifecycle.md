# inode 수명: 원본 출력

> 본문은 ../15-inode-lifecycle.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## drop_caches 뒤 inode의 reclaim과 miss

**관찰 출력**

```text
xfssyncd_centisecs=3000
== phase E1 (lines=3) all inodes
      1  xfs_destroy_inode: 
      1  xfs_inode_reclaiming: 
      1  xfs_inode_set_reclaimable: 
-- phase E1 lines for ino 0x83
sh-4070    [001] .....  1278.693282: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
sh-4070    [001] ...2.  1278.693284: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x0
sh-4070    [001] .....  1278.694663: xfs_inode_reclaiming: dev 7:0 ino 0x83 iflags 0x5
== phase E2 (lines=1) all inodes
      1  xfs_iget_miss: 
-- phase E2 lines for ino 0x83
stat-4073    [002] .....  1278.698126: xfs_iget_miss: dev 7:0 ino 0x83 iflags 0x8
== phase E3 (lines=3) all inodes
      1  xfs_destroy_inode: 
      1  xfs_inode_reclaiming: 
      1  xfs_inode_set_reclaimable: 
-- phase E3 lines for ino 0x83
sh-4081    [004] .....  1279.704919: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
sh-4081    [004] ...2.  1279.704922: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x0
sh-4081    [004] .....  1279.704964: xfs_inode_reclaiming: dev 7:0 ino 0x83 iflags 0x5
== phase E4 (lines=0) all inodes
-- phase E4 lines for ino 0x83
== phase E5 (lines=1) all inodes
      1  xfs_iget_miss: 
-- phase E5 lines for ino 0x83
stat-4087    [005] .....  1287.714619: xfs_iget_miss: dev 7:0 ino 0x83 iflags 0x8
```

## (A) 닫힌 file 하나를 지울 때의 event 순서

**관찰 출력**

```text
== phase A sequence
rm-3591    [005] .....  1233.035311: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
rm-3591    [005] .....  1233.035314: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
rm-3591    [005] .....  1233.035333: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
rm-3591    [005] .....  1233.035333: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x0
rm-3591    [005] ...1.  1233.035334: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/5:1-137     [005] .....  1233.036605: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/5:1-137     [005] .....  1233.036606: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2400
kworker/5:1-137     [005] .....  1233.036646: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/5:1-137     [005] .....  1233.036646: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/5:1-137     [005] .....  1233.036646: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
kworker/5:1-137     [005] ...2.  1233.036648: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x2400
```

## (B) 열린 채 지운 file을 닫을 때의 event 순서

**관찰 출력**

```text
== phase B (lines=2)
      1  xfs_iunlink: 
      1  xfs_iunlink_update_bucket: 
== phase B2 sequence
bash-3506    [007] .....  1236.046089: xfs_destroy_inode: dev 7:0 ino 0x84 iflags 0x0
bash-3506    [007] .....  1236.046091: xfs_inode_set_need_inactive: dev 7:0 ino 0x84 iflags 0x0
bash-3506    [007] ...1.  1236.046092: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/7:1-144     [007] .....  1236.048603: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/7:1-144     [007] .....  1236.048603: xfs_inode_inactivating: dev 7:0 ino 0x84 iflags 0x2400
kworker/7:1-144     [007] .....  1236.048630: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x84
kworker/7:1-144     [007] .....  1236.048630: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x84
kworker/7:1-144     [007] .....  1236.048631: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 4 old 0x84 new 0xffffffff
kworker/7:1-144     [007] ...2.  1236.048636: xfs_inode_set_reclaimable: dev 7:0 ino 0x84 iflags 0x2400
```

## (C) freeze 중에 닫은 file의 event 순서

**관찰 출력**

```text
== phase C (lines=3)
      1  xfs_inodegc_stop: 
      1  xfs_iunlink: 
      1  xfs_iunlink_update_bucket: 
== phase C2 (lines=2)
      1  xfs_destroy_inode: 
      1  xfs_inode_set_need_inactive: 
== phase C3 (lines=7)
      1  xfs_inode_inactivating: 
      1  xfs_inode_set_reclaimable: 
      1  xfs_inodegc_start: 
      1  xfs_inodegc_worker: 
      2  xfs_iunlink_remove: 
      1  xfs_iunlink_update_bucket: 
== phase C2 sequence
sleep-3602    [003] .....  1239.093830: xfs_destroy_inode: dev 7:0 ino 0x85 iflags 0x0
sleep-3602    [003] .....  1239.093831: xfs_inode_set_need_inactive: dev 7:0 ino 0x85 iflags 0x0
== phase C3 sequence
xfs_io-3622    [007] .....  1241.103438: xfs_inodegc_start: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_unfreeze+0x4c/0x60
kworker/3:1-70      [003] .....  1241.103445: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/3:1-70      [003] .....  1241.103445: xfs_inode_inactivating: dev 7:0 ino 0x85 iflags 0x2400
kworker/3:1-70      [003] .....  1241.103509: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x85
kworker/3:1-70      [003] .....  1241.103509: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x85
kworker/3:1-70      [003] .....  1241.103510: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 5 old 0x85 new 0xffffffff
kworker/3:1-70      [003] ...2.  1241.103512: xfs_inode_set_reclaimable: dev 7:0 ino 0x85 iflags 0x2400
```

## (D) 빈 file 200개를 지울 때의 event 수와 처리 순서

**관찰 출력**

```text
== phase D (lines=2273)
    200  xfs_destroy_inode: 
    200  xfs_inode_inactivating: 
      2  xfs_inode_reclaiming: 
    200  xfs_inode_set_need_inactive: 
    200  xfs_inode_set_reclaimable: 
    200  xfs_inodegc_queue: 
      1  xfs_inodegc_worker: 
    200  xfs_iunlink: 
    400  xfs_iunlink_remove: 
    400  xfs_iunlink_update_bucket: 
    270  xfs_iunlink_update_dinode: 
== phase D inactivating order first 5 / last 5 (count=200)
ino 0x1cd
ino 0x1cc
ino 0x1cb
ino 0x1ca
ino 0x1c9
ino 0x8a
ino 0x89
ino 0x88
ino 0x87
ino 0x86
== phase D worker task names
      1 kworker/3:1-70
```

## 열린 채 지운 file 200개의 AGI unlinked bucket

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
isize=512 agcount=1 bsize=4096 bsize=4096 bsize=4096 
pid=4477 deleted fds=200 names in dir=0
freeze rc=0
unlinked[0-63] = 0:131392 1:131393 2:131394 3:131395 4:131396 5:131397 6:131398 7:131399 8:131400 9:131401 10:131402 11:131339 12:131340 13:131341 14:131342 15:131343 16:131344 17:131345 18:131346 19:131347 20:131348 21:131349 22:131350 23:131351 24:131352 25:131353 26:131354 27:131355 28:131356 29:131357 30:131358 31:131359 32:131360 33:131361 34:131362 35:131363 36:131364 37:131365 38:131366 39:131367 40:131368 41:131369 42:131370 43:131371 44:131372 45:131373 46:131374 47:131375 48:131376 49:131377 50:131378 51:131379 52:131380 53:131381 54:131382 55:131383 56:131384 57:131385 58:131386 59:131387 60:131388 61:131389 62:131390 63:131391

nonempty buckets=64
bucket != agino%64 count=0
dump_iunlinked total aginos=200
-- chain length histogram (count length)
     56 3
      8 4
== bucket 0 head 131392
AG 0 bucket 0 agino 131392
    [0] agino 131328
    [1] agino 131264
    [2] agino NULL
core.nlinkv2 = 0
v3.inumber = 131392
next_unlinked = 131328
core.nlinkv2 = 0
v3.inumber = 131328
next_unlinked = 131264
core.nlinkv2 = 0
v3.inumber = 131264
next_unlinked = null
chain walked=3 last next=null
thaw rc=0
after close (frozen): nonempty buckets=0 dump_iunlinked aginos=0 names in dir=0
umount rc=0
after umount: nonempty buckets=0
done
exit=0
```

## shutdown 뒤 recovery mount에서 unlinked list가 정리되는 과정

**관찰 출력**

```text
deleted fds=200
Opening "/mnt/p15rc"
Calling XFS_IOC_GOINGDOWN
godown rc=0
multi_open_unlink rc=0
umount rc=0
    log tail: 2 head: 153 state: <DIRTY>
TRAN count=0 CLEAR lines=0
-- on-disk agi before recovery: nonempty buckets=0
unlinked[0-63] = 
recovery mount rc=0
[ 1386.514350] XFS (loop0): Mounting V5 Filesystem d79b0d72-9296-4a28-8f43-89d06f42f406
[ 1386.515583] XFS (loop0): Starting recovery (logdev: internal)
[ 1386.516975] XFS (loop0): Ending recovery (logdev: internal)
-- event counts
    203  xfs_iget_miss: 
    200  xfs_inode_inactivating: 
    200  xfs_inode_set_reclaimable: 
    200  xfs_inodegc_flush: 
    200  xfs_inodegc_worker: 
    400  xfs_iunlink_remove: 
    200  xfs_iunlink_update_bucket: 
    133  xfs_iunlink_update_dinode: 
      1  xfs_log_recover: 
-- event sequence (run length)
      1  xfs_log_recover: 
      5  xfs_iget_miss: 
      1  xfs_inodegc_flush: 
      1  xfs_inodegc_worker: 
      1  xfs_inode_inactivating: 
      2  xfs_iunlink_remove: 
      1  xfs_iunlink_update_bucket: 
      1  xfs_iunlink_update_dinode: 
      1  xfs_inode_set_reclaimable: 
      1  xfs_iget_miss: 
-- first iunlink lines
kworker/0:2-193     [000] .....  1386.574072: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x20140
kworker/0:2-193     [000] .....  1386.574072: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x20140
kworker/0:2-193     [000] .....  1386.574073: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 0 old 0x20140 new 0x20100
kworker/0:2-193     [000] .....  1386.574075: xfs_iunlink_update_dinode: dev 7:0 agno 0x0 agino 0x20140 old 0x20100 new 0xffffffff
kworker/0:2-193     [000] .....  1386.574085: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x20100
kworker/0:2-193     [000] .....  1386.574085: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x20100
kworker/0:2-193     [000] .....  1386.574085: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 0 old 0x20100 new 0x200c0
kworker/0:2-193     [000] .....  1386.574085: xfs_iunlink_update_dinode: dev 7:0 agno 0x0 agino 0x20100 old 0x200c0 new 0xffffffff
kworker/0:2-193     [000] .....  1386.574090: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x200c0
kworker/0:2-193     [000] .....  1386.574090: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x200c0
kworker/0:2-193     [000] .....  1386.574090: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 0 old 0x200c0 new 0xffffffff
kworker/0:2-193     [000] .....  1386.574095: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x20141
-- task names of xfs_iunlink_remove
    400 kworker/0:2
names after recovery=0
Opening "/mnt/p15rc"
Calling XFS_IOC_GOINGDOWN
godown2 rc=0
    log tail: 217 head: 261 state: <DIRTY>
TRAN count=0 CLEAR lines=0
-- on-disk agi after recovery: nonempty buckets=0 dump_iunlinked aginos=0
```

