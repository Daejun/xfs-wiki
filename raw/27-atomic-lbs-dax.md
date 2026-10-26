# atomic write, large block size, DAX: 원본 출력

> 본문은 ../27-atomic-lbs-dax.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 관찰: atomic write 한계값과 max_atomic_write mount option

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
/sys/block/loop0/queue/atomic_write_boundary_bytes:0
/sys/block/loop0/queue/atomic_write_max_bytes:0
/sys/block/loop0/queue/atomic_write_unit_max_bytes:0
/sys/block/loop0/queue/atomic_write_unit_min_bytes:0
== bdev /dev/loop0
stat.mask = 0x17ff
stat.attributes = 0x0
stat.attributes_mask = 0x203070
stat.atomic_write_unit_min = 0
stat.atomic_write_unit_max = 0
stat.atomic_write_segments_max = 0
stat.atomic_write_unit_max_opt = 0
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       reflink=0    bigtime=1 inobtcount=1 nrext64=1
== A default mount
/dev/loop0 /mnt/u27a xfs rw,relatime,inode64,logbufs=8,logbsize=32k,noquota 0 0
stat.mask = 0x1173f
stat.attributes = 0x400000
stat.attributes_mask = 0x603070
stat.atomic_write_unit_min = 4096
stat.atomic_write_unit_max = 2097152
stat.atomic_write_segments_max = 1
stat.atomic_write_unit_max_opt = 0
== B max_atomic_write=64k
/dev/loop0 /mnt/u27a xfs rw,relatime,inode64,logbufs=8,logbsize=32k,noquota,max_atomic_write=64k 0 0
stat.atomic_write_unit_min = 4096
stat.atomic_write_unit_max = 65536
stat.atomic_write_segments_max = 1
stat.atomic_write_unit_max_opt = 0
== C max_atomic_write=12k
mount: /mnt/u27a: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
C-rc=32
== D reflink=0
stat.attributes = 0x0
stat.attributes_mask = 0x603070
stat.atomic_write_unit_min = 0
stat.atomic_write_unit_max = 0
stat.atomic_write_segments_max = 0
stat.atomic_write_unit_max_opt = 0
pwrite: Operation not supported
D-pwrite-rc=1
== E reflink=0 max_atomic_write=64k
mount: /mnt/u27a: wrong fs type, bad option, bad superblock on /dev/loop1, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
E-rc=32
[  122.737164] XFS (loop0): max atomic write size of 12288 bytes is not a power of 2
[  122.756730] XFS (loop1): cannot support atomic writes of size 64k with no reflink support
              sh-1533    [005] ...1.   122.731598: tracing_mark_write: A-default
           mount-1535    [004] .....   122.735733: xfs_calc_max_atomic_write_fsblocks: dev 7:0 per_intent 208 step_size 56832 logres 226176 blockcount 814
           mount-1535    [004] .....   122.735734: xfs_calc_atomic_write_unit_max: dev 7:0 ag max_write 262144 max_ioend 512 max_gsize 65536 awu_max 512
           mount-1535    [004] .....   122.735734: xfs_calc_max_atomic_write_fsblocks: dev 7:0 per_intent 208 step_size 56832 logres 226176 blockcount 814
           mount-1535    [004] .....   122.735734: xfs_calc_atomic_write_unit_max: dev 7:0 rtg max_write 262144 max_ioend 512 max_gsize 0 awu_max 0
              sh-1553    [002] ...1.   122.744464: tracing_mark_write: B-64k
           mount-1555    [002] .....   122.748028: xfs_calc_max_atomic_write_log_geometry: dev 7:0 per_intent 208 step_size 56832 blockcount 16 min_logblocks 1985 logblocks 16384 logres 60160
           mount-1555    [002] .....   122.748029: xfs_calc_max_atomic_write_fsblocks: dev 7:0 per_intent 208 step_size 56832 logres 60160 blockcount 16
           mount-1555    [002] .....   122.748029: xfs_calc_atomic_write_unit_max: dev 7:0 ag max_write 262144 max_ioend 16 max_gsize 65536 awu_max 16
           mount-1555    [002] .....   122.748029: xfs_calc_max_atomic_write_fsblocks: dev 7:0 per_intent 208 step_size 56832 logres 60160 blockcount 16
           mount-1555    [002] .....   122.748029: xfs_calc_atomic_write_unit_max: dev 7:0 rtg max_write 262144 max_ioend 16 max_gsize 0 awu_max 0
              sh-1570    [004] ...1.   122.754093: tracing_mark_write: C-12k
              sh-1582    [000] ...1.   122.761121: tracing_mark_write: D-noreflink
           mount-1584    [003] .....   122.765596: xfs_calc_atomic_write_unit_max: dev 7:1 ag max_write 262144 max_ioend 0 max_gsize 65536 awu_max 0
           mount-1584    [003] .....   122.765597: xfs_calc_atomic_write_unit_max: dev 7:1 rtg max_write 262144 max_ioend 0 max_gsize 0 awu_max 0
              sh-1602    [002] ...1.   122.774835: tracing_mark_write: E-noreflink-64k
```

## 관찰: software atomic write의 bmap 이동과 거절

**관찰 출력**

```text
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
awu_max=2097152
== f1 before
/mnt/u27c/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..20479]:      192..20671        0 (192..20671)     20480 000000
== A one 64k atomic write at 0
wrote 65536/65536 bytes at offset 0
64 KiB, 1 ops; 0.0001 sec (393.082 MiB/sec and 6289.3082 ops/sec)
/mnt/u27c/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        20736..20863      0 (20736..20863)     128 000000
   1: [128..20479]:    320..20671        0 (320..20671)     20352 000000
== B 4k atomic write at 128k
wrote 4096/4096 bytes at offset 131072
4 KiB, 1 ops; 0.0000 sec (39.860 MiB/sec and 10204.0816 ops/sec)
/mnt/u27c/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        20736..20863      0 (20736..20863)     128 000000
   1: [128..255]:      320..447          0 (320..447)         128 000000
   2: [256..263]:      20992..20999      0 (20992..20999)       8 000000
   3: [264..20479]:    456..20671        0 (456..20671)     20216 000000
== C reflink f1 to f2, 64k atomic write at 0 on f2
wrote 65536/65536 bytes at offset 0
64 KiB, 1 ops; 0.0001 sec (529.661 MiB/sec and 8474.5763 ops/sec)
/mnt/u27c/f1:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        20736..20863      0 (20736..20863)     128 000000
   1: [128..255]:      320..447          0 (320..447)         128 100000
   2: [256..263]:      20992..20999      0 (20992..20999)       8 100000
   3: [264..20479]:    456..20671        0 (456..20671)     20216 100000
/mnt/u27c/f2:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL FLAGS
   0: [0..127]:        21248..21375      0 (21248..21375)     128 000000
   1: [128..255]:      320..447          0 (320..447)         128 100000
   2: [256..263]:      20992..20999      0 (20992..20999)       8 100000
   3: [264..20479]:    456..20671        0 (456..20671)     20216 100000
== D1 buffered atomic 4096
pwrite: Operation not supported
D1-rc=1
== D2 length 12288
pwrite: Invalid argument
D2-rc=1
== D3 pos 4096 length 8192
pwrite: Invalid argument
D3-rc=1
== D4 length 4194304
pwrite: Invalid argument
D4-rc=1
```

## 관찰: software atomic write의 trace 순서

**관찰 출력**

```text
              sh-1981    [001] ...1.   131.758392: tracing_mark_write: A-atomic-64k-at-0
          xfs_io-1982    [001] .....   131.759282: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0xa00000 pos 0x0 bytecount 0x10000
          xfs_io-1982    [001] .....   131.759284: xfs_iomap_atomic_write_cow: dev 7:0 ino 0x84 pos 0x0 bytecount 0x10000
          xfs_io-1982    [001] .....   131.759344: xfs_iomap_found: dev 7:0 ino 0x84 disize 0xa00000 pos 0x0 bytecount 0x10000 fork cow startoff 0x0 startblock 0xa20 fsbcount 0x10
          xfs_io-1982    [001] .....   131.759366: xfs_reflink_end_cow: dev 7:0 ino 0x84 isize 0xa00000 disize 0xa00000 pos 0x0 bytecount 0x10000
              sh-1985    [004] ...1.   131.761990: tracing_mark_write: B-atomic-4k-at-128k
          xfs_io-1986    [002] .....   131.762866: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0xa00000 pos 0x20000 bytecount 0x1000
          xfs_io-1986    [002] .....   131.762867: xfs_iomap_atomic_write_cow: dev 7:0 ino 0x84 pos 0x20000 bytecount 0x1000
          xfs_io-1986    [002] .....   131.762883: xfs_iomap_found: dev 7:0 ino 0x84 disize 0xa00000 pos 0x20000 bytecount 0x1000 fork cow startoff 0x20 startblock 0xa40 fsbcount 0x1
          xfs_io-1986    [002] .....   131.762904: xfs_reflink_end_cow: dev 7:0 ino 0x84 isize 0xa00000 disize 0xa00000 pos 0x20000 bytecount 0x1000
              sh-1990    [005] ...1.   131.765978: tracing_mark_write: C-atomic-64k-on-reflinked-f2
          xfs_io-1991    [004] .....   131.767045: xfs_file_direct_write: dev 7:0 ino 0x85 disize 0xa00000 pos 0x0 bytecount 0x10000
          xfs_io-1991    [004] .....   131.767046: xfs_iomap_atomic_write_cow: dev 7:0 ino 0x85 pos 0x0 bytecount 0x10000
          xfs_io-1991    [004] .....   131.767066: xfs_iomap_found: dev 7:0 ino 0x85 disize 0xa00000 pos 0x0 bytecount 0x10000 fork cow startoff 0x0 startblock 0xa60 fsbcount 0x10
          xfs_io-1991    [004] .....   131.767103: xfs_reflink_end_cow: dev 7:0 ino 0x85 isize 0xa00000 disize 0xa00000 pos 0x0 bytecount 0x10000
              sh-1995    [004] ...1.   131.770748: tracing_mark_write: D-rejects
```

## 관찰: 16 KiB block의 mount, page cache, direct I/O, atomic write

**관찰 출력**

```text
==== block size 16384
data     =                       bsize=16384  blocks=65536, imaxpct=25
naming   =version 2              bsize=16384  ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=16384  blocks=5221, version=2
[  214.582211] XFS (loop0): Mounting V5 Filesystem 5445687e-02d5-4c27-ae13-18f2ea442c46
[  214.582874] XFS (loop0): Ending clean mount
== metadata buffer backing at mount (count, kind, bbcount)
      3 xfs_buf_backing_folio bbcount 0x20
     19 xfs_buf_backing_kmem bbcount 0x1
== buffered 4k at 4k in a 64k file, then mincore
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (390.625 MiB/sec and 100000.0000 ops/sec)
0x7f66c2dad000 - 0x7f66c2db1000  4 pages (0 : 16384)
/mnt/u27l/w size=65536 blocks=32
stat.atomic_write_unit_min = 16384
stat.atomic_write_unit_max = 33554432
== 4k direct write at 4k, plain file
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (139.509 MiB/sec and 35714.2857 ops/sec)
== 4k direct write at 4k, reflinked file
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (134.698 MiB/sec and 34482.7586 ops/sec)
== atomic 4k and atomic 16384
pwrite: Invalid argument
atomic4k-rc=1
wrote 16384/16384 bytes at offset 0
16 KiB, 1 ops; 0.0001 sec (154.703 MiB/sec and 9900.9901 ops/sec)
          xfs_io-3855    [007] .....   214.645277: xfs_file_buffered_write: dev 7:0 ino 0x1c3 disize 0x10000 pos 0x1000 bytecount 0x1000
          xfs_io-3860    [007] .....   214.648082: xfs_file_buffered_write: dev 7:0 ino 0x1c4 disize 0x0 pos 0x0 bytecount 0x10000
          xfs_io-3861    [004] .....   214.649001: xfs_file_buffered_write: dev 7:0 ino 0x1c5 disize 0x0 pos 0x0 bytecount 0x10000
              sh-3864    [007] ...1.   214.651755: tracing_mark_write: dio-plain-16384
          xfs_io-3865    [002] .....   214.652678: xfs_file_direct_write: dev 7:0 ino 0x1c4 disize 0x10000 pos 0x1000 bytecount 0x1000
              sh-3867    [001] ...1.   214.654571: tracing_mark_write: dio-reflinked-16384
          xfs_io-3868    [005] .....   214.655448: xfs_reflink_bounce_dio_write: dev 7:0 ino 0x1c6 disize 0x10000 pos 0x1000 bytecount 0x1000
          xfs_io-3868    [005] .....   214.655450: xfs_file_buffered_write: dev 7:0 ino 0x1c6 disize 0x10000 pos 0x1000 bytecount 0x1000
              sh-3870    [006] ...1.   214.657438: tracing_mark_write: atomic-16384
          xfs_io-3872    [004] .....   214.659130: xfs_file_direct_write: dev 7:0 ino 0x1c4 disize 0x10000 pos 0x0 bytecount 0x4000
blocksize = 16384
```

## 관찰: 4 KiB block의 mount, page cache, direct I/O, atomic write

**관찰 출력**

```text
==== block size 4096
data     =                       bsize=4096   blocks=262144, imaxpct=25
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
[  214.680926] XFS (loop0): Mounting V5 Filesystem 7a5c559e-fe62-4ceb-a18c-4f6f32cf9429
[  214.681491] XFS (loop0): Ending clean mount
== metadata buffer backing at mount (count, kind, bbcount)
      2 xfs_buf_backing_folio bbcount 0x20
      1 xfs_buf_backing_folio bbcount 0x8
     19 xfs_buf_backing_kmem bbcount 0x1
== buffered 4k at 4k in a 64k file, then mincore
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (434.028 MiB/sec and 111111.1111 ops/sec)
0x7f5482404000 - 0x7f5482405000  1 pages (4096 : 4096)
/mnt/u27l/w size=65536 blocks=8
stat.atomic_write_unit_min = 4096
stat.atomic_write_unit_max = 2097152
== 4k direct write at 4k, plain file
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (84.918 MiB/sec and 21739.1304 ops/sec)
== 4k direct write at 4k, reflinked file
wrote 4096/4096 bytes at offset 4096
4 KiB, 1 ops; 0.0000 sec (52.083 MiB/sec and 13333.3333 ops/sec)
== atomic 4k and atomic 4096
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0001 sec (35.191 MiB/sec and 9009.0090 ops/sec)
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (42.926 MiB/sec and 10989.0110 ops/sec)
          xfs_io-3919    [004] .....   214.735556: xfs_file_buffered_write: dev 7:0 ino 0x83 disize 0x10000 pos 0x1000 bytecount 0x1000
          xfs_io-3924    [007] .....   214.738624: xfs_file_buffered_write: dev 7:0 ino 0x84 disize 0x0 pos 0x0 bytecount 0x10000
          xfs_io-3925    [004] .....   214.739589: xfs_file_buffered_write: dev 7:0 ino 0x85 disize 0x0 pos 0x0 bytecount 0x10000
              sh-3928    [001] ...1.   214.743125: tracing_mark_write: dio-plain-4096
          xfs_io-3929    [005] .....   214.744179: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x1000 bytecount 0x1000
              sh-3931    [001] ...1.   214.746571: tracing_mark_write: dio-reflinked-4096
          xfs_io-3932    [002] .....   214.747465: xfs_file_direct_write: dev 7:0 ino 0x86 disize 0x10000 pos 0x1000 bytecount 0x1000
              sh-3934    [003] ...1.   214.749780: tracing_mark_write: atomic-4096
          xfs_io-3935    [002] .....   214.750715: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x0 bytecount 0x1000
          xfs_io-3936    [004] .....   214.751582: xfs_file_direct_write: dev 7:0 ino 0x84 disize 0x10000 pos 0x0 bytecount 0x1000
blocksize = 4096
```

## 관찰: DAX를 못 켜는 kernel의 mount option, chattr, MAP_SYNC

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
no /sys/bus/dax
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
== mount -o dax
mount: /mnt/u27d: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
mount-dax-rc=32
== mount -o dax=always
mount: /mnt/u27d: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
mount-dax=always-rc=32
== mount -o dax=inode
mount: /mnt/u27d: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
mount-dax=inode-rc=32
== mount -o dax=never
mount: /mnt/u27d: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
mount-dax=never-rc=32
[  180.430500] XFS (loop0): Ending clean mount
[  180.455162] XFS (loop0): Unmounting Filesystem 13acabd9-3aeb-424f-be0f-cc350d77b029
[  214.582211] XFS (loop0): Mounting V5 Filesystem 5445687e-02d5-4c27-ae13-18f2ea442c46
[  214.582874] XFS (loop0): Ending clean mount
[  214.642198] XFS (loop0): Unmounting Filesystem 5445687e-02d5-4c27-ae13-18f2ea442c46
[  214.680926] XFS (loop0): Mounting V5 Filesystem 7a5c559e-fe62-4ceb-a18c-4f6f32cf9429
[  214.681491] XFS (loop0): Ending clean mount
[  214.734986] XFS (loop0): Unmounting Filesystem 7a5c559e-fe62-4ceb-a18c-4f6f32cf9429
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (355.114 MiB/sec and 90909.0909 ops/sec)
wrote 4096/4096 bytes at offset 0
4 KiB, 1 ops; 0.0000 sec (355.114 MiB/sec and 90909.0909 ops/sec)
== d
--------------x-X /mnt/u27d/d 
stat.attributes = 0x0
stat.attributes_mask = 0x203070
== d/f
--------------x-X /mnt/u27d/d/f 
stat.attributes = 0x0
stat.attributes_mask = 0x203070
== g
--------------x-X /mnt/u27d/g 
stat.attributes = 0x0
stat.attributes_mask = 0x203070
== mmap shared, then MAP_SYNC
[000] 0x7f3a893bd000 - 0x7f3a893be000 rw-     /mnt/u27d/g (0 : 4096)
mmap: Operation not supported
no mapped regions, try 'help mmap'
mmapS-rc=1
core.mode = 040775
v3.dax = 1
core.mode = 0100600
v3.dax = 1
core.mode = 0100600
v3.dax = 1
```

