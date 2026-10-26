# 개발 과정: 원본 출력

> 본문은 ../34-development.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## xfs/122를 xfsprogs 6.6.0과 7.1.1로 실행한 출력

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
## indent: /usr/bin/indent
## distro mkfs: /usr/sbin/mkfs.xfs mkfs.xfs version 6.6.0  metadir_in_help=0
## xp71 mkfs: mkfs.xfs version 7.1.1  metadir_in_help=1
FSTYP=xfs
TEST_DEV=/dev/loop0
TEST_DIR=/mnt/p34t
######## A: distro xfsprogs
mkfs rc=0
FSTYP         -- xfs (non-debug)
PLATFORM      -- Linux/x86_64 pnfs-m 7.1.0-rc5-g8fde5d1d47f6 #1 SMP PREEMPT_DYNAMIC Wed Oct 21 17:14:58 KST 2026
xfs/122             0s
Ran: xfs/122
Passed all 1 tests
######## B: xfsprogs 7.1.1 via xp71
mkfs rc=0
FSTYP         -- xfs (non-debug)
PLATFORM      -- Linux/x86_64 pnfs-m 7.1.0-rc5-g8fde5d1d47f6 #1 SMP PREEMPT_DYNAMIC Wed Oct 21 17:14:58 KST 2026
xfs/122      0s ... [not run] struct size checks moved to libxfs/xfs_ondisk.h
Ran: xfs/122
Not run: xfs/122
Passed all 1 tests
-- notrun:
struct size checks moved to libxfs/xfs_ondisk.h
```

## LARP knob를 켜고 setxattr한 출력

**관찰 출력**

```text
7.1.0-rc5-xdebug-g8fde5d1d47f6
larp(before)=0
## after mkfs: features_log_incompat = 0
mount rc=0
A larp=0 setfattr rc=0 warn_lines=0
B1 larp=1 setfattr rc=0 warn_lines=1
B2 setfattr rc=0 warn_lines=1
umount rc=0
## after umount: features_log_incompat = 0
C after remount setfattr rc=0 warn_lines=2
# file: mnt/p34/f
user.a="1"
user.b="2"
user.c="3"
user.d="4"
## after second umount: features_log_incompat = 0
repair -n rc=0
## dmesg XFS lines
[    8.826615] XFS (loop0): Mounting V5 Filesystem a1160405-2eb0-4294-bff9-7c7a397e6168
[    8.828020] XFS (loop0): Ending clean mount
[    8.848271] XFS (loop0): EXPERIMENTAL logged extended attributes feature enabled.  Use at your own risk!
[    8.865128] XFS (loop0): Unmounting Filesystem a1160405-2eb0-4294-bff9-7c7a397e6168
[    8.875259] XFS (loop0): Mounting V5 Filesystem a1160405-2eb0-4294-bff9-7c7a397e6168
[    8.876040] XFS (loop0): Ending clean mount
[    8.879796] XFS (loop0): EXPERIMENTAL logged extended attributes feature enabled.  Use at your own risk!
[    8.891269] XFS (loop0): Unmounting Filesystem a1160405-2eb0-4294-bff9-7c7a397e6168
```

