# 할당 한 번의 흐름: 원본 출력

> 본문은 ../22-allocation-flow.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 새 file에 buffered 1 MiB의 event

**관찰 출력**

```text
              sh-2498    [007] ...1.   737.693118: tracing_mark_write: A-buffered-1m
          xfs_io-2499    [002] .....   737.694475: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2499    [002] .....   737.694500: xfs_alloc_cur_check: dev 7:0 cntbt agbno 0x18 fsbcount 0xffe8 diff 0x8 new 1
          xfs_io-2499    [002] .....   737.694500: xfs_alloc_near_first: dev 7:0 agno 0x0 agbno 0x10 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2499    [002] .....   737.694518: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0x18 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2499    [002] .....   737.694519: xfs_ag_resv_alloc_extent: dev 7:0 agno 0x0 resv 0 freeblks 65259 flcount 6 resv 0 ask 0 len 256
          xfs_io-2499    [002] .....   737.694519: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x18 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
          xfs_io-2499    [002] .....   737.694521: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0x100 state 1
          xfs_io-2499    [002] .....   737.694529: xfs_rmap_deferred: dev 7:0 op map agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0x100 state 1
          xfs_io-2499    [002] .....   737.694530: xfs_rmap_map: dev 7:0 agno 0x0 gbno 0x18 fsbcount 0x100 owner 0x83 fileoff 0x0 flags 0x4
          xfs_io-2499    [002] .....   737.694540: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0008 fsbcount 0x100
     kworker/1:1-74      [001] .....   737.694625: xfs_rmap_defer: dev 7:0 op cvt agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0x100 state 0
     kworker/1:1-74      [001] .....   737.694629: xfs_rmap_deferred: dev 7:0 op cvt agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0x100 state 0
```

## 같은 file에 1 MiB 이어 쓰기의 event

**관찰 출력**

```text
              sh-2503    [007] ...1.   737.697939: tracing_mark_write: B-append-1m
          xfs_io-2504    [002] .....   737.699110: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 752 maxlen 752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0xffffffff
          xfs_io-2504    [002] .....   737.699113: xfs_alloc_cur_check: dev 7:0 cntbt agbno 0x118 fsbcount 0xfee8 diff 0x0 new 1
          xfs_io-2504    [002] .....   737.699113: xfs_alloc_near_first: dev 7:0 agno 0x0 agbno 0x118 minlen 752 maxlen 752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 752 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0xffffffff
          xfs_io-2504    [002] .....   737.699115: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0x118 minlen 752 maxlen 752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 752 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0xffffffff
          xfs_io-2504    [002] .....   737.699115: xfs_ag_resv_alloc_extent: dev 7:0 agno 0x0 resv 0 freeblks 64507 flcount 6 resv 0 ask 0 len 752
          xfs_io-2504    [002] .....   737.699115: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x118 minlen 752 maxlen 752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 752 wasdel 1 wasfromfl 0 resv 0 datatype 0x5 highest_agno 0x0
          xfs_io-2504    [002] .....   737.699116: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0x118 owner 0x83 data fileoff 0x100 fsbcount 0x2f0 state 1
          xfs_io-2504    [002] .....   737.699120: xfs_rmap_deferred: dev 7:0 op map agno 0x0 gbno 0x118 owner 0x83 data fileoff 0x100 fsbcount 0x2f0 state 1
          xfs_io-2504    [002] .....   737.699120: xfs_rmap_map: dev 7:0 agno 0x0 gbno 0x118 fsbcount 0x2f0 owner 0x83 fileoff 0x100 flags 0x4
          xfs_io-2504    [002] .....   737.699122: xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x100000 pos 0x100000 bytecount 0x1000 fork data startoff 0x100 startblock 0xffffffffe0009 fsbcount 0x2f0
     kworker/1:1-74      [001] .....   737.699207: xfs_rmap_defer: dev 7:0 op cvt agno 0x0 gbno 0x118 owner 0x83 data fileoff 0x100 fsbcount 0x100 state 0
     kworker/1:1-74      [001] .....   737.699210: xfs_rmap_deferred: dev 7:0 op cvt agno 0x0 gbno 0x118 owner 0x83 data fileoff 0x100 fsbcount 0x100 state 0
          xfs_io-2504    [002] .....   737.699318: xfs_rmap_defer: dev 7:0 op unmap agno 0x0 gbno 0x218 owner 0x83 data fileoff 0x200 fsbcount 0x1f0 state 1
          xfs_io-2504    [002] .....   737.699321: xfs_rmap_deferred: dev 7:0 op unmap agno 0x0 gbno 0x218 owner 0x83 data fileoff 0x200 fsbcount 0x1f0 state 1
          xfs_io-2504    [002] .....   737.699329: xfs_extent_busy: dev 7:0 agno 0x0 agbno 0x218 fsbcount 0x1f0
```

## 새 file에 direct 256 KiB의 event

**관찰 출력**

```text
              sh-2507    [003] ...1.   737.701456: tracing_mark_write: C-direct-256k
          xfs_io-2508    [002] .....   737.702337: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 64 maxlen 64 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2508    [002] ...1.   737.702339: xfs_extent_busy_trim: dev 7:0 agno 0x0 agbno 0x218 fsbcount 0xfde8 found_agbno 0x408 found_fsbcount 0xfbf8
          xfs_io-2508    [002] .....   737.702340: xfs_alloc_cur_check: dev 7:0 cntbt agbno 0x218 fsbcount 0xfde8 diff 0x3f8 new 1
          xfs_io-2508    [002] .....   737.702340: xfs_alloc_near_first: dev 7:0 agno 0x0 agbno 0x10 minlen 64 maxlen 64 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 64 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2508    [002] .....   737.702341: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0x408 minlen 64 maxlen 64 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 64 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2508    [002] .....   737.702341: xfs_ag_resv_alloc_extent: dev 7:0 agno 0x0 resv 0 freeblks 64939 flcount 6 resv 0 ask 0 len 64
          xfs_io-2508    [002] .....   737.702342: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x408 minlen 64 maxlen 64 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 64 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
          xfs_io-2508    [002] .....   737.702343: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0x408 owner 0x84 data fileoff 0x0 fsbcount 0x40 state 1
          xfs_io-2508    [002] .....   737.702345: xfs_rmap_deferred: dev 7:0 op map agno 0x0 gbno 0x408 owner 0x84 data fileoff 0x0 fsbcount 0x40 state 1
          xfs_io-2508    [002] .....   737.702346: xfs_rmap_map: dev 7:0 agno 0x0 gbno 0x408 fsbcount 0x40 owner 0x84 fileoff 0x0 flags 0x4
          xfs_io-2508    [002] .....   737.702393: xfs_rmap_defer: dev 7:0 op cvt agno 0x0 gbno 0x408 owner 0x84 data fileoff 0x0 fsbcount 0x40 state 0
          xfs_io-2508    [002] .....   737.702395: xfs_rmap_deferred: dev 7:0 op cvt agno 0x0 gbno 0x408 owner 0x84 data fileoff 0x0 fsbcount 0x40 state 0
```

## 세 번의 할당 전후의 extent 배치와 extent_alloc

**관찰 출력**

```text
events=29
extent_alloc 20 5457 11 1214
wrote 1048576/1048576 bytes at offset 0
1 MiB, 1 ops; 0.0001 sec (7.455 GiB/sec and 7633.5878 ops/sec)
/mnt/ag22a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
extent_alloc 21 5713 11 1214
== f1 after close
/mnt/ag22a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
wrote 1048576/1048576 bytes at offset 1048576
1 MiB, 1 ops; 0.0001 sec (7.940 GiB/sec and 8130.0813 ops/sec)
/mnt/ag22a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..4095]:       192..4287         0 (192..4287)       4096 000000
extent_alloc 22 6465 12 1710
wrote 262144/262144 bytes at offset 0
256 KiB, 1 ops; 0.0000 sec (3.756 GiB/sec and 15384.6154 ops/sec)
/mnt/ag22a/f2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..511]:        8256..8767        0 (8256..8767)       512 000000
extent_alloc 23 6529 12 1710
f1 ino=131 blocks=4096 f2 ino=132 blocks=512
== f1 final
/mnt/ag22a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..4095]:       192..4287         0 (192..4287)       4096 000000
```

## 1 MiB 할당 전후의 superblock, AGF, bnobt, cntbt, fsmap, inode

**관찰 출력**

```text
#### before
fdblocks = 245724
icount = 64
ifree = 61
== AG 0 AGF
freeblks = 65515
longest = 65512
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
bnolevel = 1
cntlevel = 1
rmaplevel = 1
== AG 0 bnobt
bnobt level 0 block 1 daddr 8
recs[1-2] = [startblock,blockcount] 
1:[13,3] 
2:[24,65512]
== AG 0 cntbt
cntbt level 0 block 1 daddr 16
recs[1-2] = [startblock,blockcount] 
1:[13,3] 
2:[24,65512]
ino=131 blocks=2048
/mnt/ag22b/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
extent_alloc 24 6785 12 1710
#### after
fdblocks = 245468
icount = 64
ifree = 60
== AG 0 AGF
freeblks = 65259
longest = 65256
flfirst = 1
fllast = 6
flcount = 6
btreeblks = 0
bnolevel = 1
cntlevel = 1
rmaplevel = 1
== AG 0 bnobt
bnobt level 0 block 1 daddr 8
recs[1-2] = [startblock,blockcount] 
1:[13,3] 
2:[280,65256]
== AG 0 cntbt
cntbt level 0 block 1 daddr 16
recs[1-2] = [startblock,blockcount] 
1:[13,3] 
2:[280,65256]
== fsmap
7: 0/24 len 256 owner 131 offset 0 bmbt 0 attrfork 0 extflag 0
== inode 131
core.size = 1048576
core.nblocks = 256
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,24,256,0]
```

## 공통 예제 image의 AGF와 4 KiB, 1 MiB 요청의 event

**관찰 출력**

```text
== AG 0 AGF
freeblks = 124589
longest = 123605
flcount = 12
cntlevel = 2
events=18
root ino=128
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (355.114 MiB/sec and 90909.0909 ops/sec)
/mnt/ag22d/ag22-small:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          10152..10159      0 (10152..10159)       8 000000
wrote 1048576/1048576 bytes at offset 0
1 MiB, 1 ops; 0.0001 sec (8.206 GiB/sec and 8403.3613 ops/sec)
/mnt/ag22d/ag22-big:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       23736..25783      0 (23736..25783)    2048 000000
small ino=4477 big ino=4496
              sh-2933    [007] ...1.   743.829331: tracing_mark_write: A-pop-4k
          xfs_io-2934    [002] .....   743.830498: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 1 maxlen 1 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2934    [002] .....   743.830534: xfs_alloc_cur_lookup: dev 7:0 agno 0x0 agbno 0x22f minlen 1 maxlen 1 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2934    [002] .....   743.830535: xfs_alloc_cur_check: dev 7:0 bnobt agbno 0x4f5 fsbcount 0x1 diff 0x2c6 new 1
          xfs_io-2934    [002] .....   743.830535: xfs_alloc_cur_right: dev 7:0 agno 0x0 agbno 0x22f minlen 1 maxlen 1 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 1 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2934    [002] .....   743.830551: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0x4f5 minlen 1 maxlen 1 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 1 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2934    [002] .....   743.830552: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x4f5 minlen 1 maxlen 1 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 1 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
          xfs_io-2934    [002] .....   743.830554: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0x4f5 owner 0x117d data fileoff 0x0 fsbcount 0x1 state 1
          xfs_io-2934    [002] .....   743.830583: xfs_map_blocks_alloc: dev 7:0 ino 0x117d disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
     kworker/3:1-70      [003] .....   743.830596: xfs_rmap_defer: dev 7:0 op cvt agno 0x0 gbno 0x4f5 owner 0x117d data fileoff 0x0 fsbcount 0x1 state 0
              sh-2936    [000] ...1.   743.832557: tracing_mark_write: B-pop-1m
          xfs_io-2937    [004] .....   743.833734: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2937    [004] .....   743.833738: xfs_alloc_cur_check: dev 7:0 cntbt agbno 0xb97 fsbcount 0x138 diff 0x965 new 1
          xfs_io-2937    [004] .....   743.833738: xfs_alloc_cur_check: dev 7:0 cntbt agbno 0x1d2b fsbcount 0x1e2d5 diff 0x1af9 new 0
          xfs_io-2937    [004] .....   743.833738: xfs_alloc_near_first: dev 7:0 agno 0x0 agbno 0x232 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2937    [004] .....   743.833741: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0xb97 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2937    [004] .....   743.833741: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0xb97 minlen 256 maxlen 256 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 256 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
          xfs_io-2937    [004] .....   743.833742: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0xb97 owner 0x1190 data fileoff 0x0 fsbcount 0x100 state 1
          xfs_io-2937    [004] .....   743.833784: xfs_map_blocks_alloc: dev 7:0 ino 0x1190 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0008 fsbcount 0x100
     kworker/3:1-70      [003] .....   743.833852: xfs_rmap_defer: dev 7:0 op cvt agno 0x0 gbno 0xb97 owner 0x1190 data fileoff 0x0 fsbcount 0x100 state 0
```

## AG별 free space와 220 MiB falloc, 64 MiB write의 event

**관찰 출력**

```text
events=16
        AG    extents     blocks
         0          2      65515
         1          1      65523
         2          1      49139
         3          1      65523
/mnt/ag22c/fa:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET         TOTAL FLAGS
   0: [0..450559]:     192..450751       0 (192..450751)    450560 010000
        AG    extents     blocks
         0          2       9195
         1          1      65523
         2          1      49139
         3          1      65523
wrote 67108864/67108864 bytes at offset 0
64 MiB, 64 ops; 0.0038 sec (16.117 GiB/sec and 16503.3522 ops/sec)
/mnt/ag22c/fb:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET         TOTAL FLAGS
   0: [0..131071]:     524392..655463    1 (104..131175)    131072 000000
        AG    extents     blocks
         0          2       9195
         1          1      49139
         2          1      49139
         3          1      65523
fa ino=131 fb ino=132
              sh-2777    [006] ...1.   741.690657: tracing_mark_write: A-falloc-220m
          xfs_io-2778    [001] .....   741.691654: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 56320 maxlen 56320 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2778    [001] .....   741.691672: xfs_alloc_near_first: dev 7:0 agno 0x0 agbno 0x10 minlen 56320 maxlen 56320 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 56320 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2778    [001] .....   741.691680: xfs_alloc_cur: dev 7:0 agno 0x0 agbno 0x18 minlen 56320 maxlen 56320 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 56320 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2778    [001] .....   741.691681: xfs_ag_resv_alloc_extent: dev 7:0 agno 0x0 resv 0 freeblks 9195 flcount 6 resv 0 ask 0 len 56320
          xfs_io-2778    [001] .....   741.691681: xfs_alloc_vextent_finish: dev 7:0 agno 0x0 agbno 0x18 minlen 56320 maxlen 56320 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 56320 wasdel 0 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x0
          xfs_io-2778    [001] .....   741.691683: xfs_rmap_defer: dev 7:0 op map agno 0x0 gbno 0x18 owner 0x83 data fileoff 0x0 fsbcount 0xdc00 state 1
              sh-2782    [005] ...1.   741.695828: tracing_mark_write: B-buffered-64m
          xfs_io-2783    [006] .....   741.700821: xfs_alloc_vextent_start_ag: dev 7:0 agno 0xffffffff agbno 0xffffffff minlen 32752 maxlen 32752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2783    [006] .....   741.700823: xfs_alloc_vextent_noagbp: dev 7:0 agno 0x0 agbno 0xffffffff minlen 32752 maxlen 32752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2783    [006] .....   741.700823: xfs_alloc_vextent_loopfailed: dev 7:0 agno 0x0 agbno 0xffffffff minlen 32752 maxlen 32752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 0 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2783    [006] .....   741.700887: xfs_alloc_size_done: dev 7:0 agno 0x1 agbno 0xd minlen 32752 maxlen 32752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 32752 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0xffffffff
          xfs_io-2783    [006] .....   741.700888: xfs_ag_resv_alloc_extent: dev 7:0 agno 0x1 resv 0 freeblks 32771 flcount 6 resv 0 ask 0 len 32752
          xfs_io-2783    [006] .....   741.700889: xfs_alloc_vextent_finish: dev 7:0 agno 0x1 agbno 0xd minlen 32752 maxlen 32752 mod 0 prod 1 minleft 1 total 0 alignment 1 minalignslop 0 len 32752 wasdel 1 wasfromfl 0 resv 0 datatype 0x7 highest_agno 0x1
          xfs_io-2783    [006] .....   741.700892: xfs_rmap_defer: dev 7:0 op map agno 0x1 gbno 0xd owner 0x84 data fileoff 0x0 fsbcount 0x7ff0 state 1
     kworker/3:1-70      [003] .....   741.709445: xfs_rmap_defer: dev 7:0 op cvt agno 0x1 gbno 0xd owner 0x84 data fileoff 0x0 fsbcount 0x4000 state 0
          xfs_io-2783    [006] .....   741.709768: xfs_rmap_defer: dev 7:0 op unmap agno 0x1 gbno 0x400d owner 0x84 data fileoff 0x4000 fsbcount 0x3ff0 state 1
```

## 1 MiB write와 fsync의 xfs_bmapi_allocate 호출 tree

**관찰 출력**

```text
 3)  xfs_io-3286   |               |  xfs_bmapi_allocate() {
 3)  xfs_io-3286   |               |    xfs_bmap_btalloc() {
 3)  xfs_io-3286   |   0.160 us    |      xfs_bmap_adjacent();
 3)  xfs_io-3286   |               |      xfs_bmap_longest_free_extent() {
 3)  xfs_io-3286   |               |      xfs_alloc_vextent_start_ag() {
 3)  xfs_io-3286   |               |        xfs_alloc_vextent_iterate_ags.constprop.0() {
 3)  xfs_io-3286   |               |          xfs_alloc_vextent_prepare_ag() {
 3)  xfs_io-3286   |               |            xfs_alloc_fix_freelist() {
 3)  xfs_io-3286   |               |          xfs_alloc_ag_vextent_near() {
 3)  xfs_io-3286   |               |              xfs_alloc_cur_check() {
 3)  xfs_io-3286   |               |            xfs_alloc_cur_finish() {
 3)  xfs_io-3286   |               |              xfs_alloc_fixup_trees() {
 3)  xfs_io-3286   |               |                xfs_alloc_fixup_longest() {
 3)  xfs_io-3286   |               |        xfs_alloc_vextent_finish() {
 3)  xfs_io-3286   |               |          xfs_alloc_update_counters() {
 3)  xfs_io-3286   |               |          xfs_ag_resv_alloc_extent() {
 3)  xfs_io-3286   |               |      xfs_bmap_alloc_account() {
 3)  xfs_io-3286   |               |    xfs_bmap_add_extent_delay_real() {
 3)  xfs_io-3286   |               |      xfs_rmap_map_extent() {
 3)  xfs_io-3286   |               |          xfs_rmap_defer_add() {
```

