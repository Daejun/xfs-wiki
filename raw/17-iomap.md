# iomap: 원본 출력

> 본문은 ../17-iomap.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## buf-write: 한 바퀴, 1 MiB file의 writeback

**관찰 출력**

```text
== buf-write: lines=4
xfs_io iomap_iter: dev 7:0 ino 0x83 pos 0x0 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x100000 fork data startoff 0x0 startblock 0xffffffffe0008 fsbcount 0x100
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0xffffffffffffffff offset 0x0 length 0x100000 type DELALLOC (0x1) flags NEW|DIRTY (0x3)
xfs_io iomap_iter: dev 7:0 ino 0x83 pos 0x100000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
```

## writeback: 1 MiB file의 writeback, 완료 도식

**관찰 출력**

```text
== writeback: lines=4
xfs_io iomap_writeback_folio: dev 7:0 ino 0x83 size 0x100000 offset 0x0 length 0x100000
xfs_io xfs_map_blocks_alloc: dev 7:0 ino 0x83 disize 0x0 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0008 fsbcount 0x100
xfs_io iomap_add_to_ioend: dev 7:0 ino 0x83 bdev 7:0 pos 0x0 dirty len 0x100000 addr 0x18000 offset 0x0 length 0x100000 type UNWRITTEN (0x3) flags DIRTY (0x2)
kworker/0:2 xfs_unwritten_convert: dev 7:0 ino 0x83 isize 0x100000 disize 0x0 pos 0x0 bytecount 0x100000
```

## multi: 256 KiB loop 도식

**관찰 출력**

```text
== multi: lines=16
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x30000 pos 0x0 bytecount 0x40000 fork data startoff 0x0 startblock 0xd fsbcount 0x3
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x3000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x30000 pos 0x3000 bytecount 0x3d000 fork data startoff 0x3 startblock 0x118 fsbcount 0xd
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x118000 offset 0x3000 length 0xd000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x10000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_alloc: dev 7:0 ino 0x84 disize 0x30000 pos 0x10000 bytecount 0x30000 fork data startoff 0x10 startblock 0xffffffffe0007 fsbcount 0x10
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xffffffffffffffff offset 0x10000 length 0x10000 type DELALLOC (0x1) flags NEW (0x1)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x20000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x30000 pos 0x20000 bytecount 0x20000 fork data startoff 0x20 startblock 0x135 fsbcount 0x10
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x135000 offset 0x20000 length 0x10000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x30000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
xfs_io xfs_iomap_alloc: dev 7:0 ino 0x84 disize 0x30000 pos 0x30000 bytecount 0x10000 fork data startoff 0x30 startblock 0xffffffffe0007 fsbcount 0x20
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xffffffffffffffff offset 0x30000 length 0x20000 type DELALLOC (0x1) flags NEW (0x1)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x40000 length 0x0 status 0 flags WRITE (0x1) ops xfs_buffered_write_iomap_ops caller iomap_file_buffered_write
```

## dio-new, dio-over: direct write의 새로 쓰기와 덮어쓰기

**관찰 출력**

```text
== dio-new: lines=8
xfs_io iomap_dio_rw_begin: dev 7:0 ino 0x85 size 0x0 offset 0x0 length 0x100000 done_before 0x0 flags DIRECT dio_flags  aio 0
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x0 length 0x0 status 0 flags WRITE|DIRECT (0x11) ops xfs_direct_write_iomap_ops caller __iomap_dio_rw
xfs_io xfs_iomap_alloc: dev 7:0 ino 0x85 disize 0x0 pos 0x0 bytecount 0x100000 fork data startoff 0x0 startblock 0x155 fsbcount 0x100
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x155000 offset 0x0 length 0x100000 type UNWRITTEN (0x3) flags NEW|DIRTY (0x3)
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x100000 length 0x0 status 0 flags WRITE|DIRECT (0x11) ops xfs_direct_write_iomap_ops caller __iomap_dio_rw
xfs_io xfs_end_io_direct_write: dev 7:0 ino 0x85 isize 0x0 disize 0x0 pos 0x0 bytecount 0x100000
xfs_io xfs_unwritten_convert: dev 7:0 ino 0x85 isize 0x0 disize 0x0 pos 0x0 bytecount 0x100000
xfs_io iomap_dio_complete: dev 7:0 ino 0x85 size 0x100000 offset 0x100000 flags DIRECT aio 0 error 0 ret 1048576
== dio-over: lines=7
xfs_io iomap_dio_rw_begin: dev 7:0 ino 0x85 size 0x100000 offset 0x0 length 0x100000 done_before 0x0 flags DIRECT dio_flags  aio 0
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x0 length 0x0 status 0 flags WRITE|DIRECT (0x11) ops xfs_direct_write_iomap_ops caller __iomap_dio_rw
xfs_io xfs_iomap_found: dev 7:0 ino 0x85 disize 0x100000 pos 0x0 bytecount 0x100000 fork data startoff 0x0 startblock 0x155 fsbcount 0x100
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x155000 offset 0x0 length 0x100000 type MAPPED (0x2) flags DIRTY (0x2)
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x100000 length 0x0 status 0 flags WRITE|DIRECT (0x11) ops xfs_direct_write_iomap_ops caller __iomap_dio_rw
xfs_io xfs_end_io_direct_write: dev 7:0 ino 0x85 isize 0x100000 disize 0x100000 pos 0x0 bytecount 0x100000
xfs_io iomap_dio_complete: dev 7:0 ino 0x85 size 0x100000 offset 0x100000 flags DIRECT aio 0 error 0 ret 1048576
```

## buf-read, dio-read: 진입점의 읽기, 1 MiB file의 writeback 표 마지막 줄

**관찰 출력**

```text
== buf-read: lines=4
xfs_io iomap_iter: dev 7:0 ino 0x83 pos 0x0 length 0x0 status 0 flags  (0x0) ops xfs_read_iomap_ops caller iomap_readahead
xfs_io xfs_iomap_found: dev 7:0 ino 0x83 disize 0x100000 pos 0x0 bytecount 0x100000 fork data startoff 0x0 startblock 0x18 fsbcount 0x100
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x83 bdev 7:0 addr 0x18000 offset 0x0 length 0x100000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x83 pos 0x100000 length 0x0 status 0 flags  (0x0) ops xfs_read_iomap_ops caller iomap_readahead
== dio-read: lines=6
xfs_io iomap_dio_rw_begin: dev 7:0 ino 0x85 size 0x100000 offset 0x0 length 0x100000 done_before 0x0 flags DIRECT dio_flags  aio 0
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x0 length 0x0 status 0 flags DIRECT (0x10) ops xfs_read_iomap_ops caller __iomap_dio_rw
xfs_io xfs_iomap_found: dev 7:0 ino 0x85 disize 0x100000 pos 0x0 bytecount 0x100000 fork data startoff 0x0 startblock 0x155 fsbcount 0x100
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x85 bdev 7:0 addr 0x155000 offset 0x0 length 0x100000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x85 pos 0x100000 length 0x0 status 0 flags DIRECT (0x10) ops xfs_read_iomap_ops caller __iomap_dio_rw
xfs_io iomap_dio_complete: dev 7:0 ino 0x85 size 0x100000 offset 0x100000 flags DIRECT aio 0 error 0 ret 1048576
```

## report: 진입점의 FIEMAP과 seek

**관찰 출력**

```text
/mnt/p17io/b:
 EXT: FILE-OFFSET      BLOCK-RANGE      TOTAL FLAGS
   0: [0..23]:         104..127            24   0x0
   1: [24..511]:       2240..2727         488   0x1
Whence	Result
DATA	0
HOLE	262144
== report: lines=23
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x0 status 0 flags REPORT (0x4) ops xfs_read_iomap_ops caller iomap_fiemap
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x40000 pos 0x0 bytecount 0x7fffffffffffffff fork data startoff 0x0 startblock 0xd fsbcount 0x3
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x3000 length 0x0 status 0 flags REPORT (0x4) ops xfs_read_iomap_ops caller iomap_fiemap
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x40000 pos 0x3000 bytecount 0x7fffffffffffcfff fork data startoff 0x3 startblock 0x118 fsbcount 0x3d
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x118000 offset 0x3000 length 0x3d000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x40000 length 0x0 status 0 flags REPORT (0x4) ops xfs_read_iomap_ops caller iomap_fiemap
xfs_io xfs_iomap_found: dev 7:0 ino 0x84 disize 0x40000 pos 0x40000 bytecount 0x7ffffffffffbffff fork data startoff 0x40 startblock 0xfffffffffffffffe fsbcount 0x7ffffffffffc0
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xffffffffffffffff offset 0x40000 length 0x7ffffffffffc0000 type HOLE (0x0) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x7fffffffffffffff length 0x0 status 0 flags REPORT (0x4) ops xfs_read_iomap_ops caller iomap_fiemap
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x3000 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x118000 offset 0x3000 length 0x3d000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x40000 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_data
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x3000 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_data
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x0 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0xd000 offset 0x0 length 0x3000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x3000 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x84 bdev 7:0 addr 0x118000 offset 0x3000 length 0x3d000 type MAPPED (0x2) flags  (0x0)
xfs_io iomap_iter: dev 7:0 ino 0x84 pos 0x40000 length 0x0 status 0 flags REPORT (0x4) ops xfs_seek_iomap_ops caller iomap_seek_hole
```

## fault: mmap write fault

**관찰 출력**

```text
== fault: lines=12
xfs_io iomap_iter: dev 7:0 ino 0x86 pos 0x0 length 0x0 status 0 flags  (0x0) ops xfs_read_iomap_ops caller iomap_readahead
xfs_io xfs_iomap_found: dev 7:0 ino 0x86 disize 0x10000 pos 0x0 bytecount 0x10000 fork data startoff 0x0 startblock 0xfffffffffffffffe fsbcount 0x10
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x86 bdev 7:0 addr 0xffffffffffffffff offset 0x0 length 0x10000 type HOLE (0x0) flags DIRTY (0x2)
xfs_io iomap_iter: dev 7:0 ino 0x86 pos 0x10000 length 0x0 status 0 flags  (0x0) ops xfs_read_iomap_ops caller iomap_readahead
xfs_io iomap_iter: dev 7:0 ino 0x86 pos 0x0 length 0x0 status 0 flags WRITE|FAULT (0x9) ops xfs_buffered_write_iomap_ops caller iomap_page_mkwrite
xfs_io xfs_iomap_alloc: dev 7:0 ino 0x86 disize 0x10000 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
xfs_io iomap_iter_dstmap: dev 7:0 ino 0x86 bdev 7:0 addr 0xffffffffffffffff offset 0x0 length 0x1000 type DELALLOC (0x1) flags NEW|DIRTY (0x3)
xfs_io iomap_iter: dev 7:0 ino 0x86 pos 0x1000 length 0x0 status 0 flags WRITE|FAULT (0x9) ops xfs_buffered_write_iomap_ops caller iomap_page_mkwrite
xfs_io iomap_writeback_folio: dev 7:0 ino 0x86 size 0x10000 offset 0x0 length 0x1000
xfs_io xfs_map_blocks_alloc: dev 7:0 ino 0x86 disize 0x10000 pos 0x0 bytecount 0x1000 fork data startoff 0x0 startblock 0xffffffffe0007 fsbcount 0x1
xfs_io iomap_add_to_ioend: dev 7:0 ino 0x86 bdev 7:0 pos 0x0 dirty len 0x1000 addr 0x255000 offset 0x0 length 0x1000 type UNWRITTEN (0x3) flags DIRTY (0x2)
kworker/2:1 xfs_unwritten_convert: dev 7:0 ino 0x86 isize 0x10000 disize 0x10000 pos 0x0 bytecount 0x1000
```

