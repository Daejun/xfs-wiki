# XFS의 설계와 역사: 원본 출력

> 본문은 ../01-overview-history.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## option 없는 mkfs의 출력

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
Discarding blocks...Done.
```

## superblock의 판 번호와 feature mask, mount message

**관찰 출력**

```text
versionnum [0xb4b5+0x18a] = V5,NLINK,DIRV2,ATTR,ALIGN,LOGV2,EXTFLG,MOREBITS,LAZYSBCOUNT,PROJID32BIT,CRC,FTYPE,FINOBT,SPARSE_INODES,RMAPBT,REFLINK,INOBTCNT,BIGTIME,NREXT64,EXCHANGE,PARENT
versionnum = 0xb4b5
features2 = 0x18a
features_compat = 0
features_ro_compat = 0xf
features_incompat = 0xeb
features_log_incompat = 0
agcount = 4
agblocks = 65536
crc = 0x794f9468 (correct)
[ 1324.636194] XFS (loop0): Mounting V5 Filesystem 9112dac1-faf6-41d4-b86b-2050463dcebb
[ 1324.637095] XFS (loop0): Ending clean mount
[ 1324.645226] XFS (loop0): Unmounting Filesystem 9112dac1-faf6-41d4-b86b-2050463dcebb
```

## 관찰 image의 uname과 boot banner

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
[    0.427038] SGI XFS with ACLs, security attributes, realtime, scrub, repair, quota, no debug enabled
```

## superblock geometry, inode 번호, AG 번호

**관찰 출력**

```text
agcount = 4
agblocks = 65536
agblklog = 16
inopblog = 3
rootino = 128
d1 131 132
d2 524416 524417
d3 1179776 1179777
d4 1572992 1572993
d5 133 134
d6 524418 524419
d7 1179778 1179779
d8 1572994 1572995
== d1 dir 131 file 132
0x0 (0)
0x0 (0)
== d2 dir 524416 file 524417
0x1 (1)
0x1 (1)
== d3 dir 1179776 file 1179777
0x2 (2)
0x2 (2)
== d4 dir 1572992 file 1572993
0x3 (3)
0x3 (3)
== d5 dir 133 file 134
0x0 (0)
0x0 (0)
== d6 dir 524418 file 524419
0x1 (1)
0x1 (1)
== d7 dir 1179778 file 1179779
0x2 (2)
0x2 (2)
== d8 dir 1572994 file 1572995
0x3 (3)
0x3 (3)
```

## 관찰 image의 XFS Kconfig

**관찰 출력**

```text
949:CONFIG_BLK_DEV_ZONED=y
4668:CONFIG_XFS_FS=y
4669:# CONFIG_XFS_SUPPORT_V4 is not set
4670:# CONFIG_XFS_SUPPORT_ASCII_CI is not set
4671:CONFIG_XFS_QUOTA=y
4672:CONFIG_XFS_POSIX_ACL=y
4673:CONFIG_XFS_RT=y
4674:CONFIG_XFS_DRAIN_INTENTS=y
4675:CONFIG_XFS_LIVE_HOOKS=y
4676:CONFIG_XFS_MEMORY_BUFS=y
4677:CONFIG_XFS_BTREE_IN_MEM=y
4678:CONFIG_XFS_ONLINE_SCRUB=y
4679:CONFIG_XFS_ONLINE_SCRUB_STATS=y
4680:CONFIG_XFS_ONLINE_REPAIR=y
4681:# CONFIG_XFS_WARN is not set
4682:# CONFIG_XFS_DEBUG is not set
40:CONFIG_LOCALVERSION=""
```

## zoned device, mkfs 출력, mount의 kernel log

**관찰 출력**

```text
host-managed
16
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
realtime =/dev/p01z              extsz=4096   blocks=262144, rtextents=262144
         =                       rgcount=16   rgsize=16384 extents
         =                       zoned=1      start=0 reserved=0
Discarding blocks...Done.
Resetting zones...Done.
[ 1328.942593] XFS (loop0): EXPERIMENTAL zoned RT device feature enabled.  Use at your own risk!
[ 1328.942637] XFS (loop0): Mounting V5 Filesystem 7df09360-7d65-4c8d-a50a-4fd5dfcc1a2d
[ 1328.943471] XFS (loop0): Ending clean mount
[ 1328.943477] XFS (loop0): limiting open zones to 4 due to total zone count (16)
[ 1328.943551] XFS (loop0): 16 zones of 16384 blocks (4 max open zones)
```

