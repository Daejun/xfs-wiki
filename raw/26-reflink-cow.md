# reflink, COW, dedupe, exchange-range: 원본 출력

> 본문은 ../26-reflink-cow.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## reflink 전후의 refcountbt record와 inode의 REFLINK flag

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
agcount=1 crc=1 rmapbt=1 reflink=1 exchange=1 bsize=4096 bsize=4096 bsize=4096 
used blocks=13083
== xfs_db [before-cp]
numrecs = 0
-- inode file1 (131203)
v3.reflink = 0
v3.cowextsize = 0
== bmap file1
/mnt/p26r/file1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..1279]:       131264..132543    0 (131264..132543)  1280 100000
== bmap file2
/mnt/p26r/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..1279]:       131264..132543    0 (131264..132543)  1280 100000
used blocks=13083
== xfs_db [after-cp]
numrecs = 1
recs[1] = [startblock,blockcount,refcount,cowflag] 
1:[16408,160,2,0]
-- inode file1 (131203)
v3.reflink = 1
v3.cowextsize = 0
-- inode file2 (131204)
v3.reflink = 1
v3.cowextsize = 0
used blocks=13083
== xfs_db [after-3rd-reflink]
numrecs = 1
recs[1] = [startblock,blockcount,refcount,cowflag] 
1:[16408,160,3,0]
-- inode file1 (131203)
v3.reflink = 1
v3.cowextsize = 0
-- inode file2 (131204)
v3.reflink = 1
v3.cowextsize = 0
-- inode file3 (131205)
v3.reflink = 1
v3.cowextsize = 0
== bmap file3 after overwrite
/mnt/p26r/file3:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..1279]:       132544..133823    0 (132544..133823)  1280
used blocks=13243
== xfs_db [after-overwrite-file3]
numrecs = 1
recs[1] = [startblock,blockcount,refcount,cowflag] 
1:[16408,160,2,0]
-- inode file1 (131203)
v3.reflink = 1
v3.cowextsize = 0
-- inode file2 (131204)
v3.reflink = 1
v3.cowextsize = 0
-- inode file3 (131205)
v3.reflink = 1
v3.cowextsize = 0
used blocks=13083
== xfs_db [after-rm-file2-file3]
numrecs = 0
-- inode file1 (131203)
v3.reflink = 1
v3.cowextsize = 0
```

## 공유된 file2의 data fork와 COW fork 변화

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
agcount=1 crc=1 rmapbt=1 reflink=1 bsize=4096 bsize=4096 bsize=4096 
loop=/dev/loop0 dev=7:0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
file2 ino=131204
== bmap data fork file2 [shared]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..1279]:       131264..132543    0 (131264..132543)  1280 100000
== bmap cow fork file2 [shared]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..1279]:       hole                                  1280
events set=12
== bmap data fork file2 [after-write-blk0]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          132544..132551    0 (132544..132551)     8
   1: [8..1279]:       131272..132543    0 (131272..132543)  1272 100000
== bmap cow fork file2 [after-write-blk0]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..255]:        132552..132799    0 (132552..132799)   248
   2: [256..1279]:     hole                                  1024
== bmap data fork file2 [after-write-blk2]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          132544..132551    0 (132544..132551)     8
   1: [8..15]:         131272..131279    0 (131272..131279)     8 100000
   2: [16..23]:        132560..132567    0 (132560..132567)     8
   3: [24..1279]:      131288..132543    0 (131288..132543)  1256 100000
== bmap cow fork file2 [after-write-blk2]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..15]:         132552..132559    0 (132552..132559)     8
   2: [16..23]:        hole                                     8
   3: [24..255]:       132568..132799    0 (132568..132799)   232
   4: [256..1279]:     hole                                  1024
== bmap data fork file2 [after-fsync]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          132544..132551    0 (132544..132551)     8
   1: [8..15]:         131272..131279    0 (131272..131279)     8 100000
   2: [16..23]:        132560..132567    0 (132560..132567)     8
   3: [24..1279]:      131288..132543    0 (131288..132543)  1256 100000
== bmap cow fork file2 [after-fsync]
/mnt/p26c/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..7]:          hole                                     8
   1: [8..15]:         132552..132559    0 (132552..132559)     8
   2: [16..23]:        hole                                     8
   3: [24..255]:       132568..132799    0 (132568..132799)   232
   4: [256..1279]:     hole                                  1024
```

## write, fsync, freeze, umount 구간의 tracepoint

**관찰 출력**

```text
trace lines=51
      8 commit overrun: 0
      8 overrun: 0
== phase A
sh-970     [004] ...1.     8.912629: tracing_mark_write: p26-A
xfs_io-971     [000] .....     8.913648: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x4018 st 0
xfs_io-971     [000] .....     8.913659: xfs_iomap_alloc: dev 7:0 ino 0x20084 disize 0xa0000 pos 0x0 bytecount 0x1000 fork cow startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x20
xfs_io-972     [003] .....     8.914566: xfs_refcount_deferred: dev 7:0 op alloc_cow agno 0x0 gbno 0x40b8 fsbcount 0x20
kworker/0:1-11      [000] .....     8.914607: xfs_reflink_end_cow: dev 7:0 ino 0x20084 isize 0xa0000 disize 0xa0000 pos 0x0 bytecount 0x1000
kworker/0:1-11      [000] .....     8.914609: xfs_reflink_cow_remap_from: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x40b8 st 0
kworker/0:1-11      [000] .....     8.914609: xfs_reflink_cow_remap_to: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x4018 st 0
kworker/0:1-11      [000] .....     8.914626: xfs_bmap_deferred: dev 7:0 op unmap ino 0x20084 agno 0x0 gbno 0x4018 data fileoff 0x0 fsbcount 0x1 state 0
kworker/0:1-11      [000] .....     8.914636: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x4018 fsbcount 0x1
kworker/0:1-11      [000] .....     8.914641: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x40b8 fsbcount 0x1
kworker/0:1-11      [000] .....     8.914643: xfs_bmap_deferred: dev 7:0 op map ino 0x20084 agno 0x0 gbno 0x40b8 data fileoff 0x0 fsbcount 0x1 state 0
xfs_io-972     [003] .....     8.914659: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x40b8 st 0
xfs_io-972     [003] .....     8.914660: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x1 fsbcount 0x9f startblock 0x4019 st 0
sh-975     [000] ...1.     8.917770: tracing_mark_write: p26-B
== phase B
sh-975     [000] ...1.     8.917770: tracing_mark_write: p26-B
xfs_io-976     [006] .....     8.918724: xfs_reflink_cow_found: dev 7:0 ino 0x20084 fileoff 0x1 fsbcount 0x1f startblock 0x40b9 st 1
kworker/0:1-11      [000] .....     8.919636: xfs_reflink_end_cow: dev 7:0 ino 0x20084 isize 0xa0000 disize 0xa0000 pos 0x2000 bytecount 0x1000
kworker/0:1-11      [000] .....     8.919637: xfs_reflink_cow_remap_from: dev 7:0 ino 0x20084 fileoff 0x2 fsbcount 0x1 startblock 0x40ba st 0
kworker/0:1-11      [000] .....     8.919637: xfs_reflink_cow_remap_to: dev 7:0 ino 0x20084 fileoff 0x2 fsbcount 0x1 startblock 0x401a st 0
kworker/0:1-11      [000] .....     8.919642: xfs_bmap_deferred: dev 7:0 op unmap ino 0x20084 agno 0x0 gbno 0x401a data fileoff 0x2 fsbcount 0x1 state 0
kworker/0:1-11      [000] .....     8.919650: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x401a fsbcount 0x1
kworker/0:1-11      [000] .....     8.919655: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x40ba fsbcount 0x1
kworker/0:1-11      [000] .....     8.919657: xfs_bmap_deferred: dev 7:0 op map ino 0x20084 agno 0x0 gbno 0x40ba data fileoff 0x2 fsbcount 0x1 state 0
xfs_io-977     [003] .....     8.919667: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x40b8 st 0
xfs_io-977     [003] .....     8.919670: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x1 fsbcount 0x1 startblock 0x4019 st 0
xfs_io-977     [003] .....     8.919670: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x2 fsbcount 0x1 startblock 0x40ba st 0
xfs_io-977     [003] .....     8.919671: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x3 fsbcount 0x9d startblock 0x401b st 0
sh-980     [001] ...1.     8.922452: tracing_mark_write: p26-C
== phase C
sh-980     [001] ...1.     8.922452: tracing_mark_write: p26-C
xfs_io-982     [004] .....     8.924163: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x0 fsbcount 0x1 startblock 0x40b8 st 0
xfs_io-982     [004] .....     8.924167: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x1 fsbcount 0x1 startblock 0x4019 st 0
xfs_io-982     [004] .....     8.924168: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x2 fsbcount 0x1 startblock 0x40ba st 0
xfs_io-982     [004] .....     8.924168: xfs_reflink_trim_around_shared: dev 7:0 ino 0x20084 fileoff 0x3 fsbcount 0x9d startblock 0x401b st 0
sh-985     [006] ...1.     8.926835: tracing_mark_write: p26-D
== phase D
sh-985     [006] ...1.     8.926835: tracing_mark_write: p26-D
sh-999     [005] ...1.     8.961231: tracing_mark_write: p26-E
== phase E
sh-999     [005] ...1.     8.961231: tracing_mark_write: p26-E
kworker/4:1-103     [004] .....     8.963876: xfs_reflink_cancel_cow_range: dev 7:0 ino 0x20084 isize 0xa0000 disize 0xa0000 pos 0x0 bytecount 0xffffffffffffffff
kworker/4:1-103     [004] .....     8.963878: xfs_reflink_cancel_cow: dev 7:0 ino 0x20084 fileoff 0x3 fsbcount 0x1d startblock 0x40bb st 1
kworker/4:1-103     [004] .....     8.963902: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x40bb fsbcount 0x1d
kworker/4:1-103     [004] .....     8.963918: xfs_reflink_cancel_cow: dev 7:0 ino 0x20084 fileoff 0x1 fsbcount 0x1 startblock 0x40b9 st 1
kworker/4:1-103     [004] .....     8.963922: xfs_refcount_deferred: dev 7:0 op free_cow agno 0x0 gbno 0x40b9 fsbcount 0x1
sh-1003    [004] ...1.     8.966817: tracing_mark_write: p26-end
```

## freeze 중과 umount 뒤의 refcountbt

**관찰 출력**

```text
== xfs_db refcntbt [frozen-after-fsync]
numrecs = 4
recs[1-4] = [startblock,blockcount,refcount,cowflag] 
1:[16409,1,2,0] 
2:[16411,157,2,0] 
3:[16569,1,1,1] 
4:[16571,29,1,1]
== xfs_db refcntbt [after-umount]
numrecs = 2
recs[1-2] = [startblock,blockcount,refcount,cowflag] 
1:[16409,1,2,0] 
2:[16411,157,2,0]
```

## dedupe와 reflink 명령의 실행 결과

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
agcount=4 crc=1 rmapbt=1 reflink=1 bsize=4096 bsize=4096 bsize=4096 
loop=/dev/loop0 dev=7:0
== bmap file1
/mnt/p26d/file1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        192..327          0 (192..327)         136
== bmap file2
/mnt/p26d/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        328..463          0 (328..463)         136
times file2: mtime=2026-10-23 00:31:48.293163225 +0000 ctime=2026-10-23 00:31:48.293163225 +0000
== A dedupe file1 -> file2 whole
deduped 65573/65573 bytes at offset 0
64 KiB, 1 ops; 0.0001 sec (539.097 MiB/sec and 8620.6897 ops/sec)
rc=0
times file2: mtime=2026-10-23 00:31:48.293163225 +0000 ctime=2026-10-23 00:31:48.293163225 +0000
== bmap file1
/mnt/p26d/file1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        192..327          0 (192..327)         136 100000
== bmap file2
/mnt/p26d/file2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        192..327          0 (192..327)         136 100000
== B dedupe again
deduped 65573/65573 bytes at offset 0
64 KiB, 1 ops; 0.0001 sec (573.718 MiB/sec and 9174.3119 ops/sec)
rc=0
== C dedupe file1 -> file3 (differs)
XFS_IOC_FILE_EXTENT_SAME: Extents did not match.
rc=0
== bmap file3
/mnt/p26d/file3:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        736..871          0 (736..871)         136
== D dedupe into file4 opened O_RDONLY
times file4: mtime=2026-10-23 00:31:48.293865699 +0000 ctime=2026-10-23 00:31:48.293865699 +0000
deduped 65573/65573 bytes at offset 0
64 KiB, 1 ops; 0.0000 sec (1.566 GiB/sec and 25641.0256 ops/sec)
rc=0
times file4: mtime=2026-10-23 00:31:48.293865699 +0000 ctime=2026-10-23 00:31:48.293865699 +0000
== E reflink into file5 opened O_RDONLY
XFS_IOC_CLONE: Bad file descriptor
rc=0
== F reflink into file5 opened rw
times file5: mtime=2026-10-23 00:31:48.294635905 +0000 ctime=2026-10-23 00:31:48.294635905 +0000
linked 0/0 bytes at offset 0
0.000000 bytes, 1 ops; 0.0000 sec (0.000000 bytes/sec and 14492.7536 ops/sec)
rc=0
times file5: mtime=2026-10-23 00:31:50.576599450 +0000 ctime=2026-10-23 00:31:50.576599450 +0000
== bmap file5
/mnt/p26d/file5:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..135]:        192..327          0 (192..327)         136 100000
```

## dedupe 구간별 tracepoint

**관찰 출력**

```text
trace lines=47
== phase A (lines=9)
      2  xfs_bmap_deferred: 
      2  xfs_refcount_deferred: 
      1  xfs_reflink_remap_extent_dest: 
      1  xfs_reflink_remap_extent_src: 
      1  xfs_reflink_remap_range: 
      2  xfs_reflink_set_inode_flag: 
xfs_io-2572    [001] .....   147.886697: xfs_reflink_set_inode_flag: dev 7:0 ino 0x83 iflags 0x40
xfs_io-2572    [001] .....   147.886698: xfs_reflink_set_inode_flag: dev 7:0 ino 0x84 iflags 0x40
xfs_io-2572    [001] .....   147.886708: xfs_reflink_remap_range: dev 7:0 bytecount 0x10025 ino 0x83 isize 0x10025 disize 0x10025 pos 0x0 -> ino 0x84 isize 0x10025 disize 0x10025 pos 0x0
xfs_io-2572    [001] .....   147.886710: xfs_reflink_remap_extent_src: dev 7:0 ino 0x83 fileoff 0x0 fsbcount 0x11 startblock 0x18 st 0
xfs_io-2572    [001] .....   147.886711: xfs_reflink_remap_extent_dest: dev 7:0 ino 0x84 fileoff 0x0 fsbcount 0x11 startblock 0x29 st 0
xfs_io-2572    [001] .....   147.886720: xfs_bmap_deferred: dev 7:0 op unmap ino 0x84 agno 0x0 gbno 0x29 data fileoff 0x0 fsbcount 0x11 state 0
xfs_io-2572    [001] .....   147.886740: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x29 fsbcount 0x11
xfs_io-2572    [001] .....   147.886780: xfs_refcount_deferred: dev 7:0 op incr agno 0x0 gbno 0x18 fsbcount 0x11
xfs_io-2572    [001] .....   147.886792: xfs_bmap_deferred: dev 7:0 op map ino 0x84 agno 0x0 gbno 0x18 data fileoff 0x0 fsbcount 0x11 state 0
== phase B (lines=3)
      1  xfs_reflink_remap_extent_dest: 
      1  xfs_reflink_remap_extent_src: 
      1  xfs_reflink_remap_range: 
xfs_io-2578    [002] .....   147.892161: xfs_reflink_remap_range: dev 7:0 bytecount 0x10025 ino 0x83 isize 0x10025 disize 0x10025 pos 0x0 -> ino 0x84 isize 0x10025 disize 0x10025 pos 0x0
xfs_io-2578    [002] .....   147.892162: xfs_reflink_remap_extent_src: dev 7:0 ino 0x83 fileoff 0x0 fsbcount 0x11 startblock 0x18 st 0
xfs_io-2578    [002] .....   147.892163: xfs_reflink_remap_extent_dest: dev 7:0 ino 0x84 fileoff 0x0 fsbcount 0x11 startblock 0x18 st 0
== phase C (lines=0)
== phase D (lines=8)
      2  xfs_bmap_deferred: 
      2  xfs_refcount_deferred: 
      1  xfs_reflink_remap_extent_dest: 
      1  xfs_reflink_remap_extent_src: 
      1  xfs_reflink_remap_range: 
      1  xfs_reflink_set_inode_flag: 
xfs_io-2586    [003] .....   147.898959: xfs_reflink_set_inode_flag: dev 7:0 ino 0x85 iflags 0x40
xfs_io-2586    [003] .....   147.898964: xfs_reflink_remap_range: dev 7:0 bytecount 0x10025 ino 0x83 isize 0x10025 disize 0x10025 pos 0x0 -> ino 0x85 isize 0x10025 disize 0x10025 pos 0x0
xfs_io-2586    [003] .....   147.898964: xfs_reflink_remap_extent_src: dev 7:0 ino 0x83 fileoff 0x0 fsbcount 0x11 startblock 0x18 st 0
xfs_io-2586    [003] .....   147.898965: xfs_reflink_remap_extent_dest: dev 7:0 ino 0x85 fileoff 0x0 fsbcount 0x11 startblock 0x3a st 0
xfs_io-2586    [003] .....   147.898969: xfs_bmap_deferred: dev 7:0 op unmap ino 0x85 agno 0x0 gbno 0x3a data fileoff 0x0 fsbcount 0x11 state 0
xfs_io-2586    [003] .....   147.898977: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x3a fsbcount 0x11
xfs_io-2586    [003] .....   147.898978: xfs_refcount_deferred: dev 7:0 op incr agno 0x0 gbno 0x18 fsbcount 0x11
xfs_io-2586    [003] .....   147.898985: xfs_bmap_deferred: dev 7:0 op map ino 0x85 agno 0x0 gbno 0x18 data fileoff 0x0 fsbcount 0x11 state 0
== phase E (lines=0)
== phase F (lines=8)
      2  xfs_bmap_deferred: 
      2  xfs_refcount_deferred: 
      1  xfs_reflink_remap_extent_dest: 
      1  xfs_reflink_remap_extent_src: 
      1  xfs_reflink_remap_range: 
      1  xfs_reflink_set_inode_flag: 
xfs_io-2595    [001] .....   149.006691: xfs_reflink_set_inode_flag: dev 7:0 ino 0x86 iflags 0x40
xfs_io-2595    [001] .....   149.006698: xfs_reflink_remap_range: dev 7:0 bytecount 0x10025 ino 0x83 isize 0x10025 disize 0x10025 pos 0x0 -> ino 0x86 isize 0x10025 disize 0x10025 pos 0x0
xfs_io-2595    [001] .....   149.006699: xfs_reflink_remap_extent_src: dev 7:0 ino 0x83 fileoff 0x0 fsbcount 0x11 startblock 0x18 st 0
xfs_io-2595    [001] .....   149.006700: xfs_reflink_remap_extent_dest: dev 7:0 ino 0x86 fileoff 0x0 fsbcount 0x11 startblock 0x4b st 0
xfs_io-2595    [001] .....   149.006709: xfs_bmap_deferred: dev 7:0 op unmap ino 0x86 agno 0x0 gbno 0x4b data fileoff 0x0 fsbcount 0x11 state 0
xfs_io-2595    [001] .....   149.006727: xfs_refcount_deferred: dev 7:0 op decr agno 0x0 gbno 0x4b fsbcount 0x11
xfs_io-2595    [001] .....   149.006731: xfs_refcount_deferred: dev 7:0 op incr agno 0x0 gbno 0x18 fsbcount 0x11
xfs_io-2595    [001] .....   149.006742: xfs_bmap_deferred: dev 7:0 op map ino 0x86 agno 0x0 gbno 0x18 data fileoff 0x0 fsbcount 0x11 state 0
```

## xfs/792 순서와 중간 write가 낀 commit의 실행 결과

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
agcount=4 crc=1 rmapbt=1 reflink=1 exchange=1 bsize=4096 bsize=4096 parent=1 bsize=4096 
loop=/dev/loop0 dev=7:0
== A xfs/792 commit sequence without inject
/mnt/p26x/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
/mnt/p26x/a (fileupdate):
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        hole                                   128
   1: [128..255]:      4288..4415        0 (4288..4415)       128 000000
   2: [256..1535]:     hole                                  1280
   3: [1536..1663]:    4416..4543        0 (4416..4543)       128 000000
   4: [1664..2047]:    hole                                   384
Committed updates to '/mnt/p26x/a'.
commitupdate 1048576/1048576 bytes at offset 0
1 MiB, 1 ops; 0.0000 sec (15.501 GiB/sec and 15873.0159 ops/sec)
rc=0
/mnt/p26x/a:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 000000
   1: [128..255]:      4288..4415        0 (4288..4415)       128 000000
   2: [256..1535]:     448..1727         0 (448..1727)       1280 000000
   3: [1536..1663]:    4416..4543        0 (4416..4543)       128 000000
   4: [1664..2047]:    1856..2239        0 (1856..2239)       384 000000
== content a
0000000 58 58 58 58 58 58 58 58 58 58 58
*
0065536 59 59 59 59 59 59 59 59 59 59 59
*
0131072 58 58 58 58 58 58 58 58 58 58 58
*
0786432 57 57 57 57 57 57 57 57 57 57 57
*
0851968 58 58 58 58 58 58 58 58 58 58 58
*
1048576
== B commit with an intervening write to the original
other write rc=0
update xfs_io rc=1
xfs_io> xfs_io> xfs_io> xfs_io> committing update: Device or resource busy
== content b
0000000 58 58 58 58 58 58 58 58 58 58 58
*
0524288 41 41 41 41 41 41 41 41 41 41 41
*
0528384 58 58 58 58 58 58 58 58 58 58 58
*
1048576
```

## exchange-range 구간별 tracepoint

**관찰 출력**

```text
trace lines=34
== phase A (lines=15)
      8  xfs_bmap_deferred: 
      3  xfs_exchmaps_defer: 
      1  xfs_exchrange_flush: 
      1  xfs_exchrange_freshness: 
      1  xfs_exchrange_mappings: 
      1  xfs_exchrange_prep: 
xfs_io-4665    [001] .....   337.049462: xfs_exchrange_prep: dev 7:0 flags F1_WRITTEN|CMTIME1|CMTIME2|FRESH2 bytecount 0x100000 ino1 0x85 isize 0x100000 disize 0x100000 pos 0x0 -> ino2 0x83 isize 0x100000 disize 0x100000 pos 0x0
xfs_io-4665    [001] .....   337.049463: xfs_exchrange_freshness: dev 7:0 ino 0x83 mtime 1792715698:544752406 ctime 1792715698:544752406 -> file 0x83 mtime 1792715698:544752406 ctime 1792715698:544752406
xfs_io-4665    [001] .....   337.049463: xfs_exchrange_flush: dev 7:0 flags F1_WRITTEN|CMTIME1|CMTIME2|FRESH2 bytecount 0x100000 ino1 0x85 isize 0x100000 disize 0x100000 pos 0x0 -> ino2 0x83 isize 0x100000 disize 0x100000 pos 0x0
xfs_io-4665    [001] .....   337.049478: xfs_exchrange_mappings: dev 7:0 flags F1_WRITTEN|CMTIME1|CMTIME2|FRESH2 bytecount 0x100000 ino1 0x85 isize 0x100000 disize 0x100000 pos 0x0 -> ino2 0x83 isize 0x100000 disize 0x100000 pos 0x0
xfs_io-4665    [001] .....   337.049481: xfs_exchmaps_defer: dev 7:0 ino1 0x85 fileoff1 0x0 ino2 0x83 fileoff2 0x0 fsbcount 0x100 flags (INO1_WRITTEN) isize1 0x100000 newisize1 0xffffffffffffffff isize2 0x100000 newisize2 0xffffffffffffffff
xfs_io-4665    [001] .....   337.049489: xfs_exchmaps_defer: dev 7:0 ino1 0x85 fileoff1 0x20 ino2 0x83 fileoff2 0x20 fsbcount 0xe0 flags (INO1_WRITTEN) isize1 0x100000 newisize1 0xffffffffffffffff isize2 0x100000 newisize2 0xfffffffffffffff
xfs_io-4665    [001] .....   337.049493: xfs_bmap_deferred: dev 7:0 op unmap ino 0x85 agno 0x0 gbno 0x218 data fileoff 0x10 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049501: xfs_bmap_deferred: dev 7:0 op unmap ino 0x83 agno 0x0 gbno 0x28 data fileoff 0x10 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049504: xfs_bmap_deferred: dev 7:0 op map ino 0x85 agno 0x0 gbno 0x28 data fileoff 0x10 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049507: xfs_bmap_deferred: dev 7:0 op map ino 0x83 agno 0x0 gbno 0x218 data fileoff 0x10 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049509: xfs_exchmaps_defer: dev 7:0 ino1 0x85 fileoff1 0xd0 ino2 0x83 fileoff2 0xd0 fsbcount 0x30 flags (INO1_WRITTEN) isize1 0x100000 newisize1 0xffffffffffffffff isize2 0x100000 newisize2 0xfffffffffffffff
xfs_io-4665    [001] .....   337.049511: xfs_bmap_deferred: dev 7:0 op unmap ino 0x85 agno 0x0 gbno 0x228 data fileoff 0xc0 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049513: xfs_bmap_deferred: dev 7:0 op unmap ino 0x83 agno 0x0 gbno 0xd8 data fileoff 0xc0 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049515: xfs_bmap_deferred: dev 7:0 op map ino 0x85 agno 0x0 gbno 0xd8 data fileoff 0xc0 fsbcount 0x10 state 0
xfs_io-4665    [001] .....   337.049517: xfs_bmap_deferred: dev 7:0 op map ino 0x83 agno 0x0 gbno 0x228 data fileoff 0xc0 fsbcount 0x10 state 0
== phase B (lines=0)
== phase B2 (lines=3)
      1  xfs_exchrange_error: 
      1  xfs_exchrange_freshness: 
      1  xfs_exchrange_prep: 
xfs_io-4673    [002] .....   340.055199: xfs_exchrange_prep: dev 7:0 flags F1_WRITTEN|CMTIME1|CMTIME2|FRESH2 bytecount 0x100000 ino1 0x85 isize 0x100000 disize 0x100000 pos 0x0 -> ino2 0x84 isize 0x100000 disize 0x100000 pos 0x0
xfs_io-4673    [002] .....   340.055373: xfs_exchrange_freshness: dev 7:0 ino 0x84 mtime 1792715700:129599450 ctime 1792715700:129599450 -> file 0x84 mtime 1792715698:545634651 ctime 1792715698:545634651
xfs_io-4673    [002] .....   340.055374: xfs_exchrange_error: dev 7:0 ino 0x84 error -16 caller xfs_exchange_range+0x146/0x390
```

