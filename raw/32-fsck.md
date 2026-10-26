# xfs_repair와 online fsck: 원본 출력

> 본문은 ../32-fsck.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## 깨끗한 image에서 xfs_repair -n

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
dev=/dev/loop0
scrub=/opt/xfsprogs-7.1.1/sbin/xfs_scrub
xfs_scrub version 7.1.1
xfs_repair version 7.1.1
== xfs_repair -n
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 2
        - agno = 1
        - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
```

## rw로 mount한 device에 xfs_repair -n과 xfs_info

**관찰 출력**

```text
== now mounted rw
== xfs_repair -n
rc=1
xfs_repair: /dev/loop0 contains a mounted and writable filesystem

fatal error -- couldn't initialize XFS library
meta-data=/dev/loop0             isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=1
         =                       exchange=1   metadir=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
         =                       rgcount=0    rgsize=0 extents
         =                       zoned=0      start=0 reserved=0
```

## 깨끗한 image에서 xfs_scrub -n

**관찰 출력**

```text
== xfs_scrub -n
rc=0
Info: AG 1 superblock: Optimization is possible.
Info: AG 2 superblock: Optimization is possible.
Info: AG 3 superblock: Optimization is possible.
Info: /mnt/p32: Optimizations of extended attributes are possible.
163.5MiB data used;  38.6K inodes used.
93.3MiB data found; 38.6K inodes found.
== xchk_done count by type
      4 agf
      4 agfl
      4 agi
  38551 bmapbta
  38551 bmapbtc
  38552 bmapbtd
      4 bnobt
      4 cntbt
    778 directory
    777 dirtree
      4 finobt
      1 fscounters
      1 grpquota
      1 healthy
      4 inobt
  38552 inode
      1 nlinks
  38552 parent
      1 prjquota
      1 probe
      1 quotacheck
      4 refcountbt
      1 rgsuper
      4 rmapbt
      1 rtbitmap
      1 rtrefcountbt
      1 rtrmapbt
      1 rtsummary
      5 sb
      3 symlink
      1 usrquota
  38552 xattr
== xchk_done count by error
  38463 -2
 194459 0
== xchk_done lines with nonempty flags (first 10)
       xfs_scrub-1529    [002] .....   125.050955: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (preen) error 0
       xfs_scrub-1522    [007] .....   125.050962: xchk_done: dev 7:0 ino 0x80 type sb agno 0x2 inum 0x0 gen 0x0 flags (preen) error 0
       xfs_scrub-1523    [005] .....   125.051053: xchk_done: dev 7:0 ino 0x80 type sb agno 0x3 inum 0x0 gen 0x0 flags (preen) error 0
       xfs_scrub-1537    [004] .....   125.099156: xchk_done: dev 7:0 ino 0x100198 type xattr agno 0x0 inum 0x100198 gen 0xec5cfaa4 flags (preen) error 0
       xfs_scrub-1533    [000] .....   125.148491: xchk_done: dev 7:0 ino 0x22c1 type xattr agno 0x0 inum 0x22c1 gen 0x92b17978 flags (preen) error 0
       xfs_scrub-1533    [000] .l...   125.151530: xchk_done: dev 7:0 ino 0x22c2 type xattr agno 0x0 inum 0x22c2 gen 0x3590db54 flags (preen) error 0
== overrun
      8 commit overrun: 0
      8 overrun: 0
```

## shutdown으로 만든 dirty log와 xfs_repair

**관찰 출력**

```text
== make a dirty log
== xfs_repair -n
rc=1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ALERT: The filesystem has valuable metadata changes in a log which is being
ignored because the -n option was used.  Expect spurious inconsistencies
which may be resolved by first mounting the filesystem to replay the log.
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 2
        - agno = 1
        - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
== xfs_repair 
rc=2
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ERROR: The filesystem has valuable metadata changes in a log which needs to
be replayed.  Mount the filesystem to replay the log, and unmount it before
re-running xfs_repair.  If the filesystem is a snapshot of a mounted
filesystem, you may need to give mount the nouuid option. If you are unable
to mount the filesystem, then use the -L option to destroy the log and
attempt a repair.  Note that destroying the log may cause corruption --
please attempt a mount of the filesystem before doing this.
```

## mount와 unmount로 log를 replay한 뒤의 xfs_repair

**관찰 출력**

```text
== replay by mount and umount
[  129.752959] XFS (loop0): Metadata I/O Error (0x4) detected at xfs_fs_goingdown+0x6f/0xb0 (fs/xfs/xfs_fsops.c:462).  Shutting down filesystem.
[  129.754681] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[  129.768301] XFS (loop0): Unmounting Filesystem 0ef0b5c2-41de-48a9-9829-c3d46154a46e
[  129.937273] XFS (loop0): Mounting V5 Filesystem 0ef0b5c2-41de-48a9-9829-c3d46154a46e
[  129.938135] XFS (loop0): Starting recovery (logdev: internal)
[  129.938401] XFS (loop0): Ending recovery (logdev: internal)
== xfs_repair -n
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 2
        - agno = 1
        - agno = 3
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
== xfs_repair 
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
        - agno = 2
        - agno = 3
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
done
exit=0
```

## 손상 전의 finobt와 xfs_repair -n

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
dev=/dev/loop0
meta-data=/dev/loop0             isize=512    agcount=2, agsize=131072 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
free_root=4
free_root = 4
free_level = 1
count = 1024
freecount = 53
numrecs = 16
recs[1-3] = [startino,holemask,count,freecount,free] 
1:[128,0,64,2,0x800000008] 
2:[192,0,64,2,0x800000008] 
3:[256,0,64,2,0x800000008]
== xfs_repair -n
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
```

## finobt record 손상 뒤 xfs_repair -n

**관찰 출력**

```text
== corrupt records (xfs/010 _corrupt_finobt_records)
recs[1].freecount = 70
recs[2].freecount = 0
recs[2].free = 0
recs[1-3] = [startino,holemask,count,freecount,free] 
1:[128,0,64,70,0x800000008] 
2:[192,0,64,0,0] 
3:[256,0,64,2,0x800000008]
== xfs_repair -n
rc=1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
finobt ir_freecount/free mismatch, inode chunk 0/128, freecount 70 nfree 2
finobt record with no free inodes, inode chunk 0/192
agi_freecount 53, counted 119 in ag 0 finobt
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Inode allocation btrees are too corrupted, skipping phases 6 and 7
No modify flag set, skipping filesystem flush and exiting.
```

## xfs_repair로 고친 뒤의 finobt

**관찰 출력**

```text
== xfs_repair 
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
finobt ir_freecount/free mismatch, inode chunk 0/128, freecount 70 nfree 2
finobt record with no free inodes, inode chunk 0/192
agi_freecount 53, counted 119 in ag 0 finobt
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
done
== xfs_repair -n
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
free_root after repair=87
numrecs = 16
recs[1-3] = [startino,holemask,count,freecount,free] 
1:[128,0,64,2,0x800000008] 
2:[192,0,64,2,0x800000008] 
3:[256,0,64,2,0x800000008]
```

## finobt root를 지운 뒤의 xfs_repair

**관찰 출력**

```text
== nuke finobt root (xfs/010 _corrupt_finobt_root)
Allowing write of corrupted data and bad CRC
free_root = 0
Allowing write of corrupted data and bad CRC
free_level = 0
== xfs_repair -n
rc=1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
Metadata CRC error detected at 0x5586b5cc2050, xfs_agi block 0x2/0x200
agi has bad CRC for ag 0
bad levels 0 for finobt root, agno 0
bad agbno 0 for finobt root, agno 0
bad finobt block count 1, saw 0
agi_freecount 53, counted 0 in ag 0 finobt
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
unknown block state, ag 0, blocks 87-87
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
== xfs_repair 
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
Metadata CRC error detected at 0x558760d1a050, xfs_agi block 0x2/0x200
agi has bad CRC for ag 0
bad levels 0 for finobt root, agno 0
bad agbno 0 for finobt root, agno 0
bad finobt block count 1, saw 0
agi_freecount 53, counted 0 in ag 0 finobt
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
unknown block state, ag 0, blocks 87-87
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
Phase 5 - rebuild AG headers and trees...
        - reset superblock...
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify and correct link counts...
done
== xfs_repair -n
rc=0
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
free_root = 87
free_level = 1
count = 1024
freecount = 53
exit=0
```

## mkfs, 정상 scrub, secondary superblock fuzz

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
dev=/dev/loop0
meta-data=/dev/loop0             isize=512    agcount=4, agsize=65536 blks
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
         =                       exchange=1   metadir=0
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1, parent=1
scrub=/opt/xfsprogs-7.1.1/sbin/xfs_scrub
xfs_scrub version 7.1.1
== repair -R probe (no output means supported)
probe rc=0
== clean pass (xfs/798 first scrub)
rc=0
114.9MiB data used;  3 inodes used.
64.2MiB data found; 3 inodes found.
/mnt/p32: optimizations made: 3.
dblocks = 262144
dblocks = 262144
== fuzz sb 1 dblocks
Allowing fuzz of corrupted data with good CRC
dblocks = 264161
SB sanity check failed
Metadata corruption detected at 0x55fbe307456d, xfs_sb block 0x80000/0x200
dblocks = 264161
```

## secondary superblock 손상에 xfs_scrub -n과 -p

**관찰 출력**

```text
== xfs_scrub -n
rc=1
Corruption: AG 1 superblock: Repairs are required.
Info: /mnt/p32: Filesystem has errors, skipping connectivity checks.
111.7MiB data used;  3 inodes used.
64.2MiB data found; 3 inodes found.
/mnt/p32: corruptions found: 1
/mnt/p32: Re-run xfs_scrub without -n.
-- trace: xrep events and type sb
xfs_scrub-4530    [004] .....   133.823947: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4532    [000] .....   133.823975: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4535    [002] .....   133.824140: xchk_done: dev 7:0 ino 0x80 type sb agno 0x2 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4534    [003] .....   133.824140: xchk_done: dev 7:0 ino 0x80 type sb agno 0x3 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4531    [006] .l...   133.840530: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (corrupt) error 0
== xfs_scrub -p
rc=1
Corruption: AG 1 superblock: Repairs are required.
Info: /mnt/p32: Filesystem has errors, skipping connectivity checks.
111.7MiB data used;  3 inodes used.
64.2MiB data found; 3 inodes found.
/mnt/p32: corruptions found: 1
/mnt/p32: Re-run xfs_scrub without -p.
-- trace: xrep events and type sb
xfs_scrub-4604    [000] .....   134.876409: xrep_attempt: dev 7:0 ino 0x80 type probe agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4604    [000] .....   134.876410: xrep_done: dev 7:0 ino 0x80 type probe agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4604    [000] .....   134.876510: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4611    [003] .....   134.876524: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4606    [002] .....   134.876572: xchk_done: dev 7:0 ino 0x80 type sb agno 0x3 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4608    [006] .....   134.876576: xchk_done: dev 7:0 ino 0x80 type sb agno 0x2 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4612    [006] .l...   134.888483: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (corrupt) error 0
```

## secondary superblock 손상에 인자 없이 xfs_scrub

**관찰 출력**

```text
== xfs_scrub 
rc=0
Info: AG 1 superblock: Attempting repair.
Repaired: AG 1 superblock: Repairs successful.
114.9MiB data used;  3 inodes used.
64.2MiB data found; 3 inodes found.
/mnt/p32: repairs made: 1.
-- trace: xrep events and type sb
xfs_scrub-4678    [005] .....   135.921735: xrep_attempt: dev 7:0 ino 0x80 type probe agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4678    [005] .....   135.921735: xrep_done: dev 7:0 ino 0x80 type probe agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4678    [005] .....   135.921854: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4679    [007] .....   135.921888: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4685    [000] .....   135.921936: xchk_done: dev 7:0 ino 0x80 type sb agno 0x2 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4681    [002] .....   135.921936: xchk_done: dev 7:0 ino 0x80 type sb agno 0x3 inum 0x0 gen 0x0 flags () error 0
xfs_scrub-4686    [004] .l...   135.934995: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (corrupt) error 0
xfs_scrub-4709    [004] .l...   135.947922: xrep_attempt: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4709    [004] .l...   135.947924: xrep_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_scrub-4709    [004] .l...   135.947936: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags (repair) error 0
xfs_scrub-4709    [004] .....   135.947951: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags () error 0
```

## secondary superblock 읽기 실패의 dmesg와 repair 뒤의 superblock

**관찰 출력**

```text
== dmesg since mark
[  133.743785] u32-mark-sb
[  133.765094] XFS (loop0): SB sanity check failed
[  133.765098] XFS (loop0): Metadata corruption detected at xfs_sb_read_verify+0x14a/0x170, xfs_sb block 0x80000 
[  133.766799] XFS (loop0): Unmount and run xfs_repair
[  133.767542] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[  133.780251] XFS (loop0): metadata I/O error in "xfs_sb_read_secondary+0x8d/0xf0" at daddr 0x80000 len 1 error 117
dblocks = 262144
dblocks = 262144
repair -n rc=0
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
exit=0
```

## AG 0 superblock 강제 재구성 요청

**관찰 출력**

```text
== xfs_io scrub sb 1, scrub sb 0, repair -R sb 0
repair: Operation not supported
rc=0
xfs_io-4780    [004] .....   136.982137: xchk_done: dev 7:0 ino 0x80 type sb agno 0x1 inum 0x0 gen 0x0 flags () error 0
xfs_io-4780    [004] .....   136.982138: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-4780    [004] .....   136.982141: xrep_attempt: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags (repair|rebuild) error 0
xfs_io-4780    [004] .....   136.982142: xrep_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags (repair|rebuild) error -95
xfs_io-4780    [004] .....   136.982143: xchk_done: dev 7:0 ino 0x80 type sb agno 0x0 inum 0x0 gen 0x0 flags (repair|rebuild) error -95
```

## rmapbt가 있는 fs에서 finobt를 online으로 고치기

**관찰 출력**

```text
==== rmapbt=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=1
free_root=4
recs[1].freecount = 70
recs[2].freecount = 0
recs[2].free = 0
mount rc=0
== xfs_io scrub finobt 0
Corruption detected.
rc=0
== xfs_io repair finobt 0
rc=0
== xfs_io scrub finobt 0
rc=0
== xfs_io scrub inobt 0
rc=0
-- trace
xfs_io-14643   [003] .....   622.183093: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-14643   [003] .....   622.183140: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (corrupt) error 0
xfs_io-14645   [004] .....   622.185686: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair) error 0
xfs_io-14645   [004] .....   622.185695: xrep_attempt: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_io-14645   [004] .l...   622.185991: xrep_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_io-14645   [004] .....   622.186219: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair) error 0
xfs_io-14647   [004] .....   622.188760: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-14647   [004] .....   622.188875: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-14649   [004] .....   622.191353: xchk_start: dev 7:0 ino 0x80 type inobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-14649   [004] .....   622.191482: xchk_done: dev 7:0 ino 0x80 type inobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
-- dmesg since mark
[  622.103086] u32-mark-rm1
xfs_repair -n rc=1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ALERT: The filesystem has valuable metadata changes in a log which is being
ignored because the -n option was used.  Expect spurious inconsistencies
which may be resolved by first mounting the filesystem to replay the log.
        - scan filesystem freespace and inode maps...
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Phase 6 - check inode connectivity...
        - traversing filesystem ...
        - traversal finished ...
        - moving disconnected inodes to lost+found ...
Phase 7 - verify link counts...
No modify flag set, skipping filesystem flush and exiting.
```

## rmapbt가 없는 fs에서 finobt를 online으로 고치기

**관찰 출력**

```text
==== rmapbt=0
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
free_root=4
recs[1].freecount = 70
recs[2].freecount = 0
recs[2].free = 0
mount rc=0
== xfs_io scrub finobt 0
Corruption detected.
rc=0
== xfs_io repair finobt 0
rc=0
== xfs_io scrub finobt 0
Corruption detected.
rc=0
== xfs_io scrub inobt 0
Cross-referencing failed.
rc=0
-- trace
xfs_io-15899   [007] .....   622.536041: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-15899   [007] .....   622.536076: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (corrupt) error 0
xfs_io-15901   [003] .....   622.538738: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair) error 0
xfs_io-15901   [003] .l...   622.538747: xrep_attempt: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error 0
xfs_io-15901   [003] .l...   622.538749: xrep_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error -95
xfs_io-15901   [003] .....   622.539970: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (repair|corrupt) error -95
xfs_io-15905   [003] .....   622.542764: xchk_start: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-15905   [003] .l...   622.542772: xchk_done: dev 7:0 ino 0x80 type finobt agno 0x0 inum 0x0 gen 0x0 flags (corrupt) error 0
xfs_io-15907   [000] .....   622.545731: xchk_start: dev 7:0 ino 0x80 type inobt agno 0x0 inum 0x0 gen 0x0 flags () error 0
xfs_io-15907   [006] .....   622.546016: xchk_done: dev 7:0 ino 0x80 type inobt agno 0x0 inum 0x0 gen 0x0 flags (xfail) error 0
-- dmesg since mark
[  622.457186] u32-mark-rm0
[  622.479777] XFS (loop0): Corruption not fixed during online repair.  Unmount and run xfs_repair.
xfs_repair -n rc=1
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
        - scan filesystem freespace and inode maps...
finobt ir_freecount/free mismatch, inode chunk 0/128, freecount 70 nfree 2
finobt record with no free inodes, inode chunk 0/192
agi_freecount 53, counted 119 in ag 0 finobt
        - found root inode chunk
Phase 3 - for each AG...
        - scan (but don't clear) agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
        - agno = 1
        - process newly discovered inodes...
Phase 4 - check for duplicate blocks...
        - setting up duplicate extent list...
        - check for inodes claiming duplicate blocks...
        - agno = 0
        - agno = 1
No modify flag set, skipping phase 5
Inode allocation btrees are too corrupted, skipping phases 6 and 7
No modify flag set, skipping filesystem flush and exiting.
repair: Operation not supported
exit=0
```

