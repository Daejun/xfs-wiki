# 운영: 원본 출력

> 본문은 ../33-operations.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## mkfs.xfs -N 기본 geometry와 만든 뒤 superblock

**관찰 출력**

```text
nproc=8
dev=/dev/loop0 rotational=0
######## mkfs.xfs -N []
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
rc=0
######## default mkfs, then sb 0
rc=0
agcount = 4
agblocks = 65536
dblocks = 262144
logblocks = 16384
logstart = 131079
inodesize = 512
imax_pct = 25
unit = 0
width = 0
features2 = 0x18a
features_compat = 0
features_ro_compat = 0xf
features_incompat = 0xeb
```

## mkfs.xfs -N 변형별 geometry

**관찰 출력**

```text
######## mkfs.xfs -N [-d concurrency=0 -l concurrency=0]
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
rc=0
######## mkfs.xfs -N [-d su=64k,sw=4]
meta-data=/dev/loop0             isize=512    agcount=8, agsize=32752 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
data     =                       bsize=4096   blocks=262016, imaxpct=25
         =                       sunit=16     swidth=64 blks
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=16 blks, lazy-count=1
rc=0
######## mkfs.xfs -N [-m metadir=1]
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       exchange=1   metadir=1
rc=0
######## mkfs.xfs -N -c options=/opt/xfsprogs-7.1.1/share/xfsprogs/mkfs/lts_6.18.conf
Parameters parsed from config file /opt/xfsprogs-7.1.1/share/xfsprogs/mkfs/lts_6.18.conf successfully
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       exchange=1   metadir=0
rc=0
```

## mount와 remount의 option 표시

**관찰 출력**

```text
## mount rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,inode32 rc=0 opts=rw,relatime,inode32,logbufs=8,logbsize=32k,noquota
## remount,inode64 rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,discard rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,logbsize=64k rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,largeio rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,allocsize=1m rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## remount,wsync rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
## mount -o discard rc=0 opts=rw,relatime,discard,inode64,logbufs=8,logbsize=32k,noquota
## mount -o logbufs=4,logbsize=64k rc=0 opts=rw,relatime,inode64,logbufs=4,logbsize=64k,noquota
## mount -o largeio rc=0 opts=rw,relatime,largeio,inode64,logbufs=8,logbsize=32k,noquota
## mount -o allocsize=1m rc=0 opts=rw,relatime,inode64,allocsize=1024k,logbufs=8,logbsize=32k,noquota
## mount -o inode32 rc=0 opts=rw,relatime,inode32,logbufs=8,logbsize=32k,noquota
## mount -o wsync rc=0 opts=rw,relatime,wsync,inode64,logbufs=8,logbsize=32k,noquota
## mount -o filestreams rc=0 opts=rw,relatime,filestreams,inode64,logbufs=8,logbsize=32k,noquota
```

## sysctl 값과 rotorstep 쓰기

**관찰 출력**

```text
== /proc/sys/fs/xfs
error_level=3
filestream_centisecs=3000
inherit_noatime=1
inherit_nodefrag=1
inherit_nodump=1
inherit_nosymlinks=0
inherit_sync=1
panic_mask=0
rotorstep=1
speculative_prealloc_lifetime=300
stats_clear=0
xfssyncd_centisecs=3000
== rotorstep writes
sh: 1: echo: echo: I/O error
write 256 rc=1 now=1
write 255 rc=0 now=255
sh: 1: echo: echo: I/O error
write 0 rc=1 now=255
write 1 rc=0 now=1
```

## sysfs와 procfs 구성

**관찰 출력**

```text
== /sys/fs/xfs before mount
stats
## mount rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
== /sys/fs/xfs after mount
loop0
stats
/sys/fs/xfs/loop0/error/fail_at_unmount=1
/sys/fs/xfs/loop0/error/metadata/EIO/max_retries=-1
/sys/fs/xfs/loop0/error/metadata/EIO/retry_timeout_seconds=-1
/sys/fs/xfs/loop0/error/metadata/ENODEV/max_retries=0
/sys/fs/xfs/loop0/error/metadata/ENODEV/retry_timeout_seconds=0
/sys/fs/xfs/loop0/error/metadata/ENOSPC/max_retries=-1
/sys/fs/xfs/loop0/error/metadata/ENOSPC/retry_timeout_seconds=-1
/sys/fs/xfs/loop0/error/metadata/default/max_retries=-1
/sys/fs/xfs/loop0/error/metadata/default/retry_timeout_seconds=-1
/sys/fs/xfs/loop0/log/log_head_lsn=1:2
/sys/fs/xfs/loop0/log/log_tail_lsn=1:2
/sys/fs/xfs/loop0/log/reserve_grant_head_bytes=0
/sys/fs/xfs/loop0/log/write_grant_head_bytes=0
/sys/fs/xfs/loop0/stats/stats: extent_alloc 0 0 0 0 abt 0 0 0 0 blk_map 0 0 0 0 0 0 0 bmbt 0 0 0 0 dir 0 0 0 0 trans 0 0 0 ig 3 0 0 3 0 0 0 log 0 0 0 0
/sys/fs/xfs/loop0/stats/stats_clear (write-only)
== /proc/fs/xfs
total 0
lrwxrwxrwx 1 root root 23 Oct 23 01:50 stat -> /sys/fs/xfs/stats/stats
-r--r--r-- 1 root root  0 Oct 23 01:50 xqm
-r--r--r-- 1 root root  0 Oct 23 01:50 xqmstat
extent_alloc 78 9565 6 48098
abt 0 0 0 0
== stats_clear
rc=0 stats_clear=0
extent_alloc 0 0 0 0
abt 0 0 0 0
== workqueues in sysfs
(no xfs workqueue)
```

## 일반 kernel에서 quotacheck mount의 xfs_pwork_init

**관찰 출력**

```text
== quotacheck mount with xfs_pwork_init trace
## mount -o uquota rc=0
           mount-10320   [003] .....   172.234682: xfs_pwork_init: dev 7:0 nr_threads 0 pid 10320
(no xfs workqueue after mount)
```

## debug kernel의 debug knob과 workqueue

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
dev=/dev/loop0
== workqueues before mount
xfs_mru_cache
xfsalloc
xfsdiscard
## mount rc=0
debug
loop0
stats
== /sys/fs/xfs/debug
always_cow=0
bload_leaf_slack=-1
bload_node_slack=-1
bug_on_assert=0
larp=0
log_recovery_delay=0
mount_delay=0
pwork_threads=-1
== workqueues after mount
xfs-blockgc!loop0
xfs-buf!loop0
xfs-cil!loop0
xfs-conv!loop0
xfs-inodegc!loop0
xfs-log!loop0
xfs-reclaim!loop0
xfs-sync!loop0
xfs_mru_cache
xfsalloc
xfsdiscard
== xfs-blockgc!loop0
max_active=1024
nice=0
cpumask=ff
per_cpu=0
== xfs-buf!loop0
max_active=1
per_cpu=1
== xfs-inodegc!loop0
max_active=1
per_cpu=1
== workqueues after umount
xfs_mru_cache
xfsalloc
xfsdiscard
```

## growfs: 8 KiB 증가와 100 MiB 증가

**관찰 출력**

```text
######## A: +8KiB (tests/xfs/606)
dev=/dev/loop0 bytes=1073741824
sb 0: agcount = 4 agblocks = 65536 dblocks = 262144 imax_pct = 25 fdblocks = 245724 
dev=/dev/loop0 bytes=1073750016
## mount rc=0
## xfs_growfs 
data     =                       bsize=4096   blocks=262144, imaxpct=25
log      =internal log           bsize=4096   blocks=16384, version=2
rc=0
## umount rc=0
sb 0: agcount = 4 agblocks = 65536 dblocks = 262144 imax_pct = 25 fdblocks = 245724 
######## B: 1GiB+100MiB
dev=/dev/loop0 bytes=1178599424
## mount rc=0
## xfs_growfs -n
data     =                       bsize=4096   blocks=262144, imaxpct=25
log      =internal log           bsize=4096   blocks=16384, version=2
rc=0
## xfs_growfs -d
data     =                       bsize=4096   blocks=262144, imaxpct=25
log      =internal log           bsize=4096   blocks=16384, version=2
data blocks changed from 262144 to 287744
rc=0
## umount rc=0
sb 0: agcount = 5 agblocks = 65536 dblocks = 287744 imax_pct = 25 fdblocks = 271317 
sb 4: agcount = 5 agblocks = 65536 dblocks = 287744 imax_pct = 25 fdblocks = 258399 
agf 3: length = 65536 freeblks = 65523 longest = 65523 
agf 4: length = 25600 freeblks = 25593 longest = 25593
```

## growfs: 2 GiB까지, imaxpct, shrink, log

**관찰 출력**

```text
######## C: 2GiB
dev=/dev/loop0 bytes=2147483648
## mount rc=0
## xfs_growfs -d
data     =                       bsize=4096   blocks=287744, imaxpct=25
log      =internal log           bsize=4096   blocks=16384, version=2
data blocks changed from 287744 to 524288
rc=0
## umount rc=0
sb 0: agcount = 8 agblocks = 65536 dblocks = 524288 imax_pct = 25 fdblocks = 507840 
agf 4: length = 65536 freeblks = 65523 longest = 65523 
agf 7: length = 65536 freeblks = 65529 longest = 65529 
sb 1: agcount = 8 agblocks = 65536 dblocks = 524288 imax_pct = 25 fdblocks = 494430 
sb 4: agcount = 8 agblocks = 65536 dblocks = 524288 imax_pct = 25 fdblocks = 494430 
sb 7: agcount = 8 agblocks = 65536 dblocks = 524288 imax_pct = 25 fdblocks = 494430 
######## D: imaxpct, shrink, log
dev=/dev/loop0 bytes=2147483648
## mount rc=0
## xfs_growfs -m 10
data     =                       bsize=4096   blocks=524288, imaxpct=25
log      =internal log           bsize=4096   blocks=16384, version=2
inode max percent changed from 25 to 10
rc=0
dblocks(xfs_info)=524288
## xfs_growfs -D 523288
data     =                       bsize=4096   blocks=524288, imaxpct=10
log      =internal log           bsize=4096   blocks=16384, version=2
data blocks changed from 524288 to 523288
rc=0
[  157.437138] XFS (loop0): EXPERIMENTAL online shrink feature enabled.  Use at your own risk!
## xfs_growfs -L 30000
data     =                       bsize=4096   blocks=523288, imaxpct=10
log      =internal log           bsize=4096   blocks=16384, version=2
rc=1
## umount rc=0
sb 0: agcount = 8 agblocks = 65536 dblocks = 523288 imax_pct = 10 fdblocks = 506840 
agf 7: length = 64536 freeblks = 64523 longest = 64523 
sb 7: agcount = 8 agblocks = 65536 dblocks = 523288 imax_pct = 10 fdblocks = 487908 
repair -n rc=0
```

## fstrim 전체 범위

**관찰 출력**

```text
dev=/dev/loop0 discard_max_bytes=4294966784 discard_granularity=4096
agcount=4 lastag=0x3
  65536  131072       4 499399  99.79  99.79   0.56
total free extents 711
total free blocks 500430
average free extent size 703.84
######## fstrim -v 
/mnt/p33: 2 GiB (2147483648 bytes) trimmed
rc=0
-- events per type and AG
    569 xfs_discard_exclude: agno 0x0
    105 xfs_discard_extent: agno 0x0
      2 xfs_discard_extent: agno 0x1
      3 xfs_discard_extent: agno 0x2
      2 xfs_discard_extent: agno 0x3
-- first 5 xfs_discard_extent in AG 0
gbno 0x1d2b fsbcount 0x1e2d5
gbno 0xb97 fsbcount 0x138
gbno 0x1d24 fsbcount 0x1
gbno 0x1d22 fsbcount 0x1
gbno 0x1d20 fsbcount 0x1
-- first 5 xfs_discard_extent in last AG
gbno 0x2ba fsbcount 0x6
gbno 0x2d0 fsbcount 0x1fd30
```

## fstrim -m 1M와 -o 10M -l 200M

**관찰 출력**

```text
######## fstrim -v -m 1M
/mnt/p33: 2 GiB (2147483648 bytes) trimmed
rc=0
-- events per type and AG
      2 xfs_discard_extent: agno 0x0
      1 xfs_discard_extent: agno 0x1
      1 xfs_discard_extent: agno 0x2
      1 xfs_discard_extent: agno 0x3
      1 xfs_discard_toosmall: agno 0x0
      1 xfs_discard_toosmall: agno 0x1
      1 xfs_discard_toosmall: agno 0x2
      1 xfs_discard_toosmall: agno 0x3
-- first 5 xfs_discard_extent in AG 0
gbno 0x1d2b fsbcount 0x1e2d5
gbno 0xb97 fsbcount 0x138
-- first 5 xfs_discard_extent in last AG
gbno 0x2d0 fsbcount 0x1fd30
######## fstrim -v -o 10M -l 200M
/mnt/p33: 200 MiB (209715200 bytes) trimmed
rc=0
-- events per type and AG
      1 xfs_discard_exclude: agno 0x0
    173 xfs_discard_extent: agno 0x0
-- first 5 xfs_discard_extent in AG 0
gbno 0xb97 fsbcount 0x138
gbno 0x1bd0 fsbcount 0x1
gbno 0x1bd2 fsbcount 0x1
gbno 0x1bd4 fsbcount 0x1
gbno 0x1bd6 fsbcount 0x1
-- first 5 xfs_discard_extent in last AG
```

## xfs_spaceman trim과 온라인 discard

**관찰 출력**

```text
######## xfs_spaceman trim -a 1
rc=0
-- events per type and AG
      2 xfs_discard_extent: agno 0x1
-- first 5 xfs_discard_extent in AG 0
-- first 5 xfs_discard_extent in last AG
du_k_after=87916
######## online discard (-o discard)
## opts=rw,relatime,discard,inode64,logbufs=8,logbsize=32k,noquota
-- events per type and AG
      1 xfs_discard_extent: agno 0x0
-- first 5 xfs_discard_extent in AG 0
gbno 0x1d2b fsbcount 0x800
-- first 5 xfs_discard_extent in last AG
######## remount,discard
## remount rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
-- events per type and AG
-- first 5 xfs_discard_extent in AG 0
-- first 5 xfs_discard_extent in last AG
```

## metadump 변형별 결과

**관찰 출력**

```text
######## metadump while ro mounted
rc=0 size=91446784
######## metadump while rw mounted
xfs_metadump: /dev/loop0 contains a mounted and writable filesystem

fatal error -- couldn't initialize XFS library
rc=1 size=
######## xfs_metadump []
rc=0 size=91446784
magic=XFSM
/dev/shm/p33.md: obfuscated, clean log, zeroed metadata blocks
mdrestore rc=0 apparent=2147483648 du_k=87916
## mount rc=0
first file=
repair -n rc=0
######## xfs_metadump [-o]
rc=0 size=91446784
magic=XFSM
/dev/shm/p33.md: not obfuscated, clean log, zeroed metadata blocks
mdrestore rc=0 apparent=2147483648 du_k=87916
## mount rc=0
ATTR.FMT_BTREE
first file=/mnt/p33/BNOBT
repair -n rc=0
######## xfs_metadump [-a -o]
rc=0 size=91446784
magic=XFSM
/dev/shm/p33.md: not obfuscated, clean log, full metadata blocks
mdrestore rc=0 apparent=2147483648 du_k=87916
## mount rc=0
ATTR.FMT_BTREE
first file=/mnt/p33/BNOBT
repair -n rc=0
######## xfs_metadump [-v 2]
rc=0 size=90039464
magic=XMD2
/dev/shm/p33.md: obfuscated, clean log, external log contents are not dumped, rt device contents are not dumped, zeroed metadata blocks,
mdrestore rc=0 apparent=2147483648 du_k=87916
## mount rc=0
first file=
repair -n rc=0
######## xfs_metadump [-v 2 -o]
rc=0 size=90039464
magic=XMD2
/dev/shm/p33.md: not obfuscated, clean log, external log contents are not dumped, rt device contents are not dumped, zeroed metadata blocks,
mdrestore rc=0 apparent=2147483648 du_k=87916
## mount rc=0
ATTR.FMT_BTREE
first file=/mnt/p33/BNOBT
repair -n rc=0
```

## 일반 사용자와 root의 fsmap

**관찰 출력**

```text
######## [user] xfs_io -c fsmap -v /mnt/p33
 EXT: DEV BLOCK-RANGE         OWNER            FILE-OFFSET      AG AG-OFFSET         TOTAL
   0: 7:0 [0..103]:           unknown                           0  (0..103)            104
   1: 7:0 [104..127]:         free space                        0  (104..127)           24
   2: 7:0 [128..2239]:        unknown                           0  (128..2239)        2112
   3: 7:0 [2240..524287]:     free space                        0  (2240..524287)   522048
   4: 7:0 [524288..524391]:   unknown                           1  (0..103)            104
rc=0
      1 FS_IOC_GETFSMAP
      2 _IOC(_IOC_READ
######## [root] xfs_io -c fsmap -v /mnt/p33
 EXT: DEV BLOCK-RANGE         OWNER              FILE-OFFSET      AG AG-OFFSET         TOTAL
   0: 7:0 [0..7]:             static fs metadata                  0  (0..7)                8
   1: 7:0 [8..23]:            per-AG metadata                     0  (8..23)              16
   2: 7:0 [24..39]:           inode btree                         0  (24..39)             16
   3: 7:0 [40..47]:           per-AG metadata                     0  (40..47)              8
   4: 7:0 [48..55]:           refcount btree                      0  (48..55)              8
rc=0
      1 FS_IOC_GETFSMAP
      2 _IOC(_IOC_READ
```

## 권한이 필요한 ioctl의 거절

**관찰 출력**

```text
######## [user] xfs_io -c bulkstat -q /mnt/p33
xfrog_bulkstat: Operation not permitted
rc=1
      3 _IOC(_IOC_READ
      1 _IOC(_IOC_READ|_IOC_WRITE
######## [user] xfs_io -c inumbers /mnt/p33
xfrog_inumbers: Operation not permitted
rc=1
      3 _IOC(_IOC_READ
      1 _IOC(_IOC_READ|_IOC_WRITE
######## [user] xfs_io -c parent /mnt/p33/d/g
/mnt/p33/d/g: Operation not permitted
rc=1
      1 _IOC(_IOC_READ
      2 _IOC(_IOC_READ|_IOC_WRITE
######## [root] xfs_io -c parent /mnt/p33/d/g
p_ino     = 132
p_gen     = 2192948824
p_namelen = 1
p_name    = "g"

rc=0
      1 _IOC(_IOC_READ
      2 _IOC(_IOC_READ|_IOC_WRITE
######## [root] xfs_io -x -c scrub sb 0 /mnt/p33
rc=0
      1 _IOC(_IOC_READ
      1 _IOC(_IOC_READ|_IOC_WRITE
######## [user] xfs_io -x -c scrub sb 0 /mnt/p33
scrub: Operation not permitted
rc=0
      1 _IOC(_IOC_READ
      1 _IOC(_IOC_READ|_IOC_WRITE
```

## dioinfo, resblks, shutdown

**관찰 출력**

```text
######## [user] xfs_io -c dioinfo /mnt/p33/f
command "dioinfo" not found
rc=0
      1 _IOC(_IOC_READ
######## [root] xfs_io -x -c resblks /mnt/p33
reserved blocks = 8192
available reserved blocks = 8192
rc=0
      2 _IOC(_IOC_READ
######## [root] xfs_io -x -c shutdown -f /mnt/p33
rc=0
      1 EXT4_IOC_SHUTDOWN or F2FS_IOC_SHUTDOWN
      1 _IOC(_IOC_READ
######## [user] xfs_io -c statfs /mnt/p33
stat: Input/output error
rc=1
[  174.469360] XFS (loop0): User initiated shutdown received.
[  174.470126] XFS (loop0): Metadata I/O Error (0x4) detected at xfs_fs_goingdown+0x6f/0xb0 (fs/xfs/xfs_fsops.c:462).  Shutting down filesystem.
```

