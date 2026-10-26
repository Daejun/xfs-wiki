# direct I/O와 read: 원본 출력

> 본문은 ../19-direct-io-read.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 새 file에 52 KiB direct write: tracepoint

**관찰 출력**

```text
              sh-6481    [004] ...1.  2528.849160: tracing_mark_write: s1-new-52k
          xfs_io-6482    [000] .....  2528.850169: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-6482    [000] .....  2528.850170: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_write_checks+0x279/0x2d0
          xfs_io-6482    [000] .....  2528.850171: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_attr_map_shared+0x2a/0x50
          xfs_io-6482    [000] .....  2528.850173: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-6482    [000] .....  2528.850176: xfs_ilock_demote: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_file_dio_write_aligned+0x158/0x170
          xfs_io-6482    [000] .....  2528.850176: xfs_file_direct_write: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0xd000
          xfs_io-6482    [000] .....  2528.850178: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-6482    [000] .....  2528.850179: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-6482    [000] .....  2528.850208: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0xd000 fork data startoff 0x0 startblock 0x28 fsbcount 0xd
          xfs_io-6482    [000] .....  2528.850209: iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0x28000 offset 0x0 length 0xd000 type UNWRITTEN (0x3) flags NEW|DIRTY (0x3)
          xfs_io-6482    [000] .....  2528.850213: block_bio_queue: 7,0 WS 320 + 104 [xfs_io]
          xfs_io-6482    [000] .....  2528.850255: xfs_end_io_direct_write: dev 7:0 ino 0x83 isize 0x0 disize 0x0 pos 0x0 bytecount 0xd000
          xfs_io-6482    [000] .....  2528.850255: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x0 disize 0x0 pos 0x0 bytecount 0xd000
          xfs_io-6482    [000] .....  2528.850256: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-6482    [000] .....  2528.850261: iomap_dio_complete: dev 7:0 ino 0x83 size 0xd000 offset 0xd000 flags DIRECT aio 0 error 0 ret 53248
```

## 새 file에 52 KiB direct write 뒤의 extent와 stat

**관찰 출력**

```text
== s1 new file 52k
/mnt/dio19a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..103]:        320..423          0 (320..423)         104 000000
stat.size = 53248
stat.blocks = 104
fsxattr.extsize = 0
```

## 같은 범위 덮어쓰기: tracepoint

**관찰 출력**

```text
              sh-6485    [002] ...1.  2528.853440: tracing_mark_write: s2-overwrite-52k
          xfs_io-6486    [002] .....  2528.854520: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_SHARED caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-6486    [002] .....  2528.854521: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-6486    [002] .....  2528.854523: xfs_file_direct_write: dev 7:0 ino 0x83 disize 0xd000 pos 0x0 bytecount 0xd000
          xfs_io-6486    [002] .....  2528.854524: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-6486    [002] .....  2528.854524: xfs_iomap_found: dev 7:0 ino 0x83 disize 0xd000 pos 0x0 bytecount 0xd000 fork data startoff 0x0 startblock 0x28 fsbcount 0xd
          xfs_io-6486    [002] .....  2528.854525: iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0x28000 offset 0x0 length 0xd000 type MAPPED (0x2) flags DIRTY (0x2)
          xfs_io-6486    [002] .....  2528.854526: block_bio_queue: 7,0 WS 320 + 104 [xfs_io]
          xfs_io-6486    [002] .....  2528.854546: xfs_end_io_direct_write: dev 7:0 ino 0x83 isize 0xd000 disize 0xd000 pos 0x0 bytecount 0xd000
          xfs_io-6486    [002] .....  2528.854546: iomap_dio_complete: dev 7:0 ino 0x83 size 0xd000 offset 0xd000 flags DIRECT aio 0 error 0 ret 53248
```

## EOF에 닿는 512 byte direct write: tracepoint

**관찰 출력**

```text
              sh-6488    [003] ...1.  2528.856567: tracing_mark_write: s3-subblock-extend
          xfs_io-6489    [007] .....  2528.857555: xfs_ilock: dev 7:0 ino 0x83 flags IOLOCK_EXCL caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-6489    [007] .....  2528.857556: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-6489    [007] .....  2528.857558: xfs_file_direct_write: dev 7:0 ino 0x83 disize 0xd000 pos 0xd000 bytecount 0x200
          xfs_io-6489    [007] .....  2528.857558: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-6489    [007] .....  2528.857559: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-6489    [007] .....  2528.857572: xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0xd000 pos 0xd000 bytecount 0x200 fork data startoff 0xd startblock 0x35 fsbcount 0x1
          xfs_io-6489    [007] .....  2528.857572: iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0x35000 offset 0xd000 length 0x1000 type UNWRITTEN (0x3) flags NEW|DIRTY (0x3)
          xfs_io-6489    [007] .....  2528.857573: block_bio_queue: 7,0 WS 424 + 1 [xfs_io]
          xfs_io-6489    [007] .....  2528.857575: block_bio_queue: 7,0 WS 425 + 7 [xfs_io]
          xfs_io-6489    [007] .....  2528.857607: xfs_end_io_direct_write: dev 7:0 ino 0x83 isize 0xd000 disize 0xd000 pos 0xd000 bytecount 0x200
          xfs_io-6489    [007] .....  2528.857607: xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0xd000 disize 0xd000 pos 0xd000 bytecount 0x200
          xfs_io-6489    [007] .....  2528.857607: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-6489    [007] .....  2528.857611: iomap_dio_complete: dev 7:0 ino 0x83 size 0xd200 offset 0xd200 flags DIRECT aio 0 error 0 ret 512
```

## EOF에 닿는 512 byte direct write 뒤의 extent와 on-disk 값

**관찰 출력**

```text
== s3 subblock extending 512 at 53248
/mnt/dio19a/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..111]:        320..431          0 (320..431)         112 000000
stat.size = 53760
core.size = 53760
core.nblocks = 14
core.nextents = 1
u3.bmx[0] = [startoff,startblock,blockcount,extentflag] 
0:[0,40,14,0]
```

## unwritten extent에 512 byte direct write: tracepoint

**관찰 출력**

```text
              sh-6495    [006] ...1.  2528.863840: tracing_mark_write: s5-subblock-unwritten
          xfs_io-6496    [000] .....  2528.864760: xfs_ilock: dev 7:0 ino 0x84 flags IOLOCK_SHARED caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-6496    [000] .....  2528.864761: xfs_ilock: dev 7:0 ino 0x84 flags ILOCK_EXCL caller xfs_vn_update_time+0xbe/0x1d0
          xfs_io-6496    [000] .....  2528.864763: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x200
          xfs_io-6496    [000] .....  2528.864764: xfs_ilock: dev 7:0 ino 0x84 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-6496    [000] .....  2528.864764: xfs_end_io_direct_write: dev 7:0 ino 0x84 isize 0x10000 disize 0x10000 pos 0x1000 bytecount 0x0
          xfs_io-6496    [000] .....  2528.864764: iomap_dio_complete: dev 7:0 ino 0x84 size 0x10000 offset 0x1000 flags DIRECT aio 0 error -11 ret -11
          xfs_io-6496    [000] .....  2528.864765: xfs_ilock: dev 7:0 ino 0x84 flags IOLOCK_EXCL caller xfs_ilock_iocb_for_write+0x50/0xd0
          xfs_io-6496    [000] .....  2528.864765: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x200
          xfs_io-6496    [000] .....  2528.864765: xfs_ilock: dev 7:0 ino 0x84 flags ILOCK_SHARED caller xfs_ilock_for_iomap+0x2f/0x50
          xfs_io-6496    [000] .....  2528.864765: xfs_iomap_found: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x200 fork data startoff 0x1 startblock 0x19 fsbcount 0x1
          xfs_io-6496    [000] .....  2528.864765: iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x19000 offset 0x1000 length 0x1000 type UNWRITTEN (0x3) flags DIRTY (0x2)
          xfs_io-6496    [000] .....  2528.864767: block_bio_queue: 7,0 WS 200 + 1 [xfs_io]
          xfs_io-6496    [000] .....  2528.864768: block_bio_queue: 7,0 WS 201 + 7 [xfs_io]
          xfs_io-6496    [000] .....  2528.864789: xfs_end_io_direct_write: dev 7:0 ino 0x84 isize 0x10000 disize 0x10000 pos 0x1000 bytecount 0x200
          xfs_io-6496    [000] .....  2528.864789: xfs_unwritten_convert: dev 7:0 ino 0x84 isize 0x10000 disize 0x10000 pos 0x1000 bytecount 0x200
          xfs_io-6496    [000] .....  2528.864789: xfs_ilock: dev 7:0 ino 0x84 flags ILOCK_EXCL caller xfs_trans_alloc_inode+0xa1/0x190
          xfs_io-6496    [000] .....  2528.864797: iomap_dio_complete: dev 7:0 ino 0x84 size 0x10000 offset 0x1200 flags DIRECT aio 0 error 0 ret 512
```

## unwritten extent에 512 byte direct write 뒤의 extent와 on-disk 값

**관찰 출력**

```text
== s5 subblock into unwritten
/mnt/dio19a/f2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..7]:          192..199          0 (192..199)           8 010000
   1: [8..15]:         200..207          0 (200..207)           8 000000
   2: [16..127]:       208..319          0 (208..319)         112 010000
core.size = 65536
core.nblocks = 16
core.nextents = 3
u3.bmx[0-2] = [startoff,startblock,blockcount,extentflag] 
0:[0,24,1,1] 
1:[1,25,1,0] 
2:[2,26,14,1]
```

## sector 정렬이 아닌 direct write

**관찰 출력**

```text
== s4 not sector aligned
pwrite: Invalid argument
              sh-6492    [004] ...1.  2528.860737: tracing_mark_write: s4-einval
          xfs_io-6493    [000] .....  2528.861807: xfs_ilock: dev 7:0 ino 0x83 flags ILOCK_SHARED caller xfs_can_free_eofblocks+0x131/0x1e0
              sh-6495    [006] ...1.  2528.863840: tracing_mark_write: s5-subblock-unwritten
```

## 4096 byte logical sector device의 direct I/O

**관찰 출력**

```text
getss=4096 lbs=4096 dma_alignment=511
         =                       sectsz=4096  attr=2, projid32bit=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
stat.mask = 0x23fff
stat.dio_mem_align = 512
stat.dio_offset_align = 4096
stat.dio_read_offset_align = 4096
fd.path = "/mnt/dio19d/f1"
dioattr.mem = 0x200
dioattr.miniosz = 4096
dioattr.maxiosz = 2147479552
== direct write 512 at 0
pwrite: Invalid argument
== direct write 4k at 0, read 512 and 4k
pread: Invalid argument
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (47.063 MiB/sec and 12048.1928 ops/sec)
read 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (651.042 MiB/sec and 166666.6667 ops/sec)
size=4096 blocks=8
```

## statx와 DIOINFO가 알리는 값

**관찰 출력**

```text
lbs=512 dma_alignment=511
== f1
stat.mask = 0x23fff
stat.dio_mem_align = 512
stat.dio_offset_align = 512
stat.dio_read_offset_align = 512
fd.path = "/mnt/dio19c/f1"
fsxattr.xflags = 0x80000000 [----------------X]
dioattr.mem = 0x200
dioattr.miniosz = 512
dioattr.maxiosz = 2147483136
== f2
stat.mask = 0x23fff
stat.dio_mem_align = 512
stat.dio_offset_align = 4096
stat.dio_read_offset_align = 512
fd.path = "/mnt/dio19c/f2"
fsxattr.xflags = 0x80000000 [----------------X]
dioattr.mem = 0x200
dioattr.miniosz = 4096
dioattr.maxiosz = 2147479552
== f3
stat.mask = 0x23fff
stat.dio_mem_align = 512
stat.dio_offset_align = 4096
stat.dio_read_offset_align = 512
fd.path = "/mnt/dio19c/f3"
fsxattr.xflags = 0x80000000 [----------------X]
dioattr.mem = 0x200
dioattr.miniosz = 4096
dioattr.maxiosz = 2147479552
/mnt/dio19c/f3:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        320..447          0 (320..447)         128 100000
```

## reflink한 file의 direct write와 read

**관찰 출력**

```text
== f1 direct write 512 at 512 (not reflinked)
wrote 512/512 bytes at offset 512
512.000000 bytes, 1 ops; 0.0000 sec (14.796 MiB/sec and 30303.0303 ops/sec)
== f3 direct write 512 at 512 (reflinked)
wrote 512/512 bytes at offset 512
512.000000 bytes, 1 ops; 0.0000 sec (18.084 MiB/sec and 37037.0370 ops/sec)
== f3 direct read 512 at 512
read 512/512 bytes at offset 512
              sh-6811    [001] ...1.  2732.020927: tracing_mark_write: w-f1-512
          xfs_io-6812    [003] .....  2732.021854: xfs_file_direct_write: dev 7:0 ino 0x83 disize 0x10000 pos 0x200 bytecount 0x200
              sh-6814    [006] ...1.  2732.023897: tracing_mark_write: w-f3-512
          xfs_io-6815    [002] .....  2732.024827: xfs_reflink_bounce_dio_write: dev 7:0 ino 0x85 disize 0x10000 pos 0x200 bytecount 0x200
          xfs_io-6815    [002] .....  2732.024830: xfs_file_buffered_write: dev 7:0 ino 0x85 disize 0x10000 pos 0x200 bytecount 0x200
              sh-6817    [005] ...1.  2732.026818: tracing_mark_write: r-f3-512
          xfs_io-6818    [002] .....  2732.027805: xfs_file_direct_read: dev 7:0 ino 0x85 disize 0x10000 pos 0x200 bytecount 0x200
```

## hole과 EOF를 지나는 direct read: 결과

**관찰 출력**

```text
/mnt/dio19b/h1 ino=131 size=102400
/mnt/dio19b/h2 ino=132 size=102401
/mnt/dio19b/h1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..103]:        192..295          0 (192..295)         104 000000
   1: [104..199]:      hole                                    96
== h1 direct read 200k (hole 52k..100k, EOF 100k)
read 102400/204800 bytes at offset 0
== h2 direct read 200k (EOF 102401)
read 102401/204800 bytes at offset 0
== h1 direct read at 128k (beyond EOF)
read 0/4096 bytes at offset 131072
```

## hole과 EOF를 지나는 direct read: tracepoint

**관찰 출력**

```text
              sh-6625    [003] ...1.  2538.196994: tracing_mark_write: r-h1-direct
          xfs_io-6626    [005] .....  2538.198284: xfs_file_direct_read: dev 7:0 ino 0x83 disize 0x19000 pos 0x0 bytecount 0x32000
          xfs_io-6626    [005] .....  2538.198296: xfs_iomap_found: dev 7:0 ino 0x83 disize 0x19000 pos 0x0 bytecount 0x32000 fork data startoff 0x0 startblock 0x18 fsbcount 0xd
          xfs_io-6626    [005] .....  2538.198297: iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0x18000 offset 0x0 length 0xd000 type MAPPED (0x2) flags DIRTY (0x2)
          xfs_io-6626    [005] .....  2538.198303: xfs_iomap_found: dev 7:0 ino 0x83 disize 0x19000 pos 0xd000 bytecount 0x25000 fork data startoff 0xd startblock 0xfffffffffffffffe fsbcount 0x25
          xfs_io-6626    [005] .....  2538.198303: iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0xffffffffffffffff offset 0xd000 length 0x25000 type HOLE (0x0) flags DIRTY (0x2)
          xfs_io-6626    [005] .....  2538.198348: iomap_dio_complete: dev 7:0 ino 0x83 size 0x19000 offset 0x19000 flags DIRECT aio 0 error 0 ret 102400
              sh-6628    [004] ...1.  2538.200786: tracing_mark_write: r-h2-direct
          xfs_io-6629    [005] .....  2538.201999: xfs_file_direct_read: dev 7:0 ino 0x84 disize 0x19001 pos 0x0 bytecount 0x32000
          xfs_io-6629    [005] .....  2538.202001: xfs_iomap_found: dev 7:0 ino 0x84 disize 0x19001 pos 0x0 bytecount 0x32000 fork data startoff 0x0 startblock 0x25 fsbcount 0xd
          xfs_io-6629    [005] .....  2538.202002: iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x25000 offset 0x0 length 0xd000 type MAPPED (0x2) flags DIRTY (0x2)
          xfs_io-6629    [005] .....  2538.202005: xfs_iomap_found: dev 7:0 ino 0x84 disize 0x19001 pos 0xd000 bytecount 0x25000 fork data startoff 0xd startblock 0xfffffffffffffffe fsbcount 0x25
          xfs_io-6629    [005] .....  2538.202005: iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xffffffffffffffff offset 0xd000 length 0x25000 type HOLE (0x0) flags DIRTY (0x2)
          xfs_io-6629    [005] .....  2538.202045: iomap_dio_complete: dev 7:0 ino 0x84 size 0x19001 offset 0x19001 flags DIRECT aio 0 error 0 ret 102401
              sh-6631    [003] ...1.  2538.204186: tracing_mark_write: r-h1-beyond-eof
          xfs_io-6632    [006] .....  2538.205304: xfs_file_direct_read: dev 7:0 ino 0x83 disize 0x19000 pos 0x20000 bytecount 0x1000
```

## unwritten 구간을 지나는 read: extent와 결과

**관찰 출력**

```text
/mnt/dio19b/u1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..103]:        400..503          0 (400..503)         104 000000
   1: [104..199]:      504..599          0 (504..599)          96 010000
== u1 direct read 200k (unwritten 52k..100k)
read 102400/204800 bytes at offset 0
== u1 buffered read 200k
read 102400/204800 bytes at offset 0
```

## unwritten 구간을 지나는 direct read와 buffered read: tracepoint

**관찰 출력**

```text
              sh-6634    [001] ...1.  2538.207447: tracing_mark_write: r-u1-direct
          xfs_io-6635    [007] .....  2538.208502: xfs_file_direct_read: dev 7:0 ino 0x85 disize 0x19000 pos 0x0 bytecount 0x32000
          xfs_io-6635    [007] .....  2538.208511: xfs_iomap_found: dev 7:0 ino 0x85 disize 0x19000 pos 0x0 bytecount 0x32000 fork data startoff 0x0 startblock 0x32 fsbcount 0xd
          xfs_io-6635    [007] .....  2538.208512: iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x32000 offset 0x0 length 0xd000 type MAPPED (0x2) flags DIRTY (0x2)
          xfs_io-6635    [007] .....  2538.208520: xfs_iomap_found: dev 7:0 ino 0x85 disize 0x19000 pos 0xd000 bytecount 0x25000 fork data startoff 0xd startblock 0x3f fsbcount 0xc
          xfs_io-6635    [007] .....  2538.208520: iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x3f000 offset 0xd000 length 0xc000 type UNWRITTEN (0x3) flags DIRTY (0x2)
          xfs_io-6635    [007] .....  2538.208520: xfs_iomap_found: dev 7:0 ino 0x85 disize 0x19000 pos 0x19000 bytecount 0x19000 fork data startoff 0x19 startblock 0xfffffffffffffffe fsbcount 0x19
          xfs_io-6635    [007] .....  2538.208521: iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0xffffffffffffffff offset 0x19000 length 0x19000 type HOLE (0x0) flags DIRTY (0x2)
          xfs_io-6635    [007] .....  2538.208551: iomap_dio_complete: dev 7:0 ino 0x85 size 0x19000 offset 0x19000 flags DIRECT aio 0 error 0 ret 102400
              sh-6637    [003] ...1.  2538.210666: tracing_mark_write: r-u1-buffered
          xfs_io-6638    [004] .....  2538.211722: xfs_file_buffered_read: dev 7:0 ino 0x85 disize 0x19000 pos 0x0 bytecount 0x32000
          xfs_io-6638    [004] .....  2538.211729: iomap_readahead: dev 7:0 ino 0x85 nr_pages 25
          xfs_io-6638    [004] .....  2538.211729: xfs_iomap_found: dev 7:0 ino 0x85 disize 0x19000 pos 0x0 bytecount 0x19000 fork data startoff 0x0 startblock 0x32 fsbcount 0xd
          xfs_io-6638    [004] .....  2538.211730: iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x32000 offset 0x0 length 0xd000 type MAPPED (0x2) flags DIRTY (0x2)
          xfs_io-6638    [004] .....  2538.211732: xfs_iomap_found: dev 7:0 ino 0x85 disize 0x19000 pos 0xd000 bytecount 0xc000 fork data startoff 0xd startblock 0x3f fsbcount 0xc
          xfs_io-6638    [004] .....  2538.211733: iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x3f000 offset 0xd000 length 0xc000 type UNWRITTEN (0x3) flags DIRTY (0x2)
```

## hole, unwritten, written 구간의 내용과 on-disk extent

**관찰 출력**

```text
== contents: h1 hole at 52k, u1 unwritten at 64k, u1 written at 0
0000d000:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
0000d010:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00010000:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00010010:  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ................
00000000:  cd cd cd cd cd cd cd cd cd cd cd cd cd cd cd cd  ................
00000010:  cd cd cd cd cd cd cd cd cd cd cd cd cd cd cd cd  ................
core.size = 102400
core.nblocks = 25
core.nextents = 2
u3.bmx[0-1] = [startoff,startblock,blockcount,extentflag] 
0:[0,50,13,0] 
1:[13,63,12,1]
```

