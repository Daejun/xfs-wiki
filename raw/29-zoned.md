# zoned XFS: 원본 출력

> 본문은 ../29-zoned.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## mkfs, mount, sysfs

**관찰 출력**

```text
nullb p29z: zoned=host-managed nr_zones=16 chunk_sectors=131072 max_open_zones=0
data=/dev/loop0 rt=/dev/p29z
== mkfs
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=0    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =/dev/p29z              extsz=4096   blocks=262144, rtextents=262144
         =                       rgcount=16   rgsize=16384 extents
         =                       zoned=1      start=0 reserved=0
Discarding blocks...Done.
Resetting zones...Done.
== dmesg
[  372.510847] XFS (loop0): EXPERIMENTAL zoned RT device feature enabled.  Use at your own risk!
[  372.510884] XFS (loop0): Mounting V5 Filesystem 4430db62-20aa-4a06-9542-2b4e1dd6c8e2
[  372.511690] XFS (loop0): Ending clean mount
[  372.511695] XFS (loop0): limiting open zones to 4 due to total zone count (16)
[  372.511743] XFS (loop0): 16 zones of 16384 blocks (4 max open zones)
== trace during mount
mount-2744 xfs_zones_mount: dev 7:0 zoned 16 blocks_per_zone 16384, max_open 4
== sysfs
zoned/max_open_zones=3
zoned/nr_open_zones=0
zoned/zonegc_low_space=0
```

## superblock, /rtgroups, rmap inode

**관찰 출력**

```text
== sb 0
blocksize = 4096
rblocks = 262144
rextents = 262144
rbmino = 0
rsumino = 0
rextsize = 1
rbmblocks = 0
frextents = 0
features_ro_compat = 0xb
features_incompat = 0x3eb
metadirino = 129
rgcount = 16
rgextents = 16384
rgblklog = 14
rtstart = 0
rtreserved = 0
== ls -m /rtgroups (head, tail)
/rtgroups:
8          1179776            directory      0x0000002e   1 . (good)
10         129                directory      0x0000172e   2 .. (good)
12         1179777            regular        0xee5b7172   6 0.rmap (good)
15         1179778            regular        0xee5b717a   6 1.rmap (good)
18         1179779            regular        0xee5b7162   6 2.rmap (good)
51         1179790            regular        0xee5bb56a   7 13.rmap (good)
54         1179791            regular        0xee5bb552   7 14.rmap (good)
57         1179792            regular        0xee5bb55a   7 15.rmap (good)
== rtsb (expected to fail on zoned)
Metadata CRC error detected at 0x55d3f8608b6c, xfs_rtsb block 0x0/0x1000
magicnum = 0xcdcdcdcd
crc = 0xcdcdcdcd (bad)
pad = 0xcdcdcdcd
== metadir:/rtgroups/0.rmap inode core
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.nextents = 0
core.size = 0
core.nblocks = 0
v3.flags2 = 0x38
v3.cowextsize = 256
v3.metadata = 1
== metadir:/rtgroups/1.rmap inode core
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.nextents = 0
core.size = 0
core.nblocks = 0
v3.flags2 = 0x38
v3.cowextsize = 0
v3.metadata = 1
== metadir:/rtgroups/15.rmap inode core
core.format = 5 (meta_btree)
core.metatype = 7 (rtrmap)
core.nextents = 0
core.size = 0
core.nblocks = 0
v3.flags2 = 0x38
v3.cowextsize = 0
v3.metadata = 1
```

## 1 MiB를 쓴 뒤의 xflags, mountstats, statfs

**관찰 출력**

```text
== root and new file xflags
fsxattr.xflags = 0x80000100 [-------t--------X]
fsxattr.xflags = 0x80000001 [r---------------X]
f1 rg=0 
== mountstats
device /dev/loop0 mounted on /mnt/p29 with fstype xfs 
	user free RT blocks: 196352
	reserved free RT blocks: 65536
	user available RT blocks: 212736
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	total number of zones: 16
	free zones: 15
	max open zones: 4
	nr open zones: 1
	nr open GC zones: 0
	open zones:
	  zone 0, wp 256, written 256, used 256, hint NOT_SET 
== statfs: root (rtinherit) and a data-device dir
/mnt/p29 bsize=4096 blocks=196608 bfree=196352 bavail=196352
/mnt/p29/d bsize=4096 blocks=245760 bfree=230971 bavail=230971
4K-blocks  Used  Avail
   196608   256 196352
```

## d1~d4, b1~b4, c1, c2의 zone과 위치

**관찰 출력**

```text
== A: direct 1MiB, files d1..d4 one after another
d1 rg=0 
d2 rg=1 
d3 rg=2 
d4 rg=0 
xfs_io-3049 xfs_zone_opened: dev 7:0 rgno 0x0 used 0x0 nr_open 1
xfs_io-3049 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
kworker/0:1-11 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
xfs_io-3060 xfs_zone_opened: dev 7:0 rgno 0x1 used 0x0 nr_open 2
xfs_io-3060 xfs_zone_alloc_blocks: dev 7:0 rgno 0x1 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
kworker/6:1-154 xfs_zone_record_blocks: dev 7:0 rgno 0x1 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
xfs_io-3071 xfs_zone_opened: dev 7:0 rgno 0x2 used 0x0 nr_open 3
xfs_io-3071 xfs_zone_alloc_blocks: dev 7:0 rgno 0x2 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
kworker/0:1-11 xfs_zone_record_blocks: dev 7:0 rgno 0x2 used 0x0 alloced 0x100 written 0x0 rgbno 0x0 len 0x100
xfs_io-3082 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x100 alloced 0x200 written 0x100 rgbno 0x100 len 0x100
kworker/0:1-11 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x100 alloced 0x200 written 0x100 rgbno 0x100 len 0x100
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 1, wp 256, written 256, used 256, hint NOT_SET 
	  zone 2, wp 256, written 256, used 256, hint NOT_SET 
	  zone 0, wp 512, written 512, used 512, hint NOT_SET 
== B: buffered 256KiB, files b1..b4 closed, then sync
b1 rg=0 
b2 rg=0 
b3 rg=0 
b4 rg=0 
kworker/u32:3-97 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x200 alloced 0x240 written 0x200 rgbno 0x200 len 0x40
kworker/u32:3-97 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x200 alloced 0x280 written 0x200 rgbno 0x240 len 0x40
kworker/u32:3-97 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x200 alloced 0x2c0 written 0x200 rgbno 0x280 len 0x40
kworker/u32:3-97 xfs_zone_alloc_blocks: dev 7:0 rgno 0x0 used 0x200 alloced 0x300 written 0x200 rgbno 0x2c0 len 0x40
kworker/2:0-2565 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x200 alloced 0x300 written 0x200 rgbno 0x200 len 0x40
kworker/2:0-2565 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x240 alloced 0x300 written 0x240 rgbno 0x240 len 0x40
kworker/2:0-2565 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x280 alloced 0x300 written 0x280 rgbno 0x280 len 0x40
kworker/2:0-2565 xfs_zone_record_blocks: dev 7:0 rgno 0x0 used 0x2c0 alloced 0x300 written 0x2c0 rgbno 0x2c0 len 0x40
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 1, wp 256, written 256, used 256, hint NOT_SET 
	  zone 2, wp 256, written 256, used 256, hint NOT_SET 
	  zone 0, wp 768, written 768, used 768, hint NOT_SET 
== C: buffered 256KiB, fsync while the file is open, files c1 c2
c1 rg=1 
c2 rg=2 
xfs_io-3191 xfs_zone_alloc_blocks: dev 7:0 rgno 0x1 used 0x100 alloced 0x140 written 0x100 rgbno 0x100 len 0x40
kworker/6:1-154 xfs_zone_record_blocks: dev 7:0 rgno 0x1 used 0x100 alloced 0x140 written 0x100 rgbno 0x100 len 0x40
xfs_io-3202 xfs_zone_alloc_blocks: dev 7:0 rgno 0x2 used 0x100 alloced 0x140 written 0x100 rgbno 0x100 len 0x40
kworker/1:1-72 xfs_zone_record_blocks: dev 7:0 rgno 0x2 used 0x100 alloced 0x140 written 0x100 rgbno 0x100 len 0x40
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 0, wp 768, written 768, used 768, hint NOT_SET 
	  zone 1, wp 320, written 320, used 320, hint NOT_SET 
	  zone 2, wp 320, written 320, used 320, hint NOT_SET 
== bmap d1
/mnt/p29/d1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       0..2047           0 (0..2047)         2048
== bmap d4
/mnt/p29/d4:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..2047]:       2048..4095        0 (2048..4095)      2048
== bmap b1
/mnt/p29/b1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..511]:        4096..4607        0 (4096..4607)       512
== bmap b4
/mnt/p29/b4:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..511]:        5632..6143        0 (5632..6143)       512
== bmap c1
/mnt/p29/c1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..511]:        133120..133631    1 (2048..2559)       512
== bmap c2
/mnt/p29/c2:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..511]:        264192..264703    2 (2048..2559)       512
```

## write hint별 rtgroup

**관찰 출력**

```text
######## r1: lifetime, buffered, file order (xfs/638)
r1 short: rg = 0 
r1 medium: rg = 1 
r1 long: rg = 2 
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 0, wp 2560, written 2560, used 2560, hint SHORT 
	  zone 1, wp 2560, written 2560, used 2560, hint MEDIUM 
	  zone 2, wp 2560, written 2560, used 2560, hint LONG 
######## r2: lifetime, direct, hint order
r2 short: rg = 0 
r2 medium: rg = 1 
r2 long: rg = 2 
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 0, wp 2560, written 2560, used 2560, hint SHORT 
	  zone 1, wp 2560, written 2560, used 2560, hint MEDIUM 
	  zone 2, wp 2560, written 2560, used 2560, hint LONG 
######## r3: nolifetime, direct, hint order
r3 short: rg = 0 1 2 
r3 medium: rg = 0 1 2 
r3 long: rg = 0 1 2 
	free zones: 13
	nr open zones: 3
	nr open GC zones: 0
	  zone 0, wp 2560, written 2560, used 2560, hint NOT_SET 
	  zone 1, wp 2560, written 2560, used 2560, hint NOT_SET 
	  zone 2, wp 2560, written 2560, used 2560, hint NOT_SET 
######## r4: nolifetime, buffered, hint order
r4 short: rg = 0 
r4 medium: rg = 0 
r4 long: rg = 0 
	free zones: 15
	nr open zones: 1
	nr open GC zones: 0
	  zone 0, wp 7680, written 7680, used 7680, hint NOT_SET
```

## GC 전: 세 zone을 채우고 truncate한 상태

**관찰 출력**

```text
mount-5249 xfs_zones_mount: dev 7:0 zoned 16 blocks_per_zone 16384, max_open 2
zoned/max_open_zones=1
zoned/nr_open_zones=0
zoned/zonegc_low_space=0
dd-5286 xfs_zone_opened: dev 7:0 rgno 0x0 used 0x0 nr_open 1
kworker/3:8-4772 xfs_zone_full: dev 7:0 rgno 0x0 used 0x4000 nr_open 1
dd-5288 xfs_zone_opened: dev 7:0 rgno 0x1 used 0x0 nr_open 1
kworker/1:1-72 xfs_zone_full: dev 7:0 rgno 0x1 used 0x4000 nr_open 1
dd-5290 xfs_zone_opened: dev 7:0 rgno 0x2 used 0x0 nr_open 1
kworker/4:2-198 xfs_zone_full: dev 7:0 rgno 0x2 used 0x4000 nr_open 1
data_1 rg=0 
data_2 rg=1 
data_3 rg=2 
== before GC
device /dev/loop0 mounted on /mnt/p29 with fstype xfs 
	user free RT blocks: 194208
	reserved free RT blocks: 65536
	user available RT blocks: 163840
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	total number of zones: 16
	free zones: 13
	max open zones: 2
	nr open zones: 0
	nr open GC zones: 0
	open zones:
	used blocks distribution (fully written zones):
	   0.. 9%: 3
zoned 0 0 0
gc xpc 0
```

## read-only와 read-write에서 GC 시작

**관찰 출력**

```text
== read-only: zonegc_low_space=100 for 1s
ro before: user available RT blocks: 212992 | after 1s: user available RT blocks: 212992
== read-write: zonegc_low_space=100
	user available RT blocks: 163840
	RT GC required: 0
waited 1s
xfs-zone-gc/loo-5258 xfs_zone_gc_select_victim: dev 7:0 rgno 0x0 used 0x320 bucket 0
xfs-zone-gc/loo-5258 xfs_zone_gc_target_opened: dev 7:0 rgno 0x3 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_gc_select_victim: dev 7:0 rgno 0x1 used 0x320 bucket 0
xfs-zone-gc/loo-5258 xfs_zone_emptied: dev 7:0 rgno 0x0 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_reset: dev 7:0 rgno 0x0 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_gc_select_victim: dev 7:0 rgno 0x2 used 0x320 bucket 0
xfs-zone-gc/loo-5258 xfs_zone_emptied: dev 7:0 rgno 0x1 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_reset: dev 7:0 rgno 0x1 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_emptied: dev 7:0 rgno 0x2 used 0x0 nr_open 0
xfs-zone-gc/loo-5258 xfs_zone_reset: dev 7:0 rgno 0x2 used 0x0 nr_open 0
```

## GC 뒤의 상태

**관찰 출력**

```text
== after GC
device /dev/loop0 mounted on /mnt/p29 with fstype xfs 
	user free RT blocks: 194208
	reserved free RT blocks: 65536
	user available RT blocks: 210592
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	total number of zones: 16
	free zones: 15
	max open zones: 2
	nr open zones: 0
	nr open GC zones: 1
	open zones:
	  zone 3, wp 2400, written 2400, used 2400, hint NOT_SET (GC)
zoned 9 9 3
gc xpc 9830400
data_1 rg=3 
data_2 rg=3 
data_3 rg=3 
/mnt/p29/data_1:
 EXT: FILE-OFFSET      RT-BLOCK-RANGE   RG RG-OFFSET        TOTAL
   0: [0..6399]:       393216..399615    3 (0..6399)         6400
data_1 content zero ok
data_2 content zero ok
data_3 content zero ok
```

## mount 직후의 counter와 statfs

**관찰 출력**

```text
== after mount
	user free RT blocks: 196608
	reserved free RT blocks: 65536
	user available RT blocks: 212992
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 16
	nr open zones: 0
/mnt/p29 bsize=4096 blocks=196608 bfree=196608 bavail=196608
/mnt/p29/d bsize=4096 blocks=245760 bfree=230971 bavail=230971
4K-blocks  Used  Avail
   196608     0 196608
```

## ENOSPC까지 채우기와 greedy 예약

**관찰 출력**

```text
== fill with 1MiB direct writes until ENOSPC
768+0 records in
767+0 records out
804257792 bytes (804 MB, 767 MiB) copied, 0.0719839 s, 11.2 GB/s
-rw-rw-r-- 1 ubuntu ubuntu 804257792 Oct 23 01:16 /mnt/p29/fill
	user free RT blocks: 256
	reserved free RT blocks: 65536
	user available RT blocks: 16640
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 4
	nr open zones: 1
4K-blocks   Used Avail
   196608 196352   256
== buffered 8MiB pwrite (greedy reservation)
wrote 1048576/8388608 bytes at offset 0
1 MiB, 1 ops; 0.0000 sec (13.951 GiB/sec and 14285.7143 ops/sec)
-rw------- 1 ubuntu ubuntu 1048576 Oct 23 01:16 /mnt/p29/g
	user free RT blocks: 0
	reserved free RT blocks: 65536
	user available RT blocks: 16384
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 4
	nr open zones: 1
== freecounter trace
dd-5642 xfs_freecounter_enospc: dev 7:0 ctr rtextents delta 259 avail 65536 total 65536 caller xfs_zoned_space_reserve+0x5c/0x370
dd-5642 xfs_freecounter_enospc: dev 7:0 ctr rtextents delta 259 avail 65536 total 65536 caller xfs_zoned_space_reserve+0x318/0x370
xfs_io-5650 xfs_freecounter_enospc: dev 7:0 ctr rtextents delta 2051 avail 65536 total 65536 caller xfs_zoned_space_reserve+0x5c/0x370
xfs_io-5650 xfs_freecounter_enospc: dev 7:0 ctr rtextents delta 2051 avail 65536 total 65536 caller xfs_zoned_space_reserve+0x318/0x370
```

## 지운 뒤 counter가 돌아오는 순서

**관찰 출력**

```text
== rm fill g
	user free RT blocks: 0
	reserved free RT blocks: 65536
	user available RT blocks: 16384
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 4
	nr open zones: 1
== 2s after rm
	user free RT blocks: 196608
	reserved free RT blocks: 65536
	user available RT blocks: 196864
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 15
	nr open zones: 1
== after sync
	user free RT blocks: 196608
	reserved free RT blocks: 65536
	user available RT blocks: 196864
	reserved available RT blocks: 49152
	RT reservations required: 0
	RT GC required: 0
	free zones: 15
	nr open zones: 1
4K-blocks  Used  Avail
   196608     0 196608
zoned 0 0 11
gc xpc 0
```

