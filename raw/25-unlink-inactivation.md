# unlink 뒤의 흐름: 원본 출력

> 본문은 ../25-unlink-inactivation.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 닫힌 file 하나를 rm한 뒤의 순서

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
isize=512 agcount=4 crc=1 rmapbt=1 reflink=1 bsize=4096 bsize=4096 bsize=4096 
loop=/dev/loop0 dev=7:0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
== phase A sequence
rm-4450    [002] .....   887.808449: xfs_remove: dev 7:0 dp ino 0x80 name big
rm-4450    [002] .....   887.808454: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
rm-4450    [002] .....   887.808457: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
rm-4450    [002] .....   887.808479: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
rm-4450    [002] .....   887.808480: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x0
rm-4450    [002] ...1.   887.808480: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/2:1-71      [002] .....   887.809513: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/2:1-71      [002] .....   887.809513: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2400
kworker/2:1-71      [002] .....   887.809517: xfs_itruncate_extents_start: dev 7:0 ino 0x83 disize 0x0 new_size 0x0
kworker/2:1-71      [002] .....   887.809538: xfs_extent_free_deferred: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100 flags 0x0
kworker/2:1-71      [002] .....   887.809542: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100 resv 0 none
kworker/2:1-71      [002] .....   887.809542: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
kworker/2:1-71      [002] .....   887.809549: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/2:1-71      [002] .....   887.809550: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/2:1-71      [002] .....   887.809550: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
kworker/2:1-71      [002] ...2.   887.809552: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x2400
== phase A2 sequence
kworker/6:0H-50      [006] ...1.   889.823898: xfs_extent_busy_clear: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100
phase A xfs_free_extent fsbcount sum=256
```

## 빈 file 128개를 하나씩 rm한 구간

**관찰 출력**

```text
big=0x83 f0=0x84 f127=0x903 rootdir size=4096
used blocks after=12982 rootdir size=6
== phase B (lines=1469)
    128  xfs_destroy_inode: 
      3  xfs_extent_busy: 
      3  xfs_extent_busy_clear: 
      3  xfs_extent_free_deferred: 
      3  xfs_free_extent: 
    128  xfs_inode_inactivating: 
    128  xfs_inode_set_need_inactive: 
    128  xfs_inode_set_reclaimable: 
    128  xfs_inodegc_queue: 
     49  xfs_inodegc_worker: 
    128  xfs_iunlink: 
    256  xfs_iunlink_remove: 
    256  xfs_iunlink_update_bucket: 
    128  xfs_remove: 
== phase B task x event
      2 kworker xfs_extent_busy
      3 kworker xfs_extent_busy_clear
      2 kworker xfs_extent_free_deferred
      2 kworker xfs_free_extent
    128 kworker xfs_inode_inactivating
    128 kworker xfs_inode_set_reclaimable
     49 kworker xfs_inodegc_worker
    256 kworker xfs_iunlink_remove
    128 kworker xfs_iunlink_update_bucket
    128 rm xfs_destroy_inode
      1 rm xfs_extent_busy
      1 rm xfs_extent_free_deferred
      1 rm xfs_free_extent
    128 rm xfs_inode_set_need_inactive
    128 rm xfs_inodegc_queue
    128 rm xfs_iunlink
    128 rm xfs_iunlink_update_bucket
    128 rm xfs_remove
== phase B xfs_free_extent lines
rm-4562    [000] .....   890.867196: xfs_free_extent: dev 7:0 agno 0x0 agbno 0xf fsbcount 0x1 resv 0 left
kworker/3:1-70      [003] .....   890.876566: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x118 fsbcount 0x8 resv 0 left
kworker/4:1-99      [004] .....   890.877537: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x120 fsbcount 0x8 resv 0 both
```

## 열린 채 rm한 file과 multi_open_unlink

**관찰 출력**

```text
before rm: used=13238 du_kb=1024
after rm, still open: used=13238 du_kb=0
lr-x------ 1 ubuntu ubuntu 64 Oct 22 08:36 4 -> /mnt/p25u/held (deleted)
fd4 links=0 size=1048576 blocks=2048 ino=131
after close: used=12982 du_kb=0
mou deleted fds=200 names in dir=0
== phase P1 (lines=3)
      1  xfs_iunlink: 
      1  xfs_iunlink_update_bucket: 
      1  xfs_remove: 
== phase P1e (lines=0)
== phase P3 (lines=600)
    200  xfs_iunlink: 
    200  xfs_iunlink_update_bucket: 
    200  xfs_remove: 
== phase P4 (lines=1468)
    200  xfs_destroy_inode: 
      3  xfs_free_extent: 
    200  xfs_inode_inactivating: 
    200  xfs_inode_set_need_inactive: 
    200  xfs_inode_set_reclaimable: 
    200  xfs_inodegc_queue: 
      1  xfs_inodegc_worker: 
    400  xfs_iunlink_remove: 
     64  xfs_iunlink_update_bucket: 
== phase P1 sequence
rm-4839    [004] .....   905.806562: xfs_remove: dev 7:0 dp ino 0x80 name held
rm-4839    [004] .....   905.806566: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
rm-4839    [004] .....   905.806569: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
== phase P2 sequence
sleep-4796    [002] .....   907.815642: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
sleep-4796    [002] .....   907.815643: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x0
sleep-4796    [002] ...1.   907.815644: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/2:1-71      [002] .....   907.818517: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/2:1-71      [002] .....   907.818517: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2400
kworker/2:1-71      [002] .....   907.818520: xfs_itruncate_extents_start: dev 7:0 ino 0x83 disize 0x0 new_size 0x0
kworker/2:1-71      [002] .....   907.818551: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x100 resv 0 right
kworker/2:1-71      [002] .....   907.818560: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/2:1-71      [002] .....   907.818560: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/2:1-71      [002] .....   907.818561: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
kworker/2:1-71      [002] ...2.   907.818563: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x2400
phase P2 xfs_free_extent fsbcount sum=256
== phase P1 task x event
      1 rm xfs_iunlink
      1 rm xfs_iunlink_update_bucket
      1 rm xfs_remove
== phase P3 task x event
    200 multi_open_unli xfs_iunlink
    200 multi_open_unli xfs_iunlink_update_bucket
    200 multi_open_unli xfs_remove
== phase P4 task x event
      3 kworker xfs_free_extent
    200 kworker xfs_inode_inactivating
    200 kworker xfs_inode_set_reclaimable
      1 kworker xfs_inodegc_worker
    400 kworker xfs_iunlink_remove
     64 kworker xfs_iunlink_update_bucket
    200 multi_open_unli xfs_destroy_inode
    200 multi_open_unli xfs_inode_set_need_inactive
    200 multi_open_unli xfs_inodegc_queue
```

## O_TMPFILE와 linkat

**관찰 출력**

```text
linked links=1 size=4096 ino=131
== phase T1 sequence
xfs_io-5286    [006] .....   940.522856: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
xfs_io-5286    [006] .....   940.522857: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
xfs_io-5286    [006] .....   940.523249: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x0
xfs_io-5286    [006] .....   940.523250: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x0
xfs_io-5286    [006] ...1.   940.523250: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/6:1-159     [006] .....   940.524509: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/6:1-159     [006] .....   940.524510: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2400
kworker/6:1-159     [006] .....   940.524522: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x10 resv 0 right
kworker/6:1-159     [006] .....   940.524527: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/6:1-159     [006] .....   940.524527: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/6:1-159     [006] .....   940.524528: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
kworker/6:1-159     [006] ...2.   940.524532: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x2400
== phase T2 sequence
xfs_io-5290    [004] .....   942.528418: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
xfs_io-5290    [004] .....   942.528419: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
xfs_io-5290    [004] .....   942.528557: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
xfs_io-5290    [004] .....   942.528557: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
xfs_io-5290    [004] .....   942.528558: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
== phase T3 sequence
rm-5297    [005] .....   944.535057: xfs_iunlink: dev 7:0 agno 0x0 agino 0x83
rm-5297    [005] .....   944.535063: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0xffffffff new 0x83
rm-5297    [005] .....   944.535080: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x40
rm-5297    [005] .....   944.535081: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x40
rm-5297    [005] ...1.   944.535082: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/5:1-131     [005] .....   944.537518: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/5:1-131     [005] .....   944.537519: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2440
kworker/5:1-131     [005] .....   944.537534: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/5:1-131     [005] .....   944.537534: xfs_iunlink_remove: dev 7:0 agno 0x0 agino 0x83
kworker/5:1-131     [005] .....   944.537535: xfs_iunlink_update_bucket: dev 7:0 agno 0x0 bucket 3 old 0x83 new 0xffffffff
kworker/5:1-131     [005] ...2.   944.537537: xfs_inode_set_reclaimable: dev 7:0 ino 0x83 iflags 0x2440
phase T1 xfs_free_extent fsbcount sum=16
phase T3 xfs_free_extent fsbcount sum=0
```

## 64 MiB file을 연 채 rm하고 freeze 중에 닫기

**관찰 출력**

```text
F1 rm while open: used=45881
F1 frozen, after close: used=37689
F1 after thaw: used=29497
== phase F1 sequence
xfs_io-5575    [004] .....   954.986544: xfs_inodegc_push: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_statfs+0x29/0x1e0
xfs_io-5575    [004] .....   954.986556: xfs_inodegc_push: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_statfs+0x29/0x1e0
xfs_io-5575    [004] .....   954.986559: xfs_inodegc_push: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_statfs+0x29/0x1e0
xfs_io-5575    [004] .....   955.014534: xfs_inodegc_stop: dev 7:0 m_features 0xd9ffaab opstate (clean|blockgc) s_flags 0x70810000 caller xfs_fs_sync_fs+0x56/0xc0
sleep-5531    [003] .....   955.028782: xfs_destroy_inode: dev 7:0 ino 0x83 iflags 0x40
sleep-5531    [003] .....   955.028783: xfs_inode_set_need_inactive: dev 7:0 ino 0x83 iflags 0x40
== phase F1b sequence
xfs_io-5587    [006] .....   957.039261: xfs_inodegc_start: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_unfreeze+0x4c/0x60
kworker/3:1-70      [003] .....   957.039269: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/3:1-70      [003] .....   957.039270: xfs_inode_inactivating: dev 7:0 ino 0x83 iflags 0x2440
kworker/3:1-70      [003] .....   957.039307: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x18 fsbcount 0x4000 resv 0 none
phase F1b xfs_free_extent fsbcount sum=16384
```

## 빈 file 1000개를 CPU 하나에 고정해 rm

**관찰 출력**

```text
nproc=8 preempt=full (lazy)  version=#1 SMP PREEMPT_DYNAMIC Wed Oct 21 17:14:58 KST 2026
F2 rm rc=0 names left=1
== phase F2 queue/throttle/worker run-length
    258  xfs_inodegc_queue: 
      1  xfs_inodegc_throttle: 
      1  xfs_inodegc_worker: 
     34  xfs_inodegc_queue: 
      1  xfs_inodegc_worker: 
     34  xfs_inodegc_queue: 
      1  xfs_inodegc_worker: 
     34  xfs_inodegc_queue: 
== phase F2 worker and throttle lines
rm-5597    [001] .....   959.051934: xfs_inodegc_throttle: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
kworker/1:1-74      [001] .....   959.051937: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/1:1-74      [001] .....   959.052551: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
== phase F2 task x event
     15 kworker xfs_free_extent
   1000 kworker xfs_inode_inactivating
     23 kworker xfs_inodegc_worker
   1000 rm xfs_destroy_inode
     11 rm xfs_free_extent
   1000 rm xfs_inode_set_need_inactive
   1000 rm xfs_inodegc_queue
      1 rm xfs_inodegc_throttle
```

## 64 MiB file(닫힘)을 rm한 직후의 df와 push

**관찰 출력**

```text
F3 before rm: used=29366
F3 right after rm: used=29366
F3 1s later: used=12982
== phase F3 sequence
rm-5610    [000] .....   961.066329: xfs_destroy_inode: dev 7:0 ino 0x84 iflags 0x40
rm-5610    [000] .....   961.066330: xfs_inode_set_need_inactive: dev 7:0 ino 0x84 iflags 0x40
rm-5610    [000] ...1.   961.066331: xfs_inodegc_queue: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller destroy_inode+0x3b/0x70
df-5612    [007] .....   961.067142: xfs_inodegc_push: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_statfs+0x29/0x1e0
kworker/0:0-9       [000] .....   961.067156: xfs_inodegc_worker: dev 7:0 shrinker_hits 0
kworker/0:0-9       [000] .....   961.067157: xfs_inode_inactivating: dev 7:0 ino 0x84 iflags 0x2440
kworker/0:0-9       [000] .....   961.067191: xfs_free_extent: dev 7:0 agno 0x0 agbno 0x8008 fsbcount 0x4000 resv 0 both
df-5617    [003] .....   962.069536: xfs_inodegc_push: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller xfs_fs_statfs+0x29/0x1e0
```

