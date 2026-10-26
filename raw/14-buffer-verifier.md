# buffer cache와 verifier: 원본 출력

> 본문은 ../14-buffer-verifier.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## buffer cache 통계와 구간별 event 수

**관찰 출력**

```text
stats[mounted] icluster 0 0 0 buf 37 21 19 19 0 18 0 0 18 
stats[A] icluster 0 0 0 buf 1265 37 1231 1229 0 34 0 0 26 
stats[B] icluster 0 0 0 buf 1265 37 1231 1229 0 34 0 0 26 
stats[C] icluster 0 7 201 buf 1267 37 1233 1229 0 34 0 0 26 
trace lines=8778
      8 commit overrun: 0
      8 overrun: 0
== events by phase
   1212 A xfs_buf_find:
   1228 A xfs_buf_get:
    840 A xfs_buf_item_format:
     15 A xfs_buf_item_pin:
    848 A xfs_buf_item_release:
   1220 A xfs_buf_read:
      8 A xfs_buf_submit:
    201 A xfs_inode_pin:
      8 A xfs_trans_get_buf:
   2070 A xfs_trans_log_buf:
   1045 A xfs_trans_read_buf:
     19 C xfs_buf_delwri_queue:
      2 C xfs_buf_find:
      2 C xfs_buf_get:
      2 C xfs_buf_item_format:
      2 C xfs_buf_item_pin:
     17 C xfs_buf_item_push:
      2 C xfs_buf_item_release:
     19 C xfs_buf_submit:
      2 C xfs_trans_log_buf:
== xfs_trans_read_buf in phase A: top daddr
    203 0x20
    203 0x18
    200 0x2
    175 0x78
     34 0x70
     32 0xe0
     32 0xc0
     32 0xa0
```

## 구간별 xfs_buf_submit의 ops와 flags

**관찰 출력**

```text
== xfs_buf_submit bufops and flags by phase
      2 A xfs_inode_buf_ops+0x0/0x30 READ
      1 A xfs_agfl_buf_ops+0x0/0x40 READ|KMEM
      1 A xfs_bnobt_buf_ops+0x0/0x40 READ
      1 A xfs_cntbt_buf_ops+0x0/0x40 READ
      1 A xfs_finobt_buf_ops+0x0/0x40 READ
      1 A xfs_inobt_buf_ops+0x0/0x30 READ
      1 A xfs_rmapbt_buf_ops+0x0/0x40 READ
      8 C xfs_inode_buf_ops+0x0/0x30 WRITE|ASYNC|DONE
      2 C xfs_sb_buf_ops+0x0/0x40 WRITE|ASYNC|DONE|KMEM
      1 C xfs_agf_buf_ops+0x0/0x40 WRITE|ASYNC|DONE|KMEM
      1 C xfs_agi_buf_ops+0x0/0x40 WRITE|ASYNC|DONE|KMEM
      1 C xfs_bnobt_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
      1 C xfs_cntbt_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
      1 C xfs_dir3_data_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
      1 C xfs_dir3_leaf1_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
      1 C xfs_finobt_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
      1 C xfs_inobt_buf_ops+0x0/0x30 WRITE|ASYNC|DONE
      1 C xfs_rmapbt_buf_ops+0x0/0x40 WRITE|ASYNC|DONE
```

## icluster 통계와 inode buffer의 submit

**관찰 출력**

```text
stats[mounted] icluster 0 0 0 buf 37 21 19 19 0 18 0 0 18 
stats[A] icluster 0 0 0 buf 1265 37 1231 1229 0 34 0 0 26 
stats[B] icluster 0 0 0 buf 1265 37 1231 1229 0 34 0 0 26 
stats[C] icluster 0 7 201 buf 1267 37 1233 1229 0 34 0 0 26 
== xfs_buf_submit bufops and flags by phase
      2 A xfs_inode_buf_ops+0x0/0x30 READ
      8 C xfs_inode_buf_ops+0x0/0x30 WRITE|ASYNC|DONE
```

## AGF freeblks 변경 (write -d, CRC 재계산)

**관찰 출력**

```text
== case write -d freeblks 0
-- before
freeblks = 65515
longest = 65512
crc = 0x3f3fbba1 (correct)
-- write
Allowing write of corrupted data with good CRC
freeblks = 0
-- after
Metadata corruption detected at 0x556c47a247e0, xfs_agf block 0x1/0x200
xfs_db: cannot init perag data (117). Continuing anyway.
freeblks = 0
longest = 65512
crc = 0x6b9d134b (correct)
mount: /mnt/p14v: can't read superblock on /dev/loop0.
mount rc=32
-- dmesg since p14v-d-1792638662294774750
[ 1040.353541] XFS (loop0): Mounting V5 Filesystem 7a72688b-c8f9-4168-b8f6-7de9b4ddefdd
[ 1040.354133] XFS (loop0): Metadata corruption detected at xfs_agf_verify+0x208/0x280, xfs_agf block 0x1 
[ 1040.355543] XFS (loop0): Unmount and run xfs_repair
[ 1040.356269] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 1040.368899] XFS (loop0): metadata I/O error in "xfs_read_agf+0x98/0x160" at daddr 0x1 len 1 error 117
[ 1040.370301] loop0: lost filesystem error report for type 5 error -117
[ 1040.370348] XFS (loop0): Error -117 reserving per-AG metadata reserve pool.
[ 1040.371301] XFS (loop0): Corruption of in-memory data (0x8) detected at xfs_fs_reserve_ag_blocks+0x8e/0x100 (fs/xfs/xfs_fsops.c:559).  Shutting down filesystem.
[ 1040.373507] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[ 1040.374655] loop0: lost filesystem error report for type 5 error -108
[ 1040.374658] XFS (loop0): Ending clean mount
[ 1040.375651] XFS (loop0): Error -5 reserving per-AG metadata reserve pool.
```

## AGF freeblks 변경 (write -c, CRC 재계산 없음)

**관찰 출력**

```text
== case write -c freeblks 0
-- before
freeblks = 65515
longest = 65512
crc = 0x397fb8a (correct)
-- write
Allowing write of corrupted data and bad CRC
freeblks = 0
-- after
Metadata CRC error detected at 0x56113f4c09e0, xfs_agf block 0x1/0x200
xfs_db: cannot init perag data (74). Continuing anyway.
freeblks = 0
longest = 65512
crc = 0x397fb8a (correct)
mount: /mnt/p14v: can't read superblock on /dev/loop0.
mount rc=32
-- dmesg since p14v-c-1792638662350423086
[ 1040.409594] XFS (loop0): Mounting V5 Filesystem 7ac20416-cd68-412c-97af-1c22372d3d20
[ 1040.410198] XFS (loop0): Metadata CRC error detected at xfs_agf_read_verify+0x80/0x110, xfs_agf block 0x1 
[ 1040.411612] XFS (loop0): Unmount and run xfs_repair
[ 1040.412281] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 1040.421620] XFS (loop0): metadata I/O error in "xfs_read_agf+0x98/0x160" at daddr 0x1 len 1 error 74
[ 1040.422691] loop0: lost filesystem error report for type 5 error -117
[ 1040.422744] XFS (loop0): Error -117 reserving per-AG metadata reserve pool.
[ 1040.423552] XFS (loop0): Corruption of in-memory data (0x8) detected at xfs_fs_reserve_ag_blocks+0x8e/0x100 (fs/xfs/xfs_fsops.c:559).  Shutting down filesystem.
[ 1040.425379] XFS (loop0): Please unmount the filesystem and rectify the problem(s)
[ 1040.426328] loop0: lost filesystem error report for type 5 error -108
[ 1040.426330] XFS (loop0): Ending clean mount
[ 1040.427139] XFS (loop0): Error -5 reserving per-AG metadata reserve pool.
```

## AGI blocktrash 뒤 mount

**관찰 출력**

```text
7.1.0-rc5-g8fde5d1d47f6
loop=/dev/loop0
agcount=4
-- agi 0 before
crc = 0x7fd26ba7 (correct)
count = 128
freecount = 60
-- blocktrash
1: 
	byte offset 1024, length 512
	buffer block 2 (fsbno 0), 1 bb
	inode -1, dir inode -1, type agi
blocktrash: 0/0 unknown block 388 bits starting 42:7 randomized
-- agi 0 after
Metadata CRC error detected at 0x56411e7f1a30, xfs_agi block 0x2/0x200
xfs_db: cannot init perag data (74). Continuing anyway.
crc = 0x7fd26ba7 (correct)
count = 128
freecount = 60
mount: /mnt/p14b: mount(2) system call failed: Structure needs cleaning.
mount rc=32
-- dmesg since p14b-1792638671171600233
[ 1049.231003] XFS (loop0): Mounting V5 Filesystem 3b6d5b55-d0ab-4bcc-9d32-058eeb04d9f3
[ 1049.231519] XFS (loop0): Metadata CRC error detected at xfs_agi_read_verify+0x80/0x110, xfs_agi block 0x2 
[ 1049.232763] XFS (loop0): Unmount and run xfs_repair
[ 1049.233385] XFS (loop0): First 128 bytes of corrupted metadata buffer:
[ 1049.244368] XFS (loop0): metadata I/O error in "xfs_read_agi+0x9e/0x180" at daddr 0x2 len 1 error 74
[ 1049.245517] loop0: lost filesystem error report for type 5 error -117
[ 1049.245519] XFS (loop0): xfs_imap_lookup: xfs_ialloc_read_agi() returned error -117, agno 0
[ 1049.247394] XFS (loop0): Failed to read root inode 0x80, error 117
[ 1049.247401] XFS (loop0): Uncorrected metadata errors detected; please run xfs_repair.
```

