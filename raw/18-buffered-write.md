# buffered write: 원본 출력

> 본문은 ../18-buffered-write.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 관찰에 쓴 file system의 mkfs 출력

**관찰 출력**

```text
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0
```

## 1 MiB write 직후의 statfs, stat, bmap

**관찰 출력**

```text
statfs.f_bavail = 232778
wrote 1048576/1048576 bytes at offset 0
stat.blocks = 3968
fsxattr.nextents = 1
statfs.f_bavail = 232274
/mnt/bw18a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       delalloc                              2048
 FLAG Values:
    0100000 Shared extent
    0010000 Unwritten preallocated extent
```

## fsync 뒤 1 MiB file의 stat, statfs, bmap

**관찰 출력**

```text
stat.blocks = 3968
fsxattr.nextents = 2
statfs.f_bavail = 232282
/mnt/bw18a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
```

## close 뒤와 umount 뒤의 1 MiB file

**관찰 출력**

```text
== reopen after first close
stat.blocks = 2048
fsxattr.nextents = 1
statfs.f_bavail = 232522
/mnt/bw18a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
ino=131
core.size = 1048576
core.nblocks = 256
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,24,256,0]
```

## pwrite 0 8k의 예약 event

**관찰 출력**

```text
wrote 8192/8192 bytes at offset 0
8 KiB, 2 ops; 0.0000 sec (252.016 MiB/sec and 64516.1290 ops/sec)
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-52772   [001] .....  8630.385527: xfs_update_time: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52772   [001] .....  8630.385534: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000
          xfs_io-52772   [001] .....  8630.385541: xfs_iext_insert: dev 7:0 ino 0x83 state  cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_hole_delay.constprop.0+0x127/0x450
          xfs_io-52772   [001] .....  8630.385542: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
          xfs_io-52772   [001] .....  8630.385547: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000
          xfs_io-52772   [001] .....  8630.385548: xfs_bmap_pre_update: dev 7:0 ino 0x83 state LC|0x50 cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_hole_delay.constprop.0+0x217/0x450
          xfs_io-52772   [001] .....  8630.385548: xfs_bmap_post_update: dev 7:0 ino 0x83 state LC|0x50 cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xffffffffe0007 fsbcount 0x2 flag 0 caller xfs_bmap_add_extent_hole_delay.constprop.0+0x217/0x450
          xfs_io-52772   [001] .....  8630.385548: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000 fork data startoff 0x1 startblock 0xffffffffe0007 fsbcount 0x1
```

## fsync의 writeback event

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-52772   [001] .....  8630.385556: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52772   [001] .....  8630.385612: xfs_bmap_pre_update: dev 7:0 ino 0x83 state LF|RF cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xffffffffe0007 fsbcount 0x2 flag 0 caller xfs_bmap_add_extent_delay_real+0x83c/0x18c0
          xfs_io-52772   [001] .....  8630.385612: xfs_bmap_post_update: dev 7:0 ino 0x83 state LF|RF cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xd fsbcount 0x2 flag 1 caller xfs_bmap_add_extent_delay_real+0x83c/0x18c0
          xfs_io-52772   [001] .....  8630.385635: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x2
          xfs_io-52772   [001] .....  8630.385636: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x0 dirty len 0x1000 addr 0xd000 offset 0x0 length 0x2000 type UNWRITTEN (0x3) flags DIRTY (0x2)
          xfs_io-52772   [001] .....  8630.385638: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x1000 dirty len 0x1000 addr 0xd000 offset 0x0 length 0x2000 type UNWRITTEN (0x3) flags DIRTY (0x2)
```

## 변환과 size 갱신 event

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
     kworker/3:1-71      [003] .....  8630.385651: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x2000 disize 0x0 pos 0x0 bytecount 0x2000
     kworker/3:1-71      [003] .....  8630.385654: xfs_bmap_pre_update: dev 7:0 ino 0x83 state LF|RF cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xd fsbcount 0x2 flag 1 caller xfs_bmap_add_extent_unwritten_real+0x7ef/0x1260
     kworker/3:1-71      [003] .....  8630.385654: xfs_bmap_post_update: dev 7:0 ino 0x83 state LF|RF cur 00000000a34e18c4/0 fileoff 0x0 startblock 0xd fsbcount 0x2 flag 0 caller xfs_bmap_add_extent_unwritten_real+0x7ef/0x1260
          xfs_io-52772   [001] .....  8630.385713: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x2000 pos 0x2000 bytecount 0x3e8
          xfs_io-52772   [001] .....  8630.385714: xfs_iext_insert: dev 7:0 ino 0x83 state 0x40 cur 00000000c53a152a/1 fileoff 0x2 startblock 0xffffffffe0007 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_hole_delay.constprop.0+0x127/0x450
          xfs_io-52772   [001] .....  8630.385714: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x2000 pos 0x2000 bytecount 0x3e8 fork data startoff 0x2 startblock 0xffffffffe0007 fsbcount 0x1
          xfs_io-52772   [001] .....  8630.385717: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52772   [001] .....  8630.385720: xfs_bmap_pre_update: dev 7:0 ino 0x83 state LF|RF|0x40 cur 00000000c53a152a/1 fileoff 0x2 startblock 0xffffffffe0007 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_delay_real+0x83c/0x18c0
          xfs_io-52772   [001] .....  8630.385720: xfs_bmap_post_update: dev 7:0 ino 0x83 state LF|RF|0x40 cur 00000000c53a152a/1 fileoff 0x2 startblock 0xf fsbcount 0x1 flag 1 caller xfs_bmap_add_extent_delay_real+0x83c/0x18c0
          xfs_io-52772   [001] .....  8630.385725: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x2000 pos 0x2000 bytecount 0x1000 fork data startoff 0x2 startblock 0xffffffffe0007 fsbcount 0x1
          xfs_io-52772   [001] .....  8630.385725: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x2000 dirty len 0x1000 addr 0xf000 offset 0x2000 length 0x1000 type UNWRITTEN (0x3) flags  (0x0)
     kworker/3:1-71      [003] .....  8630.385730: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x23e8 disize 0x2000 pos 0x2000 bytecount 0x3e8
     kworker/3:1-71      [003] .....  8630.385730: xfs_iext_remove: dev 7:0 ino 0x83 state LC|LF|RF|0x40 cur 00000000c53a152a/1 fileoff 0x2 startblock 0xf fsbcount 0x1 flag 1 caller xfs_bmap_add_extent_unwritten_real+0x4af/0x1260
     kworker/3:1-71      [003] .....  8630.385730: xfs_bmap_pre_update: dev 7:0 ino 0x83 state LC|LF|RF|0x40 cur 00000000c53a152a/0 fileoff 0x0 startblock 0xd fsbcount 0x2 flag 0 caller xfs_bmap_add_extent_unwritten_real+0x4d3/0x1260
     kworker/3:1-71      [003] .....  8630.385730: xfs_bmap_post_update: dev 7:0 ino 0x83 state LC|LF|RF|0x40 cur 00000000c53a152a/0 fileoff 0x0 startblock 0xd fsbcount 0x3 flag 0 caller xfs_bmap_add_extent_unwritten_real+0x4d3/0x1260
          xfs_io-52772   [001] .....  8630.385764: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x23e8 pos 0x2328 bytecount 0x3e8
          xfs_io-52772   [001] .....  8630.385765: xfs_iomap_found: dev 7:0 ino 0x83 disize 0x23e8 pos 0x2328 bytecount 0x3e8 fork data startoff 0x0 startblock 0xd fsbcount 0x3
          xfs_io-52772   [001] .....  8630.385766: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52772   [001] .....  8630.385767: xfs_map_blocks_found: dev 7:0 ino 0x83 disize 0x23e8 pos 0x2000 bytecount 0x1000 fork data startoff 0x0 startblock 0xd fsbcount 0x3
          xfs_io-52772   [001] .....  8630.385767: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x2000 dirty len 0x1000 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
     kworker/2:1-72      [002] .....  8630.385776: xfs_setfilesize: dev 7:0 ino 0x83 isize 0x2710 disize 0x23e8 pos 0x2000 bytecount 0x710
```

## umount 뒤 xfs_db의 inode와 fsmap, umount 전 bmap

**관찰 출력**

```text
/mnt/bw18b/f2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..23]:         104..127          0 (104..127)          24 000000
core.size = 10000
core.nblocks = 3
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,13,3,0]
6: 0/13 len 3 owner 131 offset 0 bmbt 0 attrfork 0 extflag 0
```

## pwrite 0 8k의 lock과 transaction event

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-52947   [002] .....  8632.195501: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x13a/0x2c0
          xfs_io-52947   [002] .....  8632.195503: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_attr_map_shared+0x2a/0x50
          xfs_io-52947   [002] .....  8632.195504: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_attr_get+0xc9/0x100
          xfs_io-52947   [002] .....  8632.195505: xfs_update_time: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52947   [002] .....  8632.195508: xfs_trans_alloc: dev 7:0 trans 27fcd214 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-52947   [002] .....  8632.195508: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-52947   [002] .....  8632.195509: xfs_trans_commit: dev 7:0 trans 27fcd214 flags 0x1 caller xfs_trans_commit+0x53/0x80
          xfs_io-52947   [002] .....  8632.195512: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-52947   [002] .....  8632.195512: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000
          xfs_io-52947   [002] .....  8632.195514: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-52947   [002] .....  8632.195517: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-52947   [002] .....  8632.195521: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x279/0x2c0
          xfs_io-52947   [002] .....  8632.195521: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x13a/0x2c0
          xfs_io-52947   [002] .....  8632.195521: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000
          xfs_io-52947   [002] .....  8632.195521: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-52947   [002] .....  8632.195522: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-52947   [002] .....  8632.195523: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x279/0x2c0
```

## 첫 fsync의 lock과 transaction event

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-52947   [002] .....  8632.195530: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52947   [002] .....  8632.195533: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_writeback_range+0xef/0x490
          xfs_io-52947   [002] .....  8632.195533: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_writeback_range+0x252/0x490
          xfs_io-52947   [002] .....  8632.195534: xfs_trans_alloc: dev 7:0 trans 4ddc7c08 flags 0x14 caller xfs_bmapi_convert_one_delalloc+0x9d/0x450
          xfs_io-52947   [002] .....  8632.195534: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_bmapi_convert_one_delalloc+0xb5/0x450
          xfs_io-52947   [002] .....  8632.195594: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-52947   [002] .....  8632.195595: xfs_trans_roll: dev 7:0 trans 4ddc7c08 flags 0x15 caller xfs_defer_trans_roll+0x77/0x130
          xfs_io-52947   [002] .....  8632.195595: xfs_trans_commit: dev 7:0 trans 4ddc7c08 flags 0x35 caller xfs_trans_roll+0x38/0xe0
          xfs_io-52947   [002] .....  8632.195599: xfs_defer_pending_finish: dev 7:0 optype rmap intent 00000000b5739756 flags  committed 0 nr 1
          xfs_io-52947   [002] .....  8632.195611: xfs_trans_commit: dev 7:0 trans 4ddc7c09 flags 0x95 caller xfs_trans_commit+0x53/0x80
          xfs_io-52947   [002] .....  8632.195611: xfs_cil_whiteout_mark: dev 7:0 lip 00000000b5739756 lsn 0/0 type XFS_LI_RUI flags WHITEOUT
          xfs_io-52947   [002] .....  8632.195612: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_bmapi_convert_one_delalloc+0x3b2/0x450
          xfs_io-52947   [002] .....  8632.195612: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x2
     kworker/1:1-70      [001] .....  8632.195623: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x2000 disize 0x0 pos 0x0 bytecount 0x2000
     kworker/1:1-70      [001] .....  8632.195624: xfs_trans_alloc: dev 7:0 trans 20a556f5 flags 0x14 caller xfs_trans_alloc_inode+0x8c/0x190
     kworker/1:1-70      [001] .....  8632.195625: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
     kworker/1:1-70      [001] .....  8632.195628: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/1:1-70      [001] .....  8632.195631: xfs_trans_roll: dev 7:0 trans 20a556f5 flags 0x15 caller xfs_defer_trans_roll+0x77/0x130
     kworker/1:1-70      [001] .....  8632.195631: xfs_trans_commit: dev 7:0 trans 20a556f5 flags 0x35 caller xfs_trans_roll+0x38/0xe0
     kworker/1:1-70      [001] .....  8632.195633: xfs_defer_pending_finish: dev 7:0 optype rmap intent 0000000073dbf125 flags  committed 0 nr 1
     kworker/1:1-70      [001] .....  8632.195639: xfs_trans_commit: dev 7:0 trans 20a556f6 flags 0x95 caller xfs_trans_commit+0x53/0x80
     kworker/1:1-70      [001] .....  8632.195640: xfs_cil_whiteout_mark: dev 7:0 lip 0000000073dbf125 lsn 0/0 type XFS_LI_RUI flags WHITEOUT
     kworker/1:1-70      [001] .....  8632.195641: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_iomap_write_unwritten+0xe1/0x310
```

## 세 번째 write, fsync, close의 lock과 transaction event

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-52947   [001] .....  8632.195729: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x13a/0x2c0
          xfs_io-52947   [001] .....  8632.195729: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x23e8 pos 0x2328 bytecount 0x3e8
          xfs_io-52947   [001] .....  8632.195730: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-52947   [001] .....  8632.195730: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_buffered_write_iomap_begin+0x55b/0xe00
          xfs_io-52947   [001] .....  8632.195731: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_buffered_write+0x279/0x2c0
          xfs_io-52947   [001] .....  8632.195736: xfs_file_fsync: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-52947   [001] .....  8632.195738: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_writeback_range+0xef/0x490
          xfs_io-52947   [001] .....  8632.195738: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_writeback_range+0x252/0x490
          xfs_io-52947   [001] .....  8632.195738: xfs_map_blocks_found: dev 7:0 ino 0x83 disize 0x23e8 pos 0x2000 bytecount 0x1000 fork data startoff 0x0 startblock 0xd fsbcount 0x3
     kworker/2:1-72      [002] .....  8632.195746: xfs_trans_alloc: dev 7:0 trans 8662068d flags 0x0 caller xfs_setfilesize+0x4e/0x160
     kworker/2:1-72      [002] .....  8632.195746: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_setfilesize+0x87/0x160
     kworker/2:1-72      [002] .....  8632.195746: xfs_setfilesize: dev 7:0 ino 0x83 isize 0x2710 disize 0x23e8 pos 0x2000 bytecount 0x710
     kworker/2:1-72      [002] .....  8632.195746: xfs_trans_commit: dev 7:0 trans 8662068d flags 0x1 caller xfs_trans_commit+0x53/0x80
     kworker/2:1-72      [002] .....  8632.195752: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xlog_cil_commit+0x570/0xb50
          xfs_io-52947   [001] .....  8632.195824: xfs_ilock_nowait: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_release+0xfe/0x160
          xfs_io-52947   [001] .....  8632.195824: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_can_free_eofblocks+0x131/0x1e0
          xfs_io-52947   [001] .....  8632.195824: xfs_iunlock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_can_free_eofblocks+0x151/0x1e0
          xfs_io-52947   [001] .....  8632.195825: xfs_iunlock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_release+0x11b/0x160
```

## log에 남은 것: tracepoint와 xfs_logprint

**관찰 출력**

```text
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
#              | |         |   |||||     |         |
          xfs_io-53067   [005] .....  8634.188029: xfs_trans_alloc: dev 7:0 trans ec6b7150 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-53067   [005] .....  8634.188055: xfs_trans_alloc: dev 7:0 trans fa165016 flags 0x14 caller xfs_bmapi_convert_one_delalloc+0x9d/0x450
          xfs_io-53067   [005] .....  8634.188102: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53067   [005] .....  8634.188117: xfs_cil_whiteout_mark: dev 7:0 lip 000000008bbda4f8 lsn 0/0 type XFS_LI_RUI flags WHITEOUT
     kworker/6:0-24348   [006] .....  8634.188127: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x2000 disize 0x0 pos 0x0 bytecount 0x2000
     kworker/6:0-24348   [006] .....  8634.188128: xfs_trans_alloc: dev 7:0 trans 390ab255 flags 0x14 caller xfs_trans_alloc_inode+0x8c/0x190
     kworker/6:0-24348   [006] .....  8634.188131: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/6:0-24348   [006] .....  8634.188139: xfs_cil_whiteout_mark: dev 7:0 lip 000000006c97213b lsn 0/0 type XFS_LI_RUI flags WHITEOUT
          xfs_io-53067   [005] .....  8634.188143: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_file_fsync+0x206/0x250
          xfs_io-53067   [005] .....  8634.188143: xfs_log_force: dev 7:0 lsn 0x1 caller xfs_log_force_seq+0x73/0x160
== xfs_logprint -t
    log tail: 2 head: 7 state: <DIRTY>
LOG REC AT LSN cycle 1 block 2 (0x1, 0x2)
TRANS: tid:0xab0c7fae  #items:21  trans:0xab0c7fae  q:0x556574316150
BUF: cnt:2 total:2 a:0x5565743161a0 len:24 a:0x556574316410 len:128 
	BUF:  #regs:2   start blkno:0x2   len:1   bmap size:1   flags:0x3800
	AGI Buffer: (XAGI)
BUF: cnt:2 total:2 a:0x5565743164a0 len:24 a:0x556574316520 len:128 
	BUF:  #regs:2   start blkno:0x20   len:8   bmap size:1   flags:0x2000
	BUF DATA
BUF: cnt:2 total:2 a:0x5565743165b0 len:24 a:0x556574316a40 len:128 
	BUF:  #regs:2   start blkno:0x18   len:8   bmap size:1   flags:0x2000
	BUF DATA
INO: cnt:3 total:3 a:0x556574316ad0 len:56 a:0x55657431ec10 len:176 a:0x556574316b80 len:16 
	INODE: #regs:3   ino:0x80  flags:0x3   dsize:16
	CORE inode:
		DATA FORK LOCAL inode data:
BUF: cnt:2 total:2 a:0x556574316ba0 len:24 a:0x55657431ed00 len:128 
	BUF:  #regs:2   start blkno:0x1   len:1   bmap size:1   flags:0x2800
	AGF Buffer: (XAGF)
BUF: cnt:2 total:2 a:0x55657431ed90 len:24 a:0x55657431ee10 len:128 
	BUF:  #regs:2   start blkno:0x10   len:8   bmap size:1   flags:0x2000
	BUF DATA
BUF: cnt:2 total:2 a:0x55657431eea0 len:24 a:0x55657431ef20 len:128 
	BUF:  #regs:2   start blkno:0x8   len:8   bmap size:1   flags:0x2000
	BUF DATA
INO: cnt:4 total:4 a:0x55657431efb0 len:56 a:0x55657431f070 len:176 a:0x55657431f130 len:16 a:0x55657431f150 len:24 
	INODE: #regs:4   ino:0x83  flags:0x45   dsize:16
	CORE inode:
		DATA FORK EXTENTS inode data:
		ATTR FORK LOCAL inode data:
BUF: cnt:2 total:2 a:0x55657431f170 len:24 a:0x55657431f1f0 len:256 
	BUF:  #regs:2   start blkno:0x28   len:8   bmap size:1   flags:0x2000
	BUF DATA
== xfs_logprint -t -i
	INODE: #regs:4   ino:0x83  flags:0x45   dsize:16
	CORE inode:
		magic:IN  mode:0x81b4  ver:3  format:2
		uid:1000  gid:1000  nlink:1 projid:0x00000000
		atime:-860362734  mtime:-840362734  ctime:-840362734
		flushiter:0
		size:0x2000  nblks:0x2  exsize:0  nextents:1  anextents:0
		forkoff:24  dmevmask:0x0  dmstate:0  flags:0x0  gen:3634879228
		flags2 0x18 cowextsize 0x0
		DATA FORK EXTENTS inode data:
		ATTR FORK LOCAL inode data:
```

## function_graph 설정 단계의 출력

**관찰 출력**

```text
current_tracer=nop
7.1.0-rc5-g8fde5d1d47f6
xfs_file_buffered_write
xfs_end_io
xfs_vm_writepages
xfs_file_buffered_write
12
function_graph
funcgraph-proc
wrote 8192/8192 bytes at offset 0
8 KiB, 2 ops; 0.0000 sec (122 MiB/sec and 31250.0000 ops/sec)
```

## 첫 write(2)의 function_graph

**관찰 출력**

```text
# tracer: function_graph
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
# |     |    |           |   |                     |   |   |   |
 3)   xfs_io-950   |               |  xfs_file_buffered_write() {
 3)   xfs_io-950   |               |    xfs_ilock() {
 3)   xfs_io-950   |   0.229 us    |      down_write();
 3)   xfs_io-950   |   0.701 us    |    }
 3)   xfs_io-950   |               |    xfs_file_write_checks() {
 3)   xfs_io-950   |               |      generic_write_checks() {
 3)   xfs_io-950   |   0.081 us    |        generic_write_check_limits();
 3)   xfs_io-950   |   0.240 us    |      }
 3)   xfs_io-950   |               |      xfs_break_layouts() {
 3)   xfs_io-950   |   0.049 us    |        xfs_break_leased_layouts();
 3)   xfs_io-950   |   0.311 us    |      }
 3)   xfs_io-950   |               |      kiocb_modified() {
 3)   xfs_io-950   |               |        file_update_time_flags() {
 3)   xfs_io-950   |               |          xfs_vn_update_time() {
 3)   xfs_io-950   | + 21.212 us   |          }
 3)   xfs_io-950   | + 21.984 us   |        }
 3)   xfs_io-950   | + 25.919 us   |      }
 3)   xfs_io-950   | + 26.881 us   |    }
 3)   xfs_io-950   |               |    iomap_file_buffered_write() {
 3)   xfs_io-950   |               |      iomap_iter() {
 3)   xfs_io-950   |               |        xfs_buffered_write_iomap_begin() {
 3)   xfs_io-950   |               |          xfs_get_extsz_hint() {
 3)   xfs_io-950   |   0.069 us    |            xfs_is_always_cow_inode();
 3)   xfs_io-950   |   0.200 us    |          }
 3)   xfs_io-950   |               |          xfs_qm_dqattach() {
 3)   xfs_io-950   |   0.060 us    |            xfs_qm_need_dqattach();
 3)   xfs_io-950   |   0.371 us    |          }
 3)   xfs_io-950   |               |          xfs_ilock_for_iomap() {
 3)   xfs_io-950   |               |            xfs_ilock() {
 3)   xfs_io-950   |   0.060 us    |              down_write();
 3)   xfs_io-950   |   0.170 us    |            }
 3)   xfs_io-950   |   0.290 us    |          }
 3)   xfs_io-950   |   0.071 us    |          xfs_iread_extents();
 3)   xfs_io-950   |   0.070 us    |          xfs_iext_lookup_extent();
 3)   xfs_io-950   |   0.060 us    |          xfs_is_always_cow_inode();
 3)   xfs_io-950   |   0.060 us    |          xfs_is_always_cow_inode();
 3)   xfs_io-950   |   0.060 us    |          xfs_is_always_cow_inode();
 3)   xfs_io-950   |   0.250 us    |          xfs_iomap_prealloc_size.isra.0();
 3)   xfs_io-950   |               |          xfs_bmapi_reserve_delalloc() {
 3)   xfs_io-950   |               |            xfs_quota_reserve_blkres() {
 3)   xfs_io-950   |   0.060 us    |              xfs_trans_reserve_quota_nblks();
 3)   xfs_io-950   |   0.300 us    |            }
 3)   xfs_io-950   |   0.080 us    |            xfs_bmap_worst_indlen();
 3)   xfs_io-950   |   0.080 us    |            xfs_dec_freecounter();
 3)   xfs_io-950   |   0.190 us    |            xfs_mod_delalloc();
 3)   xfs_io-950   |               |            xfs_bmap_add_extent_hole_delay.constprop.0() {
 3)   xfs_io-950   |               |              xfs_iext_prev() {
 3)   xfs_io-950   |   0.070 us    |                xfs_iext_last();
 3)   xfs_io-950   |   0.241 us    |              }
 3)   xfs_io-950   |   0.060 us    |              xfs_iext_get_extent();
 3)   xfs_io-950   |   0.060 us    |              xfs_iext_get_extent();
 3)   xfs_io-950   |               |              xfs_iext_insert() {
 3)   xfs_io-950   |   0.060 us    |                xfs_iext_state_to_fork();
 3)   xfs_io-950   |               |                xfs_iext_insert_raw() {
 3)   xfs_io-950   |   0.060 us    |                  __kmalloc_cache_noprof();
 3)   xfs_io-950   |   0.431 us    |                }
 3)   xfs_io-950   |   0.792 us    |              }
 3)   xfs_io-950   |   1.662 us    |            }
 3)   xfs_io-950   |   3.124 us    |          }
 3)   xfs_io-950   |               |          xfs_iunlock() {
 3)   xfs_io-950   |   0.060 us    |            up_write();
 3)   xfs_io-950   |   0.191 us    |          }
 3)   xfs_io-950   |               |          xfs_bmbt_to_iomap() {
 3)   xfs_io-950   |   0.050 us    |            _raw_spin_lock();
 3)   xfs_io-950   |   0.060 us    |            _raw_spin_unlock();
 3)   xfs_io-950   |   0.541 us    |          }
 3)   xfs_io-950   |   7.001 us    |        }
 3)   xfs_io-950   |   7.460 us    |      }
 3)   xfs_io-950   |               |      balance_dirty_pages_ratelimited_flags() {
 3)   xfs_io-950   |   0.060 us    |        inode_to_bdi();
 3)   xfs_io-950   |   0.290 us    |      }
 3)   xfs_io-950   |   0.080 us    |      fault_in_readable();
 3)   xfs_io-950   |               |      iomap_write_begin() {
 3)   xfs_io-950   |               |        iomap_get_folio() {
 3)   xfs_io-950   |   3.656 us    |        }
 3)   xfs_io-950   |   0.080 us    |        xfs_iomap_valid();
 3)   xfs_io-950   |   4.337 us    |      }
 3)   xfs_io-950   |               |      iomap_write_end() {
 3)   xfs_io-950   |   0.170 us    |        iomap_set_range_uptodate();
 3)   xfs_io-950   |               |        filemap_dirty_folio() {
 3)   xfs_io-950   |               |          __mark_inode_dirty() {
 3)   xfs_io-950   |               |            wb_wakeup_delayed() {
 3)   xfs_io-950   |   2.323 us    |            }
 3)   xfs_io-950   |   3.776 us    |          }
 3)   xfs_io-950   |   5.107 us    |        }
 3)   xfs_io-950   |   5.579 us    |      }
 3)   xfs_io-950   |               |      __iomap_put_folio.isra.0() {
 3)   xfs_io-950   |   0.200 us    |        folio_unlock();
 3)   xfs_io-950   |   0.330 us    |      }
 3)   xfs_io-950   |   0.070 us    |      iomap_iter_advance();
 3)   xfs_io-950   |               |      iomap_iter() {
 3)   xfs_io-950   |   0.069 us    |        xfs_buffered_write_iomap_end();
 3)   xfs_io-950   |   0.240 us    |      }
 3)   xfs_io-950   | + 19.800 us   |    }
 3)   xfs_io-950   |               |    xfs_iunlock() {
 3)   xfs_io-950   |   0.060 us    |      up_write();
 3)   xfs_io-950   |   0.190 us    |    }
 3)   xfs_io-950   | + 49.784 us   |  }
```

## 두 번째 write(2)의 xfs_bmapi_reserve_delalloc function_graph

**관찰 출력**

```text
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
 3)   xfs_io-950   |               |          xfs_bmapi_reserve_delalloc() {
 3)   xfs_io-950   |               |            xfs_quota_reserve_blkres() {
 3)   xfs_io-950   |   0.060 us    |              xfs_trans_reserve_quota_nblks();
 3)   xfs_io-950   |   0.171 us    |            }
 3)   xfs_io-950   |   0.060 us    |            xfs_bmap_worst_indlen();
 3)   xfs_io-950   |   0.060 us    |            xfs_dec_freecounter();
 3)   xfs_io-950   |   0.060 us    |            xfs_mod_delalloc();
 3)   xfs_io-950   |               |            xfs_bmap_add_extent_hole_delay.constprop.0() {
 3)   xfs_io-950   |   0.090 us    |              xfs_iext_prev();
 3)   xfs_io-950   |   0.080 us    |              xfs_iext_get_extent();
 3)   xfs_io-950   |   0.070 us    |              xfs_iext_get_extent();
 3)   xfs_io-950   |   0.060 us    |              xfs_bmap_worst_indlen();
 3)   xfs_io-950   |   0.070 us    |              xfs_iext_prev();
 3)   xfs_io-950   |               |              xfs_iext_update_extent() {
 3)   xfs_io-950   |   0.060 us    |                xfs_iext_state_to_fork();
 3)   xfs_io-950   |   0.201 us    |              }
 3)   xfs_io-950   |   0.060 us    |              xfs_add_freecounter();
 3)   xfs_io-950   |   0.060 us    |              xfs_mod_delalloc();
 3)   xfs_io-950   |   1.262 us    |            }
 3)   xfs_io-950   |   1.953 us    |          }
```

## 첫 write(2)의 kiocb_modified function_graph

**관찰 출력**

```text
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
 3)   xfs_io-950   |               |      kiocb_modified() {
 3)   xfs_io-950   |               |        file_remove_privs_flags() {
 3)   xfs_io-950   |   0.070 us    |          setattr_should_drop_suidgid();
 3)   xfs_io-950   |               |          security_inode_need_killpriv() {
 3)   xfs_io-950   |               |            cap_inode_need_killpriv() {
 3)   xfs_io-950   |               |              __vfs_getxattr() {
 3)   xfs_io-950   |   0.300 us    |                xattr_resolve_name();
 3)   xfs_io-950   |               |                xfs_xattr_get() {
 3)   xfs_io-950   |   0.071 us    |                  xfs_ifork_zapped();
 3)   xfs_io-950   |               |                  xfs_attr_get() {
 3)   xfs_io-950   |   0.110 us    |                    xfs_da_hashname();
 3)   xfs_io-950   |               |                    xfs_ilock_attr_map_shared() {
 3)   xfs_io-950   |               |                      xfs_ilock() {
 3)   xfs_io-950   |   0.049 us    |                        down_read();
 3)   xfs_io-950   |   0.189 us    |                      }
 3)   xfs_io-950   |   0.310 us    |                    }
 3)   xfs_io-950   |               |                    xfs_attr_get_ilocked() {
 3)   xfs_io-950   |   0.070 us    |                      xfs_assert_ilocked();
 3)   xfs_io-950   |   0.069 us    |                      xfs_iread_extents();
 3)   xfs_io-950   |               |                      xfs_attr_shortform_getvalue() {
 3)   xfs_io-950   |   0.180 us    |                        xfs_attr_sf_findname();
 3)   xfs_io-950   |   0.430 us    |                      }
 3)   xfs_io-950   |   1.181 us    |                    }
 3)   xfs_io-950   |               |                    xfs_iunlock() {
 3)   xfs_io-950   |   0.060 us    |                      up_read();
 3)   xfs_io-950   |   0.180 us    |                    }
 3)   xfs_io-950   |   2.123 us    |                  }
 3)   xfs_io-950   |   2.534 us    |                }
 3)   xfs_io-950   |   3.175 us    |              }
 3)   xfs_io-950   |   3.305 us    |            }
 3)   xfs_io-950   |   3.436 us    |          }
 3)   xfs_io-950   |   3.736 us    |        }
 3)   xfs_io-950   |               |        file_update_time_flags() {
 3)   xfs_io-950   |               |          xfs_vn_update_time() {
 3)   xfs_io-950   |               |            xfs_trans_alloc() {
 3)   xfs_io-950   |               |              xfs_log_reserve() {
 3)   xfs_io-950   |               |                xlog_ticket_alloc() {
 3)   xfs_io-950   |   4.287 us    |                }
 3)   xfs_io-950   |   0.200 us    |                xlog_grant_head_check();
 3)   xfs_io-950   |   4.998 us    |              }
 3)   xfs_io-950   | + 10.385 us   |            }
 3)   xfs_io-950   |               |            xfs_ilock() {
 3)   xfs_io-950   |   0.061 us    |              down_write();
 3)   xfs_io-950   |   0.180 us    |            }
 3)   xfs_io-950   |               |            inode_set_ctime_current() {
 3)   xfs_io-950   |   0.320 us    |            }
 3)   xfs_io-950   |               |            xfs_trans_ijoin() {
 3)   xfs_io-950   |   0.472 us    |            }
 3)   xfs_io-950   |               |            xfs_trans_log_inode() {
 3)   xfs_io-950   |   0.351 us    |            }
 3)   xfs_io-950   |               |            xfs_trans_commit() {
 3)   xfs_io-950   |               |              __xfs_trans_commit() {
 3)   xfs_io-950   |               |                xlog_cil_commit() {
 3)   xfs_io-950   |               |                  xfs_inode_item_format() {
 3)   xfs_io-950   |   2.094 us    |                  }
 3)   xfs_io-950   |               |                  xfs_log_ticket_ungrant() {
 3)   xfs_io-950   |   0.541 us    |                  }
 3)   xfs_io-950   |               |                  xfs_inode_item_committing() {
 3)   xfs_io-950   |               |                    xfs_inode_item_release() {
 3)   xfs_io-950   |               |                      xfs_iunlock() {
 3)   xfs_io-950   |   0.060 us    |                        up_write();
 3)   xfs_io-950   |   0.220 us    |                      }
 3)   xfs_io-950   |   0.471 us    |                    }
 3)   xfs_io-950   |   0.821 us    |                  }
 3)   xfs_io-950   |   5.868 us    |                }
 3)   xfs_io-950   |   8.463 us    |              }
 3)   xfs_io-950   |   8.723 us    |            }
 3)   xfs_io-950   | + 21.212 us   |          }
 3)   xfs_io-950   | + 21.984 us   |        }
 3)   xfs_io-950   | + 25.919 us   |      }
```

## 두 번째 write(2)의 kiocb_modified function_graph

**관찰 출력**

```text
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
 3)   xfs_io-950   |               |  xfs_file_buffered_write() {
 3)   xfs_io-950   |               |      kiocb_modified() {
 3)   xfs_io-950   |   0.069 us    |        file_remove_privs_flags();
 3)   xfs_io-950   |               |        file_update_time_flags() {
 3)   xfs_io-950   |               |          current_time() {
 3)   xfs_io-950   |   0.060 us    |            ktime_get_coarse_real_ts64_mg();
 3)   xfs_io-950   |   0.180 us    |          }
 3)   xfs_io-950   |   0.310 us    |        }
 3)   xfs_io-950   |   0.552 us    |      }
```

## fsync의 writeback function_graph

**관찰 출력**

```text
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
# |     |    |           |   |                     |   |   |   |
 3)   xfs_io-950   |               |  xfs_vm_writepages() {
 3)   xfs_io-950   |               |    iomap_writepages() {
 3)   xfs_io-950   |               |      writeback_iter() {
 3)   xfs_io-950   |   2.393 us    |      }
 3)   xfs_io-950   |               |      iomap_writeback_folio() {
 3)   xfs_io-950   |               |        __folio_start_writeback() {
 3)   xfs_io-950   |   1.723 us    |        }
 3)   xfs_io-950   |               |        xfs_writeback_range() {
 3)   xfs_io-950   |   0.060 us    |          xfs_imap_valid();
 3)   xfs_io-950   |               |          xfs_ilock() {
 3)   xfs_io-950   |   0.060 us    |            down_read();
 3)   xfs_io-950   |   0.180 us    |          }
 3)   xfs_io-950   |   0.060 us    |          xfs_imap_valid();
 3)   xfs_io-950   |   0.080 us    |          xfs_iext_lookup_extent();
 3)   xfs_io-950   |               |          xfs_iunlock() {
 3)   xfs_io-950   |   0.060 us    |            up_read();
 3)   xfs_io-950   |   0.189 us    |          }
 3)   xfs_io-950   |               |          xfs_bmapi_convert_delalloc() {
 3)   xfs_io-950   |               |            xfs_bmapi_convert_one_delalloc() {
 3)   xfs_io-950   |               |              xfs_trans_alloc() {
 3)   xfs_io-950   |               |                xfs_log_reserve() {
 3)   xfs_io-950   |   0.670 us    |                }
 3)   xfs_io-950   |   0.971 us    |              }
 3)   xfs_io-950   |               |              xfs_ilock() {
 3)   xfs_io-950   |   0.061 us    |                down_write();
 3)   xfs_io-950   |   0.171 us    |              }
 3)   xfs_io-950   |               |              xfs_trans_ijoin() {
 3)   xfs_io-950   |   0.281 us    |              }
 3)   xfs_io-950   |               |              xfs_bmapi_allocate() {
 3)   xfs_io-950   |               |                xfs_bmap_btalloc() {
 3)   xfs_io-950   |   0.080 us    |                  xfs_bmap_adjacent();
 3)   xfs_io-950   |               |                  xfs_bmap_longest_free_extent() {
 3)   xfs_io-950   |   0.911 us    |                  }
 3)   xfs_io-950   |               |                  xfs_alloc_vextent_start_ag() {
 3)   xfs_io-950   |               |                    xfs_alloc_vextent_iterate_ags.constprop.0() {
 3)   xfs_io-950   |               |                      xfs_alloc_vextent_prepare_ag() {
 3)   xfs_io-950   | + 40.171 us   |                        xfs_alloc_fix_freelist();
 3)   xfs_io-950   | + 40.431 us   |                      }
 3)   xfs_io-950   |               |                      xfs_alloc_ag_vextent_near() {
 3)   xfs_io-950   | + 34.422 us   |                      }
 3)   xfs_io-950   | + 75.764 us   |                    }
 3)   xfs_io-950   |               |                    xfs_alloc_vextent_finish() {
 3)   xfs_io-950   |               |                      xfs_alloc_update_counters() {
 3)   xfs_io-950   |   0.140 us    |                        xfs_alloc_log_agf();
 3)   xfs_io-950   |   0.271 us    |                      }
 3)   xfs_io-950   |   1.262 us    |                    }
 3)   xfs_io-950   | + 77.776 us   |                  }
 3)   xfs_io-950   |               |                  xfs_bmap_alloc_account() {
 3)   xfs_io-950   |   0.710 us    |                  }
 3)   xfs_io-950   | + 81.663 us   |                }
 3)   xfs_io-950   |               |                xfs_bmap_add_extent_delay_real() {
 3)   xfs_io-950   |               |                  xfs_rmap_map_extent() {
 3)   xfs_io-950   |   0.060 us    |                    xfs_ifork_is_realtime();
 3)   xfs_io-950   |               |                    __xfs_rmap_add() {
 3)   xfs_io-950   |   0.080 us    |                      kmem_cache_alloc_noprof();
 3)   xfs_io-950   |               |                      xfs_rmap_defer_add() {
 3)   xfs_io-950   |   0.310 us    |                        xfs_group_intent_get();
 3)   xfs_io-950   |   0.180 us    |                        xfs_defer_add();
 3)   xfs_io-950   |   1.142 us    |                      }
 3)   xfs_io-950   |   1.502 us    |                    }
 3)   xfs_io-950   |   1.752 us    |                  }
 3)   xfs_io-950   |   3.725 us    |                }
 3)   xfs_io-950   | + 85.849 us   |              }
 3)   xfs_io-950   |               |              xfs_trans_commit() {
 3)   xfs_io-950   |               |                xfs_defer_finish_noroll() {
 3)   xfs_io-950   |               |                  xfs_defer_create_intents() {
 3)   xfs_io-950   |   1.453 us    |                  }
 3)   xfs_io-950   |               |                  xfs_defer_trans_roll() {
 3)   xfs_io-950   |   8.072 us    |                  }
 3)   xfs_io-950   |               |                  xfs_defer_finish_one() {
 3)   xfs_io-950   |               |                    xfs_rmap_update_finish_item() {
 3)   xfs_io-950   | + 17.717 us   |                    }
 3)   xfs_io-950   | + 21.552 us   |                  }
 3)   xfs_io-950   | + 32.098 us   |                }
 3)   xfs_io-950   |               |                __xfs_trans_commit() {
 3)   xfs_io-950   |               |                  xlog_cil_commit() {
 3)   xfs_io-950   |   7.803 us    |                  }
 3)   xfs_io-950   |   9.194 us    |                }
 3)   xfs_io-950   | + 41.612 us   |              }
 3)   xfs_io-950   |               |              xfs_iunlock() {
 3)   xfs_io-950   |   0.051 us    |                up_write();
 3)   xfs_io-950   |   0.180 us    |              }
 3)   xfs_io-950   | ! 131.737 us  |            }
 3)   xfs_io-950   | ! 131.998 us  |          }
 3)   xfs_io-950   |               |          iomap_add_to_ioend() {
 3)   xfs_io-950   |   6.380 us    |          }
 3)   xfs_io-950   | ! 139.810 us  |        }
 3)   xfs_io-950   | ! 142.253 us  |      }
 3)   xfs_io-950   |   0.061 us    |      folio_unlock();
 3)   xfs_io-950   |               |      writeback_iter() {
 3)   xfs_io-950   |   1.061 us    |      }
 3)   xfs_io-950   |               |      iomap_writeback_folio() {
 3)   xfs_io-950   |               |        __folio_start_writeback() {
 3)   xfs_io-950   |   0.842 us    |        }
 3)   xfs_io-950   |               |        xfs_writeback_range() {
 3)   xfs_io-950   |   0.071 us    |          xfs_imap_valid();
 3)   xfs_io-950   |               |          iomap_add_to_ioend() {
 3)   xfs_io-950   |   0.571 us    |          }
 3)   xfs_io-950   |   0.842 us    |        }
 3)   xfs_io-950   |   1.873 us    |      }
 3)   xfs_io-950   |   0.060 us    |      folio_unlock();
 3)   xfs_io-950   |               |      writeback_iter() {
 3)   xfs_io-950   | + 24.738 us   |      }
 3)   xfs_io-950   |               |      xfs_writeback_submit() {
 3)   xfs_io-950   |               |        iomap_ioend_writeback_submit() {
 3)   xfs_io-950   |               |          submit_bio() {
 3)   xfs_io-950   |   4.927 us    |          }
 3)   xfs_io-950   |   5.058 us    |        }
 3)   xfs_io-950   |   5.357 us    |      }
 3)   xfs_io-950   | ! 178.608 us  |    }
 3)   xfs_io-950   | ! 179.269 us  |  }
```

## bio 완료의 function_graph

**관찰 출력**

```text
# CPU  TASK/PID         DURATION                  FUNCTION CALLS
# |     |    |           |   |                     |   |   |   |
 0)  kworker-150   |               |  xfs_end_io() {
 0)  kworker-150   |   0.090 us    |    iomap_sort_ioends();
 0)  kworker-150   |   0.071 us    |    iomap_ioend_try_merge();
 0)  kworker-150   |               |    xfs_end_ioend_write() {
 0)  kworker-150   |   0.160 us    |      blk_status_to_errno();
 0)  kworker-150   |               |      xfs_iomap_write_unwritten() {
 0)  kworker-150   |               |        xfs_trans_alloc_inode() {
 0)  kworker-150   |               |          xfs_trans_alloc() {
 0)  kworker-150   |               |            xfs_log_reserve() {
 0)  kworker-150   |   1.331 us    |            }
 0)  kworker-150   |   7.621 us    |          }
 0)  kworker-150   |               |          xfs_ilock() {
 0)  kworker-150   |   0.070 us    |            down_write();
 0)  kworker-150   |   0.180 us    |          }
 0)  kworker-150   |               |          xfs_trans_ijoin() {
 0)  kworker-150   |   0.341 us    |          }
 0)  kworker-150   |   8.973 us    |        }
 0)  kworker-150   |               |        xfs_bmapi_write() {
 0)  kworker-150   |               |          xfs_bmapi_convert_unwritten() {
 0)  kworker-150   |               |            xfs_bmap_add_extent_unwritten_real() {
 0)  kworker-150   |               |              xfs_iext_update_extent() {
 0)  kworker-150   |   0.221 us    |              }
 0)  kworker-150   |               |              xfs_rmap_convert_extent() {
 0)  kworker-150   |   1.653 us    |              }
 0)  kworker-150   |   3.285 us    |            }
 0)  kworker-150   |   3.796 us    |          }
 0)  kworker-150   |   6.250 us    |        }
 0)  kworker-150   |               |        xfs_trans_log_inode() {
 0)  kworker-150   |   0.191 us    |        }
 0)  kworker-150   |               |        xfs_trans_commit() {
 0)  kworker-150   |               |          xfs_defer_finish_noroll() {
 0)  kworker-150   |               |            xfs_defer_create_intents() {
 0)  kworker-150   |   3.605 us    |            }
 0)  kworker-150   |               |            xfs_defer_trans_roll() {
 0)  kworker-150   | + 10.075 us   |            }
 0)  kworker-150   |               |            xfs_defer_finish_one() {
 0)  kworker-150   |               |              xfs_rmap_update_finish_item() {
 0)  kworker-150   | + 14.551 us   |              }
 0)  kworker-150   | + 19.961 us   |            }
 0)  kworker-150   | + 34.392 us   |          }
 0)  kworker-150   |               |          __xfs_trans_commit() {
 0)  kworker-150   |               |            xlog_cil_commit() {
 0)  kworker-150   |   9.244 us    |            }
 0)  kworker-150   | + 19.078 us   |          }
 0)  kworker-150   | + 53.700 us   |        }
 0)  kworker-150   |               |        xfs_iunlock() {
 0)  kworker-150   |   0.060 us    |          up_write();
 0)  kworker-150   |   0.210 us    |        }
 0)  kworker-150   | + 70.516 us   |      }
 0)  kworker-150   |               |      iomap_finish_ioends() {
 0)  kworker-150   |               |        iomap_finish_ioend() {
 0)  kworker-150   |               |          iomap_finish_ioend_buffered_write() {
 0)  kworker-150   |               |            iomap_finish_folio_write() {
 0)  kworker-150   |               |              folio_end_writeback() {
 0)  kworker-150   |   4.357 us    |              }
 0)  kworker-150   |   4.497 us    |            }
 0)  kworker-150   |               |            iomap_finish_folio_write() {
 0)  kworker-150   |               |              folio_end_writeback() {
 0)  kworker-150   |   2.915 us    |              }
 0)  kworker-150   |   3.035 us    |            }
 0)  kworker-150   | + 10.065 us   |          }
 0)  kworker-150   | + 10.325 us   |        }
 0)  kworker-150   | + 10.676 us   |      }
 0)  kworker-150   | + 81.903 us   |    }
 0)  kworker-150   | + 83.585 us   |  }
```

## fsync 없이 45초 기다린 writeback event와 bmap

**관찰 출력**

```text
dirty_expire_centisecs=3000 dirty_writeback_centisecs=500
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-53522   [003] .....  8640.219578: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000
          xfs_io-53522   [003] .....  8640.219584: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
          xfs_io-53522   [003] .....  8640.219588: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000
          xfs_io-53522   [003] .....  8640.219588: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000 fork data startoff 0x1 startblock 0xffffffffe0007 fsbcount 0x1
   kworker/u32:0-12      [000] .....  8670.935600: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x2
   kworker/u32:0-12      [000] .....  8670.935602: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x0 dirty len 0x1000 addr 0xd000 offset 0x0 length 0x2000 type UNWRITTEN (0x3) flags  (0x0)
   kworker/u32:0-12      [000] .....  8670.935604: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x1000 dirty len 0x1000 addr 0xd000 offset 0x0 length 0x2000 type UNWRITTEN (0x3) flags  (0x0)
   kworker/u32:0-12      [000] .....  8670.935607: writeback_single_inode: bdi 7:0: ino=131 state=I_SYNC|I_SYNC_QUEUED dirtied_when=4303307268 age=45 index=0 to_write=1024 wrote=2 cgroup_ino=1
     kworker/6:0-24348   [006] .....  8670.935622: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x2000 disize 0x0 pos 0x0 bytecount 0x2000
/mnt/bw18g/f7:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..15]:         104..119          0 (104..119)          16 000000
```

## reflink file 덮어쓰기의 event와 bmap

**관찰 출력**

```text
== before CoW
/mnt/bw18e/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 100000
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-53247   [001] .....  8636.220477: xfs_file_buffered_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x1000
          xfs_io-53247   [001] .....  8636.220488: xfs_iext_insert: dev 7:0 ino 0x84 state COW cur 00000000a94a6957/0 fileoff 0x0 startblock 0xffffffffe0007 fsbcount 0x20 flag 0 caller xfs_bmap_add_extent_hole_delay.constprop.0+0x127/0x450
          xfs_io-53247   [001] .....  8636.220490: xfs_iomap_alloc: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x1000 fork cow startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x20
          xfs_io-53247   [001] .....  8636.220505: xfs_trans_alloc: dev 7:0 trans ec4034d6 flags 0x14 caller xfs_bmapi_convert_one_delalloc+0x9d/0x450
          xfs_io-53247   [001] .....  8636.220521: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53247   [001] .....  8636.220525: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53247   [001] .....  8636.220539: xfs_map_blocks_alloc: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x1000 fork cow startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x20
          xfs_io-53247   [001] .....  8636.220539: iomap_add_to_ioend: dev 7:0 ino 0x84 bdev 7:0 pos 0x1000 dirty len 0x1000 addr 0x28000 offset 0x0 length 0x20000 type UNWRITTEN (0x3) flags SHARED (0x4)
          xfs_io-53247   [001] .....  8636.220543: xfs_iext_insert: dev 7:0 ino 0x84 state COW cur 00000000432cf86e/1 fileoff 0x2 startblock 0x2a fsbcount 0x1e flag 1 caller xfs_bmap_add_extent_unwritten_real+0x6c3/0x1260
          xfs_io-53247   [001] .....  8636.220543: xfs_iext_insert: dev 7:0 ino 0x84 state COW cur 0000000010782546/1 fileoff 0x1 startblock 0x29 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_unwritten_real+0x6d8/0x1260
     kworker/0:1-11      [000] .....  8636.220573: xfs_reflink_end_cow: dev 7:0 ino 0x84 isize 0x10000 disize 0x10000 pos 0x1000 bytecount 0x1000
     kworker/0:1-11      [000] .....  8636.220591: xfs_trans_alloc: dev 7:0 trans b5c65b8 flags 0x14 caller xfs_reflink_end_cow_extent+0x52/0xe0
     kworker/0:1-11      [000] .....  8636.220594: xfs_reflink_cow_remap_from: dev 7:0 ino 0x84 fileoff 0x1 fsbcount 0x1 startblock 0x29 st 0
     kworker/0:1-11      [000] .....  8636.220595: xfs_reflink_cow_remap_to: dev 7:0 ino 0x84 fileoff 0x1 fsbcount 0x1 startblock 0x19 st 0
     kworker/0:1-11      [000] .....  8636.220598: xfs_defer_create_intent: dev 7:0 optype bmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220601: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220604: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220604: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220605: xfs_defer_create_intent: dev 7:0 optype bmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220611: xfs_iext_insert: dev 7:0 ino 0x84 state  cur 000000007c7c89c3/1 fileoff 0x2 startblock 0x1a fsbcount 0xe flag 0 caller xfs_bmap_del_extent_real+0x48a/0x8b0
     kworker/0:1-11      [000] .....  8636.220612: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
     kworker/0:1-11      [000] .....  8636.220630: xfs_iext_insert: dev 7:0 ino 0x84 state 0xc0 cur 0000000073eb5860/1 fileoff 0x1 startblock 0x29 fsbcount 0x1 flag 0 caller xfs_bmap_add_extent_hole_real+0x34d/0xd00
     kworker/0:1-11      [000] .....  8636.220631: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53247   [001] .....  8636.220751: xfs_trans_alloc: dev 7:0 trans 6c15c02f flags 0x4 caller xfs_free_eofblocks+0xe7/0x190
          xfs_io-53247   [001] .....  8636.220755: xfs_defer_create_intent: dev 7:0 optype rmap intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53247   [001] .....  8636.220755: xfs_defer_create_intent: dev 7:0 optype refcount intent 0000000000000000 flags  committed 0 nr 1
          xfs_io-53247   [001] .....  8636.220756: xfs_defer_create_intent: dev 7:0 optype extent_free intent 0000000000000000 flags  committed 0 nr 1
== after CoW
/mnt/bw18e/c1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          192..199          0 (192..199)           8 100000
   1: [8..15]:         200..207          0 (200..207)           8 000000
   2: [16..127]:       208..319          0 (208..319)         112 100000
/mnt/bw18e/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          192..199          0 (192..199)           8 100000
   1: [8..15]:         328..335          0 (328..335)           8 000000
   2: [16..127]:       208..319          0 (208..319)         112 100000
```

## debug kernel에서 본 COW fork와 data fork

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
/mnt/bw18i/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..255]:        delalloc                               256
/mnt/bw18i/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 100000
/mnt/bw18i/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          320..327          0 (320..327)           8
   1: [8..15]:         hole                                     8
   2: [16..255]:       336..575          0 (336..575)         240
/mnt/bw18i/c2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          192..199          0 (192..199)           8 100000
   1: [8..15]:         328..335          0 (328..335)           8 000000
   2: [16..127]:       208..319          0 (208..319)         112 100000
```

## extent size hint file의 bmap과 event

**관찰 출력**

```text
[65536] /mnt/bw18h/f8
/mnt/bw18h/f8:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 010000
 FLAG Values:
    0010000 Unwritten preallocated extent
/mnt/bw18h/f8:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..15]:         192..207          0 (192..207)          16 000000
   1: [16..127]:       208..319          0 (208..319)         112 010000
#           TASK-PID     CPU#  |||||  TIMESTAMP  FUNCTION
          xfs_io-53668   [005] .....  8686.285151: xfs_trans_alloc: dev 7:0 trans 5aa21976 flags 0x0 caller xfs_vn_update_time+0x87/0x1d0
          xfs_io-53668   [005] .....  8686.285156: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000
          xfs_io-53668   [005] .....  8686.285159: xfs_trans_alloc: dev 7:0 trans a8f9f473 flags 0x4 caller xfs_trans_alloc_inode+0x8c/0x190
          xfs_io-53668   [005] .....  8686.285226: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0x18 fsbcount 0x10
          xfs_io-53668   [005] .....  8686.285231: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000
          xfs_io-53668   [005] .....  8686.285231: xfs_iomap_found: dev 7:0 ino 0x83 disize 0x0 pos 0x1000 bytecount 0x1000 fork data startoff 0x1 startblock 0x19 fsbcount 0x1
          xfs_io-53668   [005] .....  8686.285261: xfs_map_blocks_found: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0x18 fsbcount 0x10
          xfs_io-53668   [005] .....  8686.285262: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x0 dirty len 0x1000 addr 0x18000 offset 0x0 length 0x10000 type UNWRITTEN (0x3) flags DIRTY (0x2)
          xfs_io-53668   [005] .....  8686.285263: iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x1000 dirty len 0x1000 addr 0x18000 offset 0x0 length 0x10000 type UNWRITTEN (0x3) flags DIRTY (0x2)
     kworker/1:1-70      [001] .....  8686.285273: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x2000 disize 0x0 pos 0x0 bytecount 0x2000
     kworker/1:1-70      [001] .....  8686.285275: xfs_trans_alloc: dev 7:0 trans 79b91900 flags 0x14 caller xfs_trans_alloc_inode+0x8c/0x190
```

