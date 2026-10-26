# fallocate와 mmap: 원본 출력

> 본문은 ../21-fallocate-mmap.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## written, unwritten file에 fzero, fpunch, falloc -k, 부분 block fpunch를 한 뒤 extent map

**관찰 출력**

```text
== c2z
/mnt/fm21a/c2z:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        192..319          0 (192..319)         128 000000
   1: [128..383]:      832..1087         0 (832..1087)        256 010000
   2: [384..639]:      576..831          0 (576..831)         256 000000
== c2p
/mnt/fm21a/c2p:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        1088..1215        0 (1088..1215)       128 000000
   1: [128..383]:      hole                                   256
   2: [384..639]:      1472..1727        0 (1472..1727)       256 000000
== c3z
/mnt/fm21a/c3z:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        1728..1855        0 (1728..1855)       128 010000
   1: [128..383]:      2368..2623        0 (2368..2623)       256 010000
   2: [384..639]:      2112..2367        0 (2112..2367)       256 010000
stat.size = 327680
stat.blocks = 640
fsxattr.xflags = 0x80000002 [-p--------------X]
fsxattr.nextents = 3
== c3p
/mnt/fm21a/c3p:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        2624..2751        0 (2624..2751)       128 010000
   1: [128..383]:      hole                                   256
   2: [384..639]:      3008..3263        0 (3008..3263)       256 010000
== c8p
/mnt/fm21a/c8p:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        320..447          0 (320..447)         128 010000
   1: [128..639]:      hole                                   512
== ck
stat.size = 65536
stat.blocks = 640
fsxattr.xflags = 0x80000002 [-p--------------X]
fsxattr.nextents = 2
/mnt/fm21a/ck:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        3264..3391        0 (3264..3391)       128 000000
   1: [128..639]:      3392..3903        0 (3392..3903)       512 010000
== cu
/mnt/fm21a/cu:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..135]:        3904..4039        0 (3904..4039)       136 000000
   1: [136..375]:      hole                                   240
   2: [376..639]:      4280..4543        0 (4280..4543)       264 000000
c2z ino=131 size=327680 blocks=640
c2p ino=132 size=327680 blocks=384
c3z ino=133 size=327680 blocks=640
c3p ino=134 size=327680 blocks=384
c8p ino=135 size=327680 blocks=128
ck ino=136 size=65536 blocks=640
cu ino=137 size=327680 blocks=400
```

## fzero, fpunch, falloc -k의 event 순서

**관찰 출력**

```text
              sh-2036    [000] ...1.   719.649583: tracing_mark_write: C2Z-data-fzero
          xfs_io-2037    [006] .....   719.651091: xfs_zero_file_space: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-2037    [006] .....   719.651092: xfs_free_file_space: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-2037    [006] .....   719.651098: xfs_bunmap: dev 7:0 ino 0x83 disize 0x50000 fileoff 0x10 fsbcount 0x20 flags  caller xfs_bunmapi+0x19/0x40
          xfs_io-2037    [006] .....   719.651111: xfs_alloc_file_space: dev 7:0 ino 0x83 iflags 0x0
              sh-2039    [000] ...1.   719.653004: tracing_mark_write: C2P-data-fpunch
          xfs_io-2040    [000] .....   719.654125: xfs_free_file_space: dev 7:0 ino 0x84 iflags 0x0
          xfs_io-2040    [000] .....   719.654129: xfs_bunmap: dev 7:0 ino 0x84 disize 0x50000 fileoff 0x10 fsbcount 0x20 flags  caller xfs_bunmapi+0x19/0x40
              sh-2042    [000] ...1.   719.656179: tracing_mark_write: C3Z-unwritten-fzero
          xfs_io-2043    [006] .....   719.657114: xfs_alloc_file_space: dev 7:0 ino 0x85 iflags 0x0
          xfs_io-2043    [006] .....   719.657125: xfs_zero_file_space: dev 7:0 ino 0x85 iflags 0x0
          xfs_io-2043    [006] .....   719.657125: xfs_free_file_space: dev 7:0 ino 0x85 iflags 0x0
          xfs_io-2043    [006] .....   719.657125: xfs_bunmap: dev 7:0 ino 0x85 disize 0x50000 fileoff 0x10 fsbcount 0x20 flags  caller xfs_bunmapi+0x19/0x40
          xfs_io-2043    [006] .....   719.657132: xfs_alloc_file_space: dev 7:0 ino 0x85 iflags 0x0
              sh-2045    [006] ...1.   719.659158: tracing_mark_write: C3P-unwritten-fpunch
          xfs_io-2046    [005] .....   719.660083: xfs_alloc_file_space: dev 7:0 ino 0x86 iflags 0x0
          xfs_io-2046    [005] .....   719.660093: xfs_free_file_space: dev 7:0 ino 0x86 iflags 0x0
          xfs_io-2046    [005] .....   719.660094: xfs_bunmap: dev 7:0 ino 0x86 disize 0x50000 fileoff 0x10 fsbcount 0x20 flags  caller xfs_bunmapi+0x19/0x40
              sh-2048    [004] ...1.   719.662046: tracing_mark_write: C8P-unwritten-to-hole
          xfs_io-2049    [005] .....   719.662933: xfs_alloc_file_space: dev 7:0 ino 0x87 iflags 0x0
          xfs_io-2049    [005] .....   719.662943: xfs_free_file_space: dev 7:0 ino 0x87 iflags 0x0
          xfs_io-2049    [005] .....   719.662944: xfs_bunmap: dev 7:0 ino 0x87 disize 0x50000 fileoff 0x10 fsbcount 0x20 flags  caller xfs_bunmapi+0x19/0x40
              sh-2051    [001] ...1.   719.664896: tracing_mark_write: CK-falloc-keep-size
          xfs_io-2052    [000] .....   719.665894: xfs_alloc_file_space: dev 7:0 ino 0x88 iflags 0x0
              sh-2054    [000] ...1.   719.667726: tracing_mark_write: CU-unaligned-fpunch
          xfs_io-2055    [001] .....   719.668735: xfs_free_file_space: dev 7:0 ino 0x89 iflags 0x0
          xfs_io-2055    [001] .....   719.668739: xfs_bunmap: dev 7:0 ino 0x89 disize 0x50000 fileoff 0x11 fsbcount 0x1e flags  caller xfs_bunmapi+0x19/0x40
```

## finsert, fcollapse, funshare 전후 extent map

**관찰 출력**

```text
== s1 create
/mnt/fm21b/s1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
== finsert 512k 64k, then pwrite into the hole
/mnt/fm21b/s1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..1023]:       192..1215         0 (192..1215)       1024 000000
   1: [1024..1151]:    hole                                   128
   2: [1152..2175]:    1216..2239        0 (1216..2239)      1024 000000
/mnt/fm21b/s1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..1023]:       192..1215         0 (192..1215)       1024 000000
   1: [1024..1151]:    2240..2367        0 (2240..2367)       128 000000
   2: [1152..2175]:    1216..2239        0 (1216..2239)      1024 000000
s1 size=1114112 blocks=2176
== fcollapse 512k 64k
/mnt/fm21b/s1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       192..2239         0 (192..2239)       2048 000000
s1 size=1048576 blocks=2048
== reflink u1 -> u2
/mnt/fm21b/u2:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       2368..4415        2048 0x2001
/mnt/fm21b/u2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       2368..4415        0 (2368..4415)      2048 100000
-- u1
/mnt/fm21b/u1:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       2368..4415        2048 0x2001
== funshare u2 0 1m
/mnt/fm21b/u2:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       4416..6463        2048   0x1
/mnt/fm21b/u2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..2047]:       4416..6463        0 (4416..6463)      2048 000000
-- u1
/mnt/fm21b/u1:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..2047]:       2368..4415        2048   0x1
s1 ino=131 size=1048576 blocks=2048
u1 ino=132 size=1048576 blocks=2048
u2 ino=133 size=1048576 blocks=2048
```

## finsert, fcollapse, funshare의 event 순서

**관찰 출력**

```text
              sh-2190    [005] ...1.   728.699895: tracing_mark_write: INSERT
          xfs_io-2191    [002] .....   728.700805: xfs_insert_file_space: dev 7:0 ino 0x83 iflags 0x0
              sh-2194    [003] ...1.   728.703234: tracing_mark_write: COLLAPSE
          xfs_io-2195    [000] .....   728.704114: xfs_collapse_file_space: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-2195    [000] .....   728.704114: xfs_free_file_space: dev 7:0 ino 0x83 iflags 0x0
          xfs_io-2195    [000] .....   728.704118: xfs_bunmap: dev 7:0 ino 0x83 disize 0x110000 fileoff 0x80 fsbcount 0x10 flags  caller xfs_bunmapi+0x19/0x40
          xfs_io-2195    [000] .....   728.704139: xfs_itruncate_extents_start: dev 7:0 ino 0x83 disize 0x100000 new_size 0x100000
          xfs_io-2195    [000] .....   728.704140: xfs_bunmap: dev 7:0 ino 0x83 disize 0x100000 fileoff 0x100 fsbcount 0x400000001ffeff flags  caller xfs_bunmapi_range+0x81/0xb0
              sh-2198    [007] ...1.   728.706362: tracing_mark_write: REFLINK
              sh-2203    [004] ...1.   728.710975: tracing_mark_write: UNSHARE
          xfs_io-2204    [000] .....   728.711865: xfs_reflink_unshare: dev 7:0 ino 0x85 isize 0x100000 disize 0x100000 pos 0x0 bytecount 0x100000
     kworker/3:1-70      [003] .....   728.712104: xfs_reflink_end_cow: dev 7:0 ino 0x85 isize 0x100000 disize 0x100000 pos 0x0 bytecount 0x100000
     kworker/3:1-70      [003] .....   728.712110: xfs_bunmap: dev 7:0 ino 0x85 disize 0x100000 fileoff 0x0 fsbcount 0x100 flags REMAP caller xfs_bmap_update_finish_item+0x16/0x40
```

## close, spaceman, background blockgc worker의 정리 결과와 event 순서

**관찰 출력**

```text
lifetime=5 (was 300)
== a: one open, 1m write
stat.size = 1048576
stat.blocks = 2048
a closed size=1048576 blocks=2048
== d: falloc -k 0 1m, write 64k
d closed size=65536 blocks=2048
== b: xfs/014 loop, then 128m
b closed size=134217728 blocks=524160
== e: xfs/014 loop, then 8m
e closed size=8388608 blocks=32640
== spaceman prealloc -m 64m -s
b size=134217728 blocks=262144
e size=8388608 blocks=32640
== fsync e and b, wait 15s for blockgc worker
e after fsync size=8388608 blocks=32640
a ino=131 size=1048576 blocks=2048
b ino=133 size=134217728 blocks=262144
d ino=132 size=65536 blocks=2048
e ino=134 size=8388608 blocks=16384
              sh-5090    [000] ...1.   924.564936: tracing_mark_write: A-single-open
          xfs_io-5091    [000] .....   924.566338: xfs_itruncate_extents_start: dev 7:0 ino 0x83 disize 0x0 new_size 0x100000
          xfs_io-5091    [000] .....   924.566341: xfs_inode_clear_eofblocks_tag: dev 7:0 ino 0x83 iflags 0x40
              sh-5095    [005] ...1.   924.568681: tracing_mark_write: D-falloc-keep
              sh-5099    [000] ...1.   924.572594: tracing_mark_write: B-xfs014-128m
          xfs_io-5101    [000] .....   924.574228: xfs_itruncate_extents_start: dev 7:0 ino 0x85 disize 0x0 new_size 0x8000
          xfs_io-5101    [000] .....   924.574229: xfs_inode_clear_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x40
          xfs_io-5103    [001] .....   924.576151: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 32 shift 0 m_allocsize_blocks 16
          xfs_io-5103    [001] .....   924.576153: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x40
          xfs_io-5107    [004] .....   924.579607: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 64 shift 0 m_allocsize_blocks 16
          xfs_io-5107    [004] .....   924.579608: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.582066: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 128 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.582067: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.582094: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 512 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.582094: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.582220: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 1024 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.582220: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.582463: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 2048 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.582464: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.582954: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 4096 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.582954: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.583422: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 8192 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.583422: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.585205: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 16384 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.585207: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
          xfs_io-5110    [006] .....   924.589380: xfs_iomap_prealloc_size: dev 7:0 ino 0x85 prealloc blocks 32768 shift 0 m_allocsize_blocks 16
          xfs_io-5110    [006] .....   924.589382: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
              sh-5113    [006] ...1.   924.593061: tracing_mark_write: E-xfs014-8m
          xfs_io-5115    [002] .....   924.594668: xfs_itruncate_extents_start: dev 7:0 ino 0x86 disize 0x0 new_size 0x8000
          xfs_io-5115    [002] .....   924.594671: xfs_inode_clear_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x40
          xfs_io-5117    [004] .....   924.596254: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 32 shift 0 m_allocsize_blocks 16
          xfs_io-5117    [004] .....   924.596255: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x40
          xfs_io-5121    [006] .....   924.599522: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 64 shift 0 m_allocsize_blocks 16
          xfs_io-5121    [006] .....   924.599522: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
          xfs_io-5124    [006] .....   924.602025: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 128 shift 0 m_allocsize_blocks 16
          xfs_io-5124    [006] .....   924.602026: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
          xfs_io-5124    [006] .....   924.602065: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 512 shift 0 m_allocsize_blocks 16
          xfs_io-5124    [006] .....   924.602065: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
          xfs_io-5124    [006] .....   924.602126: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 1024 shift 0 m_allocsize_blocks 16
          xfs_io-5124    [006] .....   924.602126: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
          xfs_io-5124    [006] .....   924.602405: xfs_iomap_prealloc_size: dev 7:0 ino 0x86 prealloc blocks 2048 shift 0 m_allocsize_blocks 16
          xfs_io-5124    [006] .....   924.602406: xfs_inode_set_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
              sh-5127    [006] ...1.   924.605021: tracing_mark_write: SPACEMAN-prealloc-m64m-s
    xfs_spaceman-5129    [001] .....   924.607617: xfs_ioc_free_eofblocks: dev 7:0 flags 0x11 uid 4294967295 gid 4294967295 prid 0 minsize 67108864 scan_limit 0 caller __x64_sys_ioctl+0x96/0xe0
    xfs_spaceman-5129    [001] .....   924.607618: xfs_blockgc_free_space: dev 7:0 flags 0x11 uid 4294967295 gid 4294967295 prid 0 minsize 67108864 scan_limit 0 caller xfs_file_ioctl+0x37a/0x1130
    xfs_spaceman-5129    [001] .....   924.607621: xfs_itruncate_extents_start: dev 7:0 ino 0x85 disize 0x0 new_size 0x8000000
    xfs_spaceman-5129    [001] .....   924.607626: xfs_inode_clear_eofblocks_tag: dev 7:0 ino 0x85 iflags 0x240
              sh-5133    [006] ...1.   924.610529: tracing_mark_write: WAIT-blockgc
   kworker/u32:4-1838    [003] .....   930.003680: xfs_blockgc_worker: dev 7:0 m_features 0xd9ffaab opstate (clean|inodegc|blockgc) s_flags 0x70810000 caller process_one_work+0x16f/0x3d0
   kworker/u32:4-1838    [003] .....   930.003689: xfs_itruncate_extents_start: dev 7:0 ino 0x86 disize 0x800000 new_size 0x800000
   kworker/u32:4-1838    [003] .....   930.003739: xfs_inode_clear_eofblocks_tag: dev 7:0 ino 0x86 iflags 0x240
```

## unwritten 범위와 sparse file의 mmap write fault

**관찰 출력**

```text
== xfs/166 unwritten_mmap 12M
/mnt/fm21c/t:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          192..199          0 (192..199)           8 000000
   1: [8..12287]:      200..12479        0 (200..12479)     12280 010000
   2: [12288..12295]:  12480..12487      0 (12480..12487)       8 000000
   3: [12296..24567]:  12488..24759      0 (12488..24759)   12272 010000
   4: [24568..24575]:  24760..24767      0 (24760..24767)       8 000000
== sparse 1m, mmap write 4k
/mnt/fm21c/h:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          delalloc                                 8
   1: [8..2047]:       hole                                  2040
/mnt/fm21c/h:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          104..111          0 (104..111)           8 000000
   1: [8..2047]:       hole                                  2040
t ino=131 size=12582912 blocks=24576
h ino=132 size=1048576 blocks=8
              sh-2328    [000] ...1.   735.645329: tracing_mark_write: X166-unwritten-mmap
  unwritten_mmap-2329    [003] .....   735.645967: xfs_alloc_file_space: dev 7:0 ino 0x83 iflags 0x0
  unwritten_mmap-2329    [003] .....   735.646013: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646017: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0xbef000 bytecount 0x11000 fork data startoff 0xbef startblock 0xc07 fsbcount 0x11
  unwritten_mmap-2329    [003] .....   735.646020: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646020: xfs_write_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646021: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0xbff000 bytecount 0x1000 fork data startoff 0x0 startblock 0x18 fsbcount 0xc00
  unwritten_mmap-2329    [003] .....   735.646022: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646026: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0x5f0000 bytecount 0x20000 fork data startoff 0x5f0 startblock 0x608 fsbcount 0x20
  unwritten_mmap-2329    [003] .....   735.646030: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646030: xfs_write_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646030: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0x600000 bytecount 0x1000 fork data startoff 0x0 startblock 0x18 fsbcount 0xc00
  unwritten_mmap-2329    [003] .....   735.646031: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646034: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0x0 bytecount 0x20000 fork data startoff 0x0 startblock 0x18 fsbcount 0x20
  unwritten_mmap-2329    [003] .....   735.646037: xfs_read_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646038: xfs_write_fault: dev 7:0 ino 0x83 order 0
  unwritten_mmap-2329    [003] .....   735.646038: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xc00000 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0x18 fsbcount 0xc00
     kworker/6:1-159     [006] .....   735.646056: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0xc00000 disize 0xc00000 pos 0x0 bytecount 0x1000
     kworker/6:1-159     [006] .....   735.646071: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0xc00000 disize 0xc00000 pos 0x600000 bytecount 0x1000
     kworker/6:1-159     [006] .....   735.646078: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0xc00000 disize 0xc00000 pos 0xbff000 bytecount 0x1000
              sh-2332    [001] ...1.   735.648798: tracing_mark_write: HOLE-mmap
          xfs_io-2333    [003] .....   735.649694: xfs_read_fault: dev 7:0 ino 0x84 order 0
          xfs_io-2333    [003] .....   735.649698: xfs_iomap_found: dev 7:0 ino 0x84 disize 0x100000 pos 0x0 bytecount 0x20000 fork data startoff 0x0 startblock 0xfffffffffffffffe fsbcount 0x20
          xfs_io-2333    [003] .....   735.649701: xfs_read_fault: dev 7:0 ino 0x84 order 0
          xfs_io-2333    [003] .....   735.649702: xfs_write_fault: dev 7:0 ino 0x84 order 0
          xfs_io-2333    [003] .....   735.649703: xfs_iomap_alloc: dev 7:0 ino 0x84 disize 0x100000 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
     kworker/6:1-159     [006] .....   735.649743: xfs_unwritten_convert: dev 7:0 ino 0x84 isize 0x100000 disize 0x100000 pos 0x0 bytecount 0x1000
```

