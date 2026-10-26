# shutdown, error 처리, health: 원본 출력

> 본문은 ../31-shutdown-health.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## xfs_io shutdown 기본 모드(NOLOGFLUSH), dmesg, shutdown 뒤 동작, healthmon, trace, remount

**관찰 출력**

```text
error_level=3 panic_mask=0
######## xfs_io shutdown []
sysfs dir: error log stats 
shutdown rc=0
XFS (loop0): User initiated shutdown received.
XFS (loop0): Log I/O Error (0x6) detected at xfs_fs_goingdown+0x55/0xb0 (fs/xfs/xfs_fsops.c:465).  Shutting down filesystem.
XFS (loop0): Please unmount the filesystem and rectify the problem(s)
-- after shutdown
stat: cannot statx '/mnt/p31/d/f1': Input/output error
stat rc=1
ls: cannot access '/mnt/p31/d': Input/output error
ls rc=2
cat: /mnt/p31/d/f1: Input/output error
cat rc=1
touch: cannot touch '/mnt/p31/d/x': Input/output error
touch rc=1
/mnt/p31: Input/output error
spaceman health rc=1
umount rc=0
XFS (loop0): Unmounting Filesystem 5ab09fed-dc82-4c95-a875-b3abfc7ad92a
healthmon rc=0
/mnt/p31: monitoring started
/mnt/p31: filesystem shut down due to log I/O error, forced unmount
/mnt/p31: filesystem unmounted
xfs_io-11368   [002] .....   321.733549: xfs_healthmon_create: dev 7:0 flags verbose format v0
xfs_io-11372   [002] .....   322.738661: xfs_force_shutdown: dev 7:0 tag none|logerror flags log_io|force_umount file fs/xfs/xfs_fsops.c line_num 465
xfs_io-11372   [002] .l...   322.741414: xfs_healthmon_report_shutdown: dev 7:0 shutdown_flags log_io|force_umount
umount-11391   [004] .....   322.752303: xfs_healthmon_report_unmount: dev 7:0 events 0 lost_prev? 0
    log tail: 2 head: 10 state: <DIRTY>
remount rc=0
XFS (loop0): Mounting V5 Filesystem 5ab09fed-dc82-4c95-a875-b3abfc7ad92a
XFS (loop0): Starting recovery (logdev: internal)
XFS (loop0): Ending recovery (logdev: internal)
65536 f1
65536 f2
65536 f3
```

## xfs_io shutdown -f (LOGFLUSH), dmesg, shutdown 뒤 동작, healthmon, trace, remount

**관찰 출력**

```text
######## xfs_io shutdown [-f]
sysfs dir: error log stats 
shutdown rc=0
XFS (loop0): User initiated shutdown received.
XFS (loop0): Metadata I/O Error (0x4) detected at xfs_fs_goingdown+0x6f/0xb0 (fs/xfs/xfs_fsops.c:462).  Shutting down filesystem.
XFS (loop0): Please unmount the filesystem and rectify the problem(s)
-- after shutdown
stat: cannot statx '/mnt/p31/d/f1': Input/output error
stat rc=1
ls: cannot access '/mnt/p31/d': Input/output error
ls rc=2
cat: /mnt/p31/d/f1: Input/output error
cat rc=1
touch: cannot touch '/mnt/p31/d/x': Input/output error
touch rc=1
/mnt/p31: Input/output error
spaceman health rc=1
umount rc=0
XFS (loop0): Unmounting Filesystem a3aa50e2-0c1e-48a4-a82a-4a7736711117
healthmon rc=0
/mnt/p31: monitoring started
/mnt/p31: filesystem shut down due to forced unmount
/mnt/p31: filesystem unmounted
xfs_io-11462   [003] .....   322.840082: xfs_healthmon_create: dev 7:0 flags verbose format v0
xfs_io-11466   [004] .....   323.844876: xfs_force_shutdown: dev 7:0 tag none|ioerror flags force_umount file fs/xfs/xfs_fsops.c line_num 462
xfs_io-11466   [004] .....   323.847564: xfs_healthmon_report_shutdown: dev 7:0 shutdown_flags force_umount
umount-11485   [007] .....   323.859409: xfs_healthmon_report_unmount: dev 7:0 events 0 lost_prev? 0
    log tail: 2 head: 15 state: <DIRTY>
remount rc=0
XFS (loop0): Mounting V5 Filesystem a3aa50e2-0c1e-48a4-a82a-4a7736711117
XFS (loop0): Starting recovery (logdev: internal)
XFS (loop0): Ending recovery (logdev: internal)
65536 f1
65536 f2
65536 f3
0 newfile
0 unsynced
```

## error 설정 기본값, 범위 검사, remount 뒤 값

**관찰 출력**

```text
######## step 1 defaults and range checks
./fail_at_unmount:1
./metadata/EIO/max_retries:-1
./metadata/EIO/retry_timeout_seconds:-1
./metadata/ENODEV/max_retries:0
./metadata/ENODEV/retry_timeout_seconds:0
./metadata/ENOSPC/max_retries:-1
./metadata/ENOSPC/retry_timeout_seconds:-1
./metadata/default/max_retries:-1
./metadata/default/retry_timeout_seconds:-1
sh: 1: echo: echo: I/O error
write metadata/EIO/max_retries=-2 rc=1 now=-1
sh: 1: echo: echo: I/O error
write metadata/EIO/retry_timeout_seconds=86401 rc=1 now=-1
sh: 1: echo: echo: I/O error
write fail_at_unmount=2 rc=1 now=1
write metadata/EIO/retry_timeout_seconds=86400 rc=0 now=86400
write metadata/EIO/max_retries=7 rc=0 now=7
-- after remount
./fail_at_unmount:1
./metadata/EIO/max_retries:-1
./metadata/EIO/retry_timeout_seconds:-1
```

## async write 실패, 무한 retry 10초, max_retries=2로 변경, shutdown, remount

**관찰 출력**

```text
######## step 3 async write retry, then max_retries=2
table: 0 2097152 error 
-- after 10 s with EIO max_retries=-1: iodone_async=0 shutdown=0
write metadata/EIO/max_retries=2 rc=0 now=2
shutdown seen within 21 s after max_retries=2
      1 XFS (dm-0): metadata I/O error in "__xfs_buf_ioend+0x21f/0x5a0" at daddr 0x1 len 1 error 5
      1 XFS (dm-0): Failing async write on buffer block 0xc0. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0xe0. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x100. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x120. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x140. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x160. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x78. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x180. Retrying async write.
      1 XFS (dm-0): Failing async write on buffer block 0x1a0. Retrying async write.
      1 XFS (dm-0): Metadata I/O Error (0x1) detected at __xfs_buf_ioend+0x531/0x5a0 (fs/xfs/xfs_buf.c:1058).  Shutting down filesystem.
      1 XFS (dm-0): Please unmount the filesystem and rectify the problem(s)
trace lines=288
      8 commit overrun: 0
      8 overrun: 0
     26  xfs_buf_error_relse: 
     79  xfs_buf_iodone_async: 
    182  xfs_buf_ioerror: 
      1  xfs_force_shutdown: 
-- iodone_async per bno (top 8)
-- first error_relse and shutdown lines
kworker/2:1-71      [002] .....   356.169325: xfs_buf_error_relse: dev 251:0 daddr 0x1 bbcount 0x1 hold 2 pincount 0 lock 0 flags ASYNC|DONE|STALE|WRITE_FAIL|KMEM bufops xfs_agf_buf_ops+0x0/0x40 caller xfs_buf_ioend_work+0x19/0x40
kworker/2:1-71      [002] .....   356.169328: xfs_buf_error_relse: dev 251:0 daddr 0x2 bbcount 0x1 hold 2 pincount 0 lock 0 flags ASYNC|DONE|STALE|WRITE_FAIL|KMEM bufops xfs_agi_buf_ops+0x0/0x40 caller xfs_buf_ioend_work+0x19/0x40
kworker/2:1-71      [002] .....   356.166396: xfs_force_shutdown: dev 251:0 tag none|ioerror flags metadata_io file fs/xfs/xfs_buf.c line_num 1058
umount rc=0
table: 0 2097152 linear 7:0 0
remount rc=0
      1 XFS (dm-0): Mounting V5 Filesystem f4be1f73-4666-4dc2-85f3-8627a6eb5415
      1 XFS (dm-0): Starting recovery (logdev: internal)
      1 XFS (dm-0): Ending recovery (logdev: internal)
files after recovery=500
```

## xfs/006 모양, dm error table을 적용한 뒤 unmount

**관찰 출력**

```text
######## step 2 xfs/006 shape: retry forever, fail_at_unmount=1, error table (lockfs), umount
./fail_at_unmount:1
./metadata/EIO/max_retries:-1
./metadata/EIO/retry_timeout_seconds:-1
fsstress rc=0
table: 0 2097152 error 
umount rc=0 secs=0
      1 XFS (dm-0): Unmounting Filesystem 39ad50c9-92ab-4bf9-a8ad-8350e939c4fc
      1 XFS (dm-0): log I/O error -5
      1 XFS (dm-0): Filesystem has been shut down due to log error (0x2).
      1 XFS (dm-0): Please unmount the filesystem and rectify the problem(s).
trace lines=2
      8 commit overrun: 0
      8 overrun: 0
      1  xfs_buf_error_relse: 
      1  xfs_buf_ioerror: 
-- iodone_async per bno (top 8)
-- first error_relse and shutdown lines
kworker/3:1H-106     [003] .....   325.885540: xfs_buf_error_relse: dev 251:0 daddr 0x0 bbcount 0x1 hold 3 pincount 0 lock 0 flags ASYNC|DONE|STALE|KMEM bufops xfs_sb_buf_ops+0x0/0x40 caller xfs_buf_ioend_fail+0xdc/0x100
table: 0 2097152 linear 7:0 0
remount rc=0
      1 XFS (dm-0): Mounting V5 Filesystem 39ad50c9-92ab-4bf9-a8ad-8350e939c4fc
      1 XFS (dm-0): Starting recovery (logdev: internal)
      1 XFS (dm-0): Ending recovery (logdev: internal)
repair -n rc=0
```

## errortag log_bad_crc=1, 주입, shutdown, remount의 torn write 처리

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
loop=/dev/loop0
errortag files=47 log_bad_crc=0 panic_mask=0
log_bad_crc=1
fsync rc=1
touch rc=1
XFS (loop0): Injecting error at file fs/xfs/xfs_log.c, line 1761, on filesystem "loop0"
XFS (loop0): Intentionally corrupted log record at LSN 0x100000007. Shutdown imminent.
XFS (loop0): log I/O error -5
XFS (loop0): Filesystem has been shut down due to log error (0x2).
XFS (loop0): Please unmount the filesystem and rectify the problem(s).
xfs_force_shutdown events=0
umount rc=0
XFS (loop0): Unmounting Filesystem 682a885b-21fe-4e9a-b6e9-270c7b212bb2
    log tail: 2 head: 13 state: <DIRTY>
remount rc=0
XFS (loop0): Mounting V5 Filesystem 682a885b-21fe-4e9a-b6e9-270c7b212bb2
XFS (loop0): Torn write (CRC failure) detected at log block 0x7. Truncating head block from 0xd.
XFS (loop0): Starting recovery (logdev: internal)
XFS (loop0): Ending recovery (logdev: internal)
errortag log_bad_crc after remount=0
65536 before
second mount rc=0
XFS (loop0): Mounting V5 Filesystem 682a885b-21fe-4e9a-b6e9-270c7b212bb2
XFS (loop0): Ending clean mount
done
fsync: Input/output error
touch: cannot touch '/mnt/p31c/x': Input/output error
```

## scrub 전후의 xfs_spaceman health, AG 1 superblock 손상 뒤

**관찰 출력**

```text
######## A: xfs/506 shape
Health status has not been collected for this filesystem.
Please run xfs_scrub(8) to remedy this situation.
[health] rc=0
scrub rc=0
AG 0 superblock: ok
AG 0 AGF header: ok
AG 0 AGFL header: ok
AG 0 AGI header: ok
AG 0 free space by block btree: ok
AG 0 free space by length btree: ok
AG 0 inode btree: ok
AG 0 free inode btree: ok
AG 0 reverse mappings btree: ok
AG 0 reference count btree: ok
[health -a 0] rc=0
Allowing fuzz of corrupted data with good CRC
magicnum = 0x4204fe75
mount rc=0
Health status has not been collected for this filesystem.
Please run xfs_scrub(8) to remedy this situation.
[health -a 1] rc=0
scrub rc=1
Info: /mnt/p31h: Filesystem has errors, skipping connectivity checks.
111.7MiB data used;  3 inodes used.
64.2MiB data found; 3 inodes found.
/mnt/p31h: corruptions found: 1
/mnt/p31h: Re-run xfs_scrub without -n.
AG 1 superblock: unhealthy
AG 1 AGF header: ok
AG 1 AGFL header: ok
AG 1 AGI header: ok
AG 1 free space by block btree: ok
AG 1 free space by length btree: ok
AG 1 inode btree: ok
AG 1 free inode btree: ok
AG 1 reverse mappings btree: ok
AG 1 reference count btree: ok
[health -a 1] rc=0
filesystem summary counters: ok
filesystem inode link counts: ok
[health -f] rc=0
umount rc=0
XFS (loop0): Mounting V5 Filesystem e22b01bd-0613-415d-87f9-cc21e6f98b65
XFS (loop0): Ending clean mount
XFS (loop0): Superblock has bad magic number 0x4204fe75. Not an XFS filesystem?
XFS (loop0): metadata I/O error in "xfs_sb_read_secondary+0x8d/0xf0" at daddr 0x80000 len 1 error 22
XFS (loop0): Unmounting Filesystem e22b01bd-0613-415d-87f9-cc21e6f98b65
XFS (loop0): Uncorrected metadata errors detected; please run xfs_repair.
```

## directory block owner 손상, touch의 실패, health, healthmon의 SICK event

**관찰 출력**

```text
######## B: xfs/655 shape
mkfs agcount=1 rc=0
isize=512 entries=2 dir ino=131203
bhdr.hdr.owner = 133220
bhdr.hdr.magic = 0x58444233
bhdr.hdr.owner = 133220
touch: cannot touch '/mnt/p31h/a/farts': Structure needs cleaning
touch rc=1
/mnt/p31h/a directory: unhealthy
[health /mnt/p31h/a] rc=0
Health status has not been collected for this filesystem.
Please run xfs_scrub(8) to remedy this situation.
[health -a 0] rc=0
umount rc=0
healthmon rc=0
-- healthmon (runtime)
/mnt/p31h: monitoring started
/mnt/p31h ino 131203 gen 0xc4710cf6 directory: sick
/mnt/p31h ino 131203 gen 0xc4710cf6 directory: sick
/mnt/p31h: filesystem unmounted
XFS (loop0): Mounting V5 Filesystem 928a4c51-2dee-4177-aea2-b1676c0cda19
XFS (loop0): Ending clean mount
XFS (loop0): Metadata corruption detected at xfs_dir3_block_read+0xa9/0xd0, xfs_dir3_block block 0x20078
XFS (loop0): Unmount and run xfs_repair
XFS (loop0): Metadata corruption detected at xfs_dir3_block_read+0xa9/0xd0, xfs_dir3_block block 0x20078
XFS (loop0): Unmount and run xfs_repair
XFS (loop0): Unmounting Filesystem 928a4c51-2dee-4177-aea2-b1676c0cda19
trace lines=6
      1  xfs_healthmon_create: 
      2  xfs_healthmon_report_inode: 
      1  xfs_healthmon_report_unmount: 
      2  xfs_inode_mark_sick: 
xfs_io-13154   [006] .....   359.988789: xfs_healthmon_create: dev 7:0 flags verbose format v0
touch-13156   [002] .....   360.992695: xfs_inode_mark_sick: dev 7:0 ino 0x20083 flags 0x10
touch-13156   [002] .l...   360.992700: xfs_healthmon_report_inode: dev 7:0 type sick domain inode oldmask 0x0 newmask 0x10 imask 0x10 ino 0x20083 gen 0xc4710cf6
touch-13156   [002] .....   360.996582: xfs_inode_mark_sick: dev 7:0 ino 0x20083 flags 0x10
touch-13156   [002] .l...   360.996584: xfs_healthmon_report_inode: dev 7:0 type sick domain inode oldmask 0x10 newmask 0x10 imask 0x10 ino 0x20083 gen 0xc4710cf6
umount-13166   [006] .....   361.004263: xfs_healthmon_report_unmount: dev 7:0 events 0 lost_prev? 0
```

## 같은 손상에 scrub을 실행할 때, health와 healthmon의 CORRUPT, HEALTHY event

**관찰 출력**

```text
-- B2: scrub under healthmon
scrub rc=1
Info: /mnt/p31h: Filesystem has errors, skipping connectivity checks.
114.5MiB data used;  6 inodes used.
64.1MiB data found; 6 inodes found.
/mnt/p31h: corruptions found: 2
/mnt/p31h: Re-run xfs_scrub without -n.
/mnt/p31h/a inode core: ok
/mnt/p31h/a data fork: ok
/mnt/p31h/a extended attribute fork: ok
/mnt/p31h/a directory: unhealthy
/mnt/p31h/a extended attributes: ok
/mnt/p31h/a parent pointers: unhealthy
[health /mnt/p31h/a] rc=0
healthmon rc=0
-- healthmon (scrub)
/mnt/p31h: monitoring started
/mnt/p31h ino 131203 gen 0xc4710cf6 core: healthy
/mnt/p31h ino 131203 gen 0xc4710cf6 datafork, attrfork: healthy
/mnt/p31h ino 131203 gen 0xc4710cf6 directory: sick
/mnt/p31h ino 131203 gen 0xc4710cf6 directory: corrupt
/mnt/p31h ino 131203 gen 0xc4710cf6 xattr: healthy
/mnt/p31h ino 131203 gen 0xc4710cf6 directory: sick
/mnt/p31h ino 131203 gen 0xc4710cf6 parent: corrupt
/mnt/p31h fscounters: healthy
/mnt/p31h: filesystem unmounted
```

