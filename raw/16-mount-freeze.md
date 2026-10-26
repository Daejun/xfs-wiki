# mount, unmount, freeze: 원본 출력

> 본문은 ../16-mount-freeze.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## mount-clean: clean log를 mount할 때 log와 recovery 호출 순서

**관찰 출력**

```text
== mount-clean: trace lines=1324 calls=1087
-- log and recovery calls at any level (line in calls list)
425:   xfs_readsb() {
726:     xfs_log_mount() {
776:       xlog_recover() {
777:         xlog_find_tail();
886:     xfs_check_summary_counts() {
935:     xfs_log_mount_finish() {
```

## mount-dirty: dirty log를 mount할 때 log와 recovery 호출 순서

**관찰 출력**

```text
== mount-dirty: trace lines=1399 calls=1157
-- log and recovery calls at any level (line in calls list)
434:   xfs_readsb() {
735:     xfs_log_mount() {
785:       xlog_recover() {
786:         xlog_find_tail() {
790:         xlog_do_recover();
889:     xfs_check_summary_counts() {
950:     xfs_log_mount_finish() {
955:       xlog_recover_finish() {
957:         xfs_log_force();
979:       xfs_log_force() {
[  772.010610] XFS (loop0): Mounting V5 Filesystem ae2cc2cc-ff57-4341-9693-23c225d1f40d
[  772.013062] XFS (loop0): Starting recovery (logdev: internal)
[  772.013890] XFS (loop0): Ending recovery (logdev: internal)
files f=100
```

## umount-clean: clean unmount에서 log 호출 순서

**관찰 출력**

```text
== umount-clean: trace lines=2431 calls=1960
-- log and recovery calls at any level (line in calls list)
1: xfs_fs_sync_fs();
2: xfs_fs_sync_fs() {
3:   xfs_log_force() {
90:   xfs_unmountfs() {
282:       xfs_log_force() {
412:     xfs_log_unmount() {
413:       xfs_log_quiesce() {
419:         xfs_log_force() {
442:         xfs_sync_sb() {
461:         xfs_sync_sb() {
478:       xfs_log_unmount_write() {
480:         xfs_log_force() {
487:         xfs_log_reserve() {
```

## umount-shutdown: shutdown 뒤 unmount에서 log 호출 순서

**관찰 출력**

```text
== umount-shutdown: trace lines=2218 calls=1787
-- levels 0-5, xfs/xlog only
     xfs_log_unmount() {
       xfs_log_quiesce() {
         xfs_clear_incompat_log_features();
         xfs_log_force() {
           xlog_cil_force_seq();
         xfs_ail_push_all_sync() {
         xfs_buftarg_wait() {
         xfs_log_writable();
       xfs_log_unmount_write() {
         xfs_log_writable();
-- log and recovery calls at any level (line in calls list)
1: xfs_fs_sync_fs();
2: xfs_fs_sync_fs() {
3:   xfs_log_force() {
33:   xfs_unmountfs() {
225:       xfs_log_force() {
326:     xfs_log_unmount() {
327:       xfs_log_quiesce() {
333:         xfs_log_force() {
350:       xfs_log_unmount_write() {
```

## freeze: freeze와 thaw에서 log 호출 순서

**관찰 출력**

```text
== freeze: trace lines=75 calls=59
-- log and recovery calls at any level (line in calls list)
1: xfs_fs_sync_fs();
2: xfs_fs_sync_fs() {
3:   xfs_log_force() {
24: xfs_fs_freeze() {
29:   xfs_log_quiesce() {
32:     xfs_log_force();
39:     xfs_sync_sb();
42:     xfs_sync_sb();
45: xfs_fs_unfreeze() {
52:   xfs_log_work_queue() {
```

## 다섯 경우의 mount 메시지, event 수, logprint 상태

**관찰 출력**

```text
loop=/dev/loop0 dev=7:0
set_event=xfs:xfs_log_recover
[mkfs]     log tail: 2 head: 2 state: <CLEAN>
== 1 clean mount
mount  rc=0
[  800.470493] XFS (loop0): Mounting V5 Filesystem 7c08711e-4858-457a-9cca-5fc89a8687fe
[  800.471377] XFS (loop0): Ending clean mount
xfs_log_recover events=0
[clean umount]     log tail: 12 head: 12 state: <CLEAN>
== 2 dirty log, rw mount
[a after shutdown -f and umount]     log tail: 12 head: 80 state: <DIRTY>
mount  rc=0
[  800.545283] XFS (loop0): Mounting V5 Filesystem 7c08711e-4858-457a-9cca-5fc89a8687fe
[  800.546724] XFS (loop0): Starting recovery (logdev: internal)
[  800.547167] XFS (loop0): Ending recovery (logdev: internal)
xfs_log_recover events=1
mount-1762    [001] .....   800.604860: xfs_log_recover: dev 7:0 headblk 0x50 tailblk 0xc
files a=100
[after replay]     log tail: 86 head: 86 state: <CLEAN>
== 3 dirty log, norecovery without ro
[b after shutdown -f and umount]     log tail: 86 head: 158 state: <DIRTY>
mount: /mnt/p16rc: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
       dmesg(1) may have more information after failed mount system call.
mount -o norecovery rc=32
[  800.615032] XFS (loop0): no-recovery mounts must be read-only.
xfs_log_recover events=0
== 4 dirty log, ro,norecovery
mount -o ro,norecovery rc=0
[  800.657580] XFS (loop0): Mounting V5 filesystem 7c08711e-4858-457a-9cca-5fc89a8687fe in no-recovery mode. Filesystem will be inconsistent.
xfs_log_recover events=0
files b=0
[after ro,norecovery umount]     log tail: 86 head: 158 state: <DIRTY>
== 5 dirty log, ro
mount -o ro rc=0
[  800.703314] XFS (loop0): Mounting V5 Filesystem 7c08711e-4858-457a-9cca-5fc89a8687fe
[  800.704174] XFS (loop0): Starting recovery (logdev: internal)
[  800.704430] XFS (loop0): Ending recovery (logdev: internal)
xfs_log_recover events=1
mount-1882    [002] .....   800.762305: xfs_log_recover: dev 7:0 headblk 0x9e tailblk 0x56
files b=100
[after ro umount]     log tail: 168 head: 168 state: <CLEAN>
done
```

## freeze 전후의 superblock, sysfs log 값, unmount 뒤 log 상태

**관찰 출력**

```text
lazy-count=1
[mkfs] sb: icount = 64 ifree = 61 fdblocks = 245724 lsn = 0 
[mount] head=1:2 tail=1:2 rsv=0 wr=0 iused=3 bfree=232778
[touched] head=1:2 tail=1:2 rsv=438684 wr=438684 iused=504 bfree=232719
freeze rc=0
[frozen] head=1:326 tail=1:324 rsv=1024 wr=1024 iused=504 bfree=240911
[frozen] sb: icount = 512 ifree = 8 fdblocks = 240943 lsn = 0x100000144 
thaw rc=0
[thawed] head=1:326 tail=1:324 rsv=1024 wr=1024 iused=504 bfree=232719
[removed] head=1:326 tail=1:324 rsv=308244 wr=308244 iused=404 bfree=232727
[umount] sb: icount = 448 ifree = 44 fdblocks = 245673 lsn = 0x10000018a 
    log tail: 398 head: 398 state: <CLEAN>
done
```

