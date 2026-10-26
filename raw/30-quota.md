# quota: 원본 출력

> 본문은 ../30-quota.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## non-metadir fs의 mount와 sb_qflags 변화

**관찰 출력**

```text
######## fs A: -m metadir=0
uquotino = 0
gquotino = 0
qflags = 0
pquotino = 0
== A1 first quota mount
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 7c3e5434-5e63-41e4-94cc-eada42cd363e
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x60f
pquotino = 132
== A2 same options again
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 7c3e5434-5e63-41e4-94cc-eada42cd363e
XFS (loop0): Ending clean mount
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x60f
pquotino = 132
== A3 no quota options
## mount [] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
XFS (loop0): Mounting V5 Filesystem 7c3e5434-5e63-41e4-94cc-eada42cd363e
XFS (loop0): resetting quota flags
XFS (loop0): Ending clean mount
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0
pquotino = 132
== A4 quota options again
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 7c3e5434-5e63-41e4-94cc-eada42cd363e
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x60f
pquotino = 132
```

## metadir fs의 mount와 sb_qflags 변화

**관찰 출력**

```text
######## fs B: -m metadir=1
uquotino = 0
gquotino = 0
qflags = 0
pquotino = 0
metadirino = 129
== B1 first quota mount
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0x60f
pquotino = 0
metadirino = 129
== B2 no options
## mount [] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): Ending clean mount
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0x60f
pquotino = 0
metadirino = 129
== B3 -o noquota
## mount [-o noquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): resetting quota flags
XFS (loop0): Ending clean mount
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0
pquotino = 0
metadirino = 129
== B4 no options
## mount [] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,noquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): Ending clean mount
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0
pquotino = 0
metadirino = 129
== B5 quota options
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0x60f
pquotino = 0
metadirino = 129
```

## mkfs -m metadir=1,uquota 직후의 첫 mount

**관찰 출력**

```text
######## fs C: -m metadir=1,uquota
uquotino = 0
gquotino = 0
qflags = 0x3
pquotino = 0
metadirino = 129
== C1 no options
## mount [] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota
XFS (loop0): Mounting V5 Filesystem 6c41b1e2-1de7-4eea-84ab-9a597c0feeb9
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          1          0          0     00 [--------]

## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0x7
pquotino = 0
metadirino = 129
```

## quota inode의 위치와 extent

**관찰 출력**

```text
######## fs A: -m metadir=0
== A1 first quota mount
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x60f
pquotino = 132
== uquotino=131
core.mode = 0100000
core.size = 0
core.nblocks = 2
core.nextents = 2
core.format = 2 (extents)
data offset 0 startblock 15 (0/15) count 1 flag 0
data offset 33 startblock 48 (0/48) count 1 flag 0
== pquotino=132
core.mode = 0100000
core.size = 0
core.nblocks = 2
core.nextents = 2
core.format = 2 (extents)
data offset 0 startblock 14 (0/14) count 1 flag 0
data offset 1 startblock 13 (0/13) count 1 flag 0
######## fs B: -m metadir=1
uquotino = 0
gquotino = 0
qflags = 0
pquotino = 0
metadirino = 129
== B1 first quota mount
## umount rc=0
uquotino = 0
gquotino = 0
qflags = 0x60f
pquotino = 0
metadirino = 129
/quota:
8          1179777            directory      0x0000002e   1 . (good)
10         129                directory      0x0000172e   2 .. (good)
12         1179778            regular        0x0ebcf2f2   4 user (good)
14         1179779            regular        0xfd58b262   7 project (good)
== metadir /quota/user
core.mode = 0100000
core.size = 0
core.nblocks = 2
core.nextents = 2
core.metatype = 2 (usrquota)
data offset 0 startblock 147471 (2/16399) count 1 flag 0
data offset 33 startblock 147504 (2/16432) count 1 flag 0
== metadir /quota/project
core.mode = 0100000
core.size = 0
core.nblocks = 2
core.nextents = 2
core.metatype = 4 (prjquota)
data offset 0 startblock 147470 (2/16398) count 1 flag 0
data offset 1 startblock 147469 (2/16397) count 1 flag 0
```

## dquot record 두 개 (xfs_db)

**관찰 출력**

```text
######## fs A: -m metadir=0
== xfs_db dquot -u 1000
diskdq.magic = 0x4451
diskdq.version = 0x1
diskdq.type = 0x81
diskdq.id = 1000
diskdq.blk_hardlimit = 0
diskdq.blk_softlimit = 0
diskdq.ino_hardlimit = 0
diskdq.ino_softlimit = 0
diskdq.bcount = 112
diskdq.icount = 7
diskdq.itimer = 0
diskdq.btimer = 0
diskdq.iwarns = 0
diskdq.bwarns = 0
diskdq.rtb_hardlimit = 0
diskdq.rtb_softlimit = 0
diskdq.rtbcount = 0
diskdq.rtbtimer = 0
diskdq.rtbwarns = 0
crc = 0xa9b3f904 (correct)
lsn = 0x100000008
uuid = 7c3e5434-5e63-41e4-94cc-eada42cd363e
== xfs_db dquot -p 42
diskdq.magic = 0x4451
diskdq.version = 0x1
diskdq.type = 0x82
diskdq.id = 42
diskdq.blk_hardlimit = 0
diskdq.blk_softlimit = 0
diskdq.ino_hardlimit = 0
diskdq.ino_softlimit = 0
diskdq.bcount = 32
diskdq.icount = 2
diskdq.itimer = 0
diskdq.btimer = 0
diskdq.iwarns = 0
diskdq.bwarns = 0
diskdq.rtb_hardlimit = 0
diskdq.rtb_softlimit = 0
diskdq.rtbcount = 0
diskdq.rtbtimer = 0
diskdq.rtbwarns = 0
crc = 0xafead00d (correct)
lsn = 0x100000008
uuid = 7c3e5434-5e63-41e4-94cc-eada42cd363e
```

## projid와 report (non-metadir fs)

**관찰 출력**

```text
######## fs A: -m metadir=0
== A1 first quota mount
/mnt/p30/d: projid = 42
/mnt/p30/d/f1: projid = 0
/mnt/p30/d/p1: projid = 42
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              448          0          0     00 [--------]          7          0          0     00 [--------]

Project quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
Project ID       Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                 320          0          0     00 [--------]          8          0          0     00 [--------]
#42                128          0          0     00 [--------]          2          0          0     00 [--------]
```

## quotacheck가 센 사용량 (non-metadir fs)

**관찰 출력**

```text
######## fs A: -m metadir=0
== A4 quota options again
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 7c3e5434-5e63-41e4-94cc-eada42cd363e
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              448          0          0     00 [--------]          7          0          0     00 [--------]

Project quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
Project ID       Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                 320          0          0     00 [--------]          8          0          0     00 [--------]
#42                128          0          0     00 [--------]          2          0          0     00 [--------]
```

## quotacheck가 센 사용량 (metadir fs)

**관찰 출력**

```text
######## fs B: -m metadir=1
== B5 quota options
## mount [-o uquota,pquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,prjquota
XFS (loop0): Mounting V5 Filesystem 9a347436-f0b6-4435-ba33-cb29e8509831
XFS (loop0): Ending clean mount
XFS (loop0): Quotacheck needed: Please wait.
XFS (loop0): Quotacheck: Done.
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          1          0          0     00 [--------]
#1000              448          0          0     00 [--------]          7          0          0     00 [--------]

Project quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
Project ID       Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                 320          0          0     00 [--------]          6          0          0     00 [--------]
#42                128          0          0     00 [--------]          2          0          0     00 [--------]
```

## state, disable -u, off -u

**관찰 출력**

```text
## mount [-o uquota,pqnoenforce] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,pqnoenforce
User quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: ON
Group quota state on /mnt/p30 (/dev/loop0)
  Accounting: OFF
  Enforcement: OFF
Project quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: OFF
## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x40f
## mount [-o uquota,pqnoenforce] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota,pqnoenforce
== disable -u
User quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: OFF
rc=0
== off -u
User quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: OFF
rc=0
XFS (loop0): disabling of quota accounting not supported.
== state after off
User quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: OFF
Group quota state on /mnt/p30 (/dev/loop0)
  Accounting: OFF
  Enforcement: OFF
Project quota state on /mnt/p30 (/dev/loop0)
  Accounting: ON
  Enforcement: OFF
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              256          0          0     00 [--------]          1          0          0     00 [--------]

## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x40d
```

## xattr block이 사용량에 들어간다

**관찰 출력**

```text
== E1 xattr blocks
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                     
User ID          Used       Soft       Hard    Warn/Grace     
---------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]
#1000                4          0          0     00 [--------]

setfattr rc=0
/mnt/p30/x:
 EXT: FILE-OFFSET      BLOCK-RANGE      AG AG-OFFSET        TOTAL
   0: [0..127]:        192..319          0 (192..319)         128
fsxattr.nextents = 1
fsxattr.naextents = 1
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                     
User ID          Used       Soft       Hard    Warn/Grace     
---------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]
#1000               68          0          0     00 [--------]
```

## block hard limit과 grace

**관찰 출력**

```text
== E2 user limits bsoft=512k bhard=1m ihard=4
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000               68        512       1024     00 [--------]          2          0          4     00 [--------]
pwrite: Disk quota exceeded
pwrite rc=1
-rw------- 1 1000 1000 917504 Oct 23 01:12 /mnt/p30/u/big
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              964        512       1024     00  [7 days]          3          0          4     00 [--------]
touch: cannot touch '/mnt/p30/u/i1': Disk quota exceeded
touch i1 rc=1
touch: cannot touch '/mnt/p30/u/i2': Disk quota exceeded
touch i2 rc=1
touch: cannot touch '/mnt/p30/u/i3': Disk quota exceeded
touch i3 rc=1
touch: cannot touch '/mnt/p30/u/i4': Disk quota exceeded
touch i4 rc=1
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              964        512       1024     00  [7 days]          3          0          4     00 [--------]
```

## id 0 limit이 default로 걸린다

**관찰 출력**

```text
== E3 default limit via id 0 (bhard=256k)
/mnt/p30/o/u1001: Disk quota exceeded
uid1001 pwrite rc=1
wrote 1048576/1048576 bytes at offset 0
1 MiB, 16 ops; 0.0000 sec (23.819 GiB/sec and 390243.9024 ops/sec)
root pwrite rc=0
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                1024          0        256     00 [--none--]          5          0          0     00 [--------]
#1000              964        512       1024     00  [7 days]          3          0          4     00 [--------]
```

## project limit에서는 root도 ENOSPC

**관찰 출력**

```text
== E4 project 42 bhard=512k, written by root
pwrite: No space left on device
root pwrite in project rc=1
Project quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
Project ID       Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                1988          0          0     00 [--------]          8          0          0     00 [--------]
#42                512          0        512     00 [--------]          2          0          0     00 [--------]
```

## limit이 disk의 dquot에 들어간 모양

**관찰 출력**

```text
== dquot -u 1000 on disk
diskdq.magic = 0x4451
diskdq.version = 0x1
diskdq.type = 0x81
diskdq.id = 1000
diskdq.blk_hardlimit = 256
diskdq.blk_softlimit = 128
diskdq.ino_hardlimit = 4
diskdq.ino_softlimit = 0
diskdq.bcount = 241
diskdq.icount = 3
diskdq.itimer = 0
diskdq.btimer = Fri Oct 30 01:12:24 2026
diskdq.iwarns = 0
diskdq.bwarns = 0
diskdq.rtb_hardlimit = 0
diskdq.rtb_softlimit = 0
diskdq.rtbcount = 0
diskdq.rtbtimer = 0
diskdq.rtbwarns = 0
crc = 0xe5aef996 (correct)
lsn = 0x100000025
uuid = fa6b75f7-6882-4965-83cd-335c6654aee0
== dquot -u 0 on disk
diskdq.magic = 0x4451
diskdq.version = 0x1
diskdq.type = 0x1
diskdq.id = 0
diskdq.blk_hardlimit = 64
diskdq.blk_softlimit = 0
diskdq.ino_hardlimit = 0
diskdq.ino_softlimit = 0
diskdq.bcount = 384
diskdq.icount = 7
diskdq.itimer = 0
diskdq.btimer = 0
diskdq.iwarns = 0
diskdq.bwarns = 0
diskdq.rtb_hardlimit = 0
diskdq.rtb_softlimit = 0
diskdq.rtbcount = 0
diskdq.rtbtimer = 0
diskdq.rtbwarns = 0
crc = 0xaa4b0c6f (correct)
lsn = 0x100000033
uuid = fa6b75f7-6882-4965-83cd-335c6654aee0
```

## create부터 freeze까지 tracepoint

**관찰 출력**

```text
== step1 create (uid 1000)
touch-1882 xfs_dqread: dev 7:0 id 0x3e8 type USER flags  nrefs 0 res_bc 0x0 res_rtbc 0x0 res_ic 0x0 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x0]
touch-1882 xfs_dqalloc: dev 7:0 id 0x3e8 type USER flags  nrefs 0 res_bc 0x0 res_rtbc 0x0 res_ic 0x0 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x0]
touch-1882 xfs_trans_commit: dev 7:0 trans 614bf4a8 flags 0x25 caller xfs_trans_roll+0x38/0xe0
touch-1882 xfs_trans_commit: dev 7:0 trans 614bf4a9 flags 0x85 caller xfs_trans_commit+0x53/0x80
touch-1882 xfs_dqget_miss: dev 7:0 id 0x3e8 type USER|BIGTIME flags  nrefs 1 res_bc 0x0 res_rtbc 0x0 res_ic 0x0 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x0]
touch-1882 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags  field RES_REGBLKS delta 90
touch-1882 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags  field RES_INOS delta 1
touch-1882 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags  field ICOUNT delta 1
touch-1882 xfs_trans_commit: dev 7:0 trans dbfcbf01 flags 0x5 caller xfs_trans_commit+0x53/0x80
touch-1882 xfs_trans_apply_dquot_deltas_after: dev 7:0 id 0x3e8 type USER|BIGTIME flags  nrefs 2 res_bc 0x5a res_rtbc 0x0 res_ic 0x1 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x1]
touch-1882 xfs_trans_commit: dev 7:0 trans ce519eb6 flags 0x1 caller xfs_trans_commit+0x53/0x80
lines=11
== step2 buffered pwrite 256k, no fsync
xfs_io-1913 xfs_trans_commit: dev 7:0 trans 5645981 flags 0x1 caller xfs_trans_commit+0x53/0x80
xfs_io-1913 xfs_trans_commit: dev 7:0 trans 12ea565b flags 0x5 caller xfs_trans_commit+0x53/0x80
lines=2
== step3 fsync
xfs_io-1944 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field DELBCOUNT delta 64
xfs_io-1944 xfs_trans_commit: dev 7:0 trans 3c16ad5b flags 0x35 caller xfs_trans_roll+0x38/0xe0
xfs_io-1944 xfs_trans_apply_dquot_deltas_after: dev 7:0 id 0x3e8 type USER|BIGTIME flags DIRTY nrefs 1 res_bc 0x40 res_rtbc 0x0 res_ic 0x1 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x1]
xfs_io-1944 xfs_trans_commit: dev 7:0 trans 3c16ad5c flags 0x95 caller xfs_trans_commit+0x53/0x80
kworker/3:1-71 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field RES_REGBLKS delta 12
kworker/3:1-71 xfs_trans_commit: dev 7:0 trans d3abd11c flags 0x35 caller xfs_trans_roll+0x38/0xe0
kworker/3:1-71 xfs_trans_commit: dev 7:0 trans d3abd11d flags 0x95 caller xfs_trans_commit+0x53/0x80
lines=7
== step4 direct write 64k to a new file
xfs_io-1975 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field RES_REGBLKS delta 90
xfs_io-1975 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field RES_INOS delta 1
xfs_io-1975 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field ICOUNT delta 1
xfs_io-1975 xfs_trans_commit: dev 7:0 trans 5c62f2eb flags 0x5 caller xfs_trans_commit+0x53/0x80
xfs_io-1975 xfs_trans_apply_dquot_deltas_after: dev 7:0 id 0x3e8 type USER|BIGTIME flags DIRTY nrefs 3 res_bc 0x9a res_rtbc 0x0 res_ic 0x2 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x2]
xfs_io-1975 xfs_trans_commit: dev 7:0 trans b791f394 flags 0x1 caller xfs_trans_commit+0x53/0x80
xfs_io-1975 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field RES_REGBLKS delta 7
xfs_io-1975 xfs_trans_mod_dquot: dev 7:0 dquot id 0x3e8 type USER|BIGTIME flags DIRTY field BCOUNT delta 1
xfs_io-1975 xfs_trans_commit: dev 7:0 trans cdc6d4f2 flags 0x25 caller xfs_trans_roll+0x38/0xe0
xfs_io-1975 xfs_trans_apply_dquot_deltas_after: dev 7:0 id 0x3e8 type USER|BIGTIME flags DIRTY nrefs 2 res_bc 0x47 res_rtbc 0x0 res_ic 0x2 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x2]
xfs_io-1975 xfs_trans_commit: dev 7:0 trans cdc6d4f3 flags 0x85 caller xfs_trans_commit+0x53/0x80
lines=134
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              320          0          0     00 [--------]          2          0          0     00 [--------]

== step5 freeze and thaw
xfsaild/loop0-1852 xfs_dqflush: dev 7:0 id 0x3e8 type USER|BIGTIME flags DIRTY nrefs 2 res_bc 0x50 res_rtbc 0x0 res_ic 0x2 bhardlimit 0x0 bsoftlimit 0x0 rtbhardlimit 0x0 rtbsoftlimit 0x0 icnt 0x2]
xfs_io-2009 xfs_trans_commit: dev 7:0 trans 15b86b03 flags 0x29 caller xfs_trans_commit+0x53/0x80
xfs_io-2009 xfs_trans_commit: dev 7:0 trans f02fa13f flags 0x29 caller xfs_trans_commit+0x53/0x80
lines=3
```

## shutdown 뒤의 DQUOT item과 recovery

**관찰 출력**

```text
== step6 more changes, then shutdown -f
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              448          0          0     00 [--------]          4          0          0     00 [--------]

## umount rc=0
== logprint -t, DQUOT items
logprint lines=54
1
	DQUOT: #regs:2  blkno:384  boffset:1360 id: 1000
== remount (log recovery)
## mount [-o uquota] rc=0 opts=rw,relatime,inode64,logbufs=8,logbsize=32k,usrquota
XFS (loop0): Mounting V5 Filesystem ec09188d-5c29-4252-96a3-cee2a64fe68e
XFS (loop0): Starting recovery (logdev: internal)
XFS (loop0): Ending recovery (logdev: internal)
User quota on /mnt/p30 (/dev/loop0)
                               Blocks                                          Inodes                     
User ID          Used       Soft       Hard    Warn/Grace           Used       Soft       Hard    Warn/ Grace     
---------- -------------------------------------------------- -------------------------------------------------- 
#0                   0          0          0     00 [--------]          3          0          0     00 [--------]
#1000              448          0          0     00 [--------]          4          0          0     00 [--------]

## umount rc=0
uquotino = 131
gquotino = null
qflags = 0x7
```

