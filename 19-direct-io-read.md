# direct I/O와 read

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/19-direct-io-read.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/19-direct-io-read.md, 접힌 원본 출력은 raw/19-direct-io-read.md.

## 개요

- XFS에서 O_DIRECT write는 page cache와 delalloc을 거치지 않는다. hole을 만나면 write를 부른 task가 그 자리에서 unwritten extent로 할당해 bio를 보내고, 완료 때 written으로 바꾸면서 size를 올린다.
- direct I/O에서는 file offset과 길이가 device의 logical block size에 맞아야 하고, 이 정렬 값은 `statx`로 알 수 있다. reflink한 file에서는 fs block에 맞지 않는 write가 buffered write로 내려간다.
- direct read와 buffered read는 hole과 unwritten 구간을 device에서 읽지 않고 0으로 채운다(관찰한 것은 direct read의 0뿐). EOF를 넘는 direct read는 결과를 byte 단위 EOF에서 자른다.

예의 값은 option 없이 만든 1 GiB file system(512 byte sector loop device)에서 동기 I/O로 얻었다. [buffered write](18-buffered-write.md)와 [iomap](17-iomap.md)을 먼저 읽어 두면 따라가기 쉽다.

## 호출 순서

direct write와 direct read는 모두 `iomap_dio_rw`를 거친다. 동기 I/O면 호출한 task가 완료까지 처리한다. VFS에서 들어오는 분기와 완료를 모으는 방식은 [iomap](17-iomap.md)에 있다.

| 단계 | direct write | direct read |
|---|---|---|
| XFS 진입 | `xfs_file_dio_write`가 정렬을 검사한 뒤 aligned, unaligned, zoned, atomic 경로로 가른다 | `xfs_file_dio_read`가 atime을 갱신하고 IOLOCK_SHARED를 잡는다 |
| page cache | 같은 범위를 비운다. 비우지 못하면 `-ENOTBLK`로 buffered write로 내려간다(atomic write는 `-EAGAIN`) | 같은 범위의 dirty page cache를 먼저 writeback한다 |
| `->iomap_begin` | `xfs_direct_write_iomap_begin`이 ILOCK_SHARED로 조회하고, hole이면 ILOCK을 놓고 할당한다 | `xfs_read_iomap_begin`이 ILOCK_SHARED로 data fork만 조회한다. COW fork는 보지 않는다 |
| I/O | `iomap_dio_bio_iter`가 offset과 길이를 다시 검사하고 bio를 보낸다. block 일부를 쓰는 곳이 새 block이나 unwritten이면 zero bio도 보낸다 | written이면 bio를 보내고, hole과 unwritten이면 user buffer를 0으로 채운다 |
| 완료 | `xfs_dio_write_end_io`가 unwritten을 변환하거나 size를 올린다 | EOF 뒤로 읽은 만큼 결과에서 뺀다 |

code: xfs_file.c, xfs_iomap.c, fs/iomap/direct-io.c

## direct write: 할당과 완료

direct write는 hole을 만나면 bio를 만들기 전에 unwritten extent로 할당하고, I/O가 끝나면 written으로 바꾼다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 52 KiB 새 file

빈 file(inode 131)에 52 KiB를 direct write하면 extent와 on-disk size가 이렇게 바뀐다.

| 단계 | 실행한 task | extent(file block, disk block, 길이, 상태) | on-disk size |
|---|---|---|---|
| `->iomap_begin` | `xfs_io` | hole을 할당: 0, 40, 13 block, unwritten | 0 |
| data bio | `xfs_io` | 그대로. bio 하나가 block 40부터 52 KiB를 쓴다 | 0 |
| 완료 | `xfs_io` | 0, 40, 13 block, written | 0 → 53248 |
| 같은 범위 덮어쓰기 | `xfs_io` | 할당도 변환도 없다(mapping이 written) | 53248 그대로 |

할당과 변환에 transaction을 하나씩 썼고 `xfs_setfilesize`는 불리지 않았다. EOF 뒤로 더 잡은 block도 없었다.

### 할당

- 할당은 mapping이 hole이나 delalloc일 때만 한다. 이미 written이나 unwritten block이 있으면 할당 없이 그대로 쓴다(event `xfs_iomap_found`).
- `->iomap_begin`이 ILOCK을 놓은 뒤, `xfs_iomap_write_direct`가 transaction 하나로 할당한다. ILOCK_EXCL은 이 transaction이 다시 잡는다. commit은 [log](12-log.md)의 CIL에 넣을 뿐 log force를 하지 않는다.
- 한 번에 1024 page(4 KiB page면 4 MiB)까지 할당하고, 더 긴 write는 iomap이 `->iomap_begin`을 다시 불러 나머지를 할당한다(관찰하지 않음). EOF 안쪽 hole이면 hole 끝까지만 할당한다.
- EOF 너머로 할당할 때 끝을 올리는 기준은 extent size hint와 stripe unit(`swalloc`이면 stripe width)뿐이다. hint가 있으면 file 크기와 관계없이 hint 단위로 올리고, stripe 단위는 file이 그보다 작으면 쓰지 않는다. buffered write의 speculative preallocation은 이 경로에 없다.

### unwritten으로 잡는 이유

새 extent를 unwritten으로 잡아야 하는 이유가 두 가지 있다.

- crash: 할당 transaction은 data bio보다 먼저 commit된다. written으로 할당했는데 할당 기록은 log에 닿고 data bio는 device에 닿기 전에 crash하면, [log recovery](13-intent-recovery.md) 뒤 그 block의 예전 내용(다른 file이 쓰던 것)이 이 file의 내용으로 보인다. unwritten extent는 읽으면 0이 나온다. 이 이유는 DAX 분기와 delalloc 변환의 주석에 적힌 같은 이유에서 가져왔다(추정).
- 동시 reader: aligned direct write는 I/O 동안 IOLOCK을 shared로만 잡고, buffered read와 direct read도 shared로 잡는다. 그래서 할당이 끝난 뒤 data가 닿기 전에 같은 범위를 읽는 reader가 있을 수 있다. extent가 written이면 그 reader는 device에서 이전 주인의 내용을 읽고, unwritten이면 0을 받는다(code로만 확인).

새 extent는 `XFS_BMAPI_PREALLOC`으로 할당해 unwritten이 되고, iomap은 이 mapping을 보고 완료 때 변환하라고 표시해 둔다(`IOMAP_DIO_UNWRITTEN`).

### 완료

- I/O가 끝나면 `xfs_dio_write_end_io`가 `xfs_iomap_write_unwritten`으로 unwritten을 written으로 바꾸고, 같은 transaction에서 on-disk size와 in-core `i_size`를 올린다. 변환 범위는 block 단위로 올림한다. 변환 transaction은 [buffered write](18-buffered-write.md)의 변환과 같다.
- 변환할 것이 없는 write가 EOF를 넘으면 `xfs_setfilesize`가 size만 올린다. 이미 written인 block에 EOF를 넘겨 쓴 경우가 그렇다.
- in-core `i_size`는 변환이 끝난 뒤에 올린다. 먼저 올리면 동시에 도는 direct read가 아직 변환하지 않은 unwritten extent를 만나기 때문이다. XFS의 완료 주석에 이 순서의 이유가 적혀 있다.
- iomap은 완료 때 page cache를 한 번 더 비우는데, 이것도 `->end_io`(변환) 뒤에 한다. 주석에 따르면, 먼저 비우면 동시에 도는 buffered read가 unwritten의 0을 cache에 남긴다.
- 동기 I/O에서는 `iomap_dio_complete`를 부른 task가 완료를 직접 처리한다. 52 KiB 예에서도 `xfs_end_io_direct_write`와 `xfs_unwritten_convert`가 `xfs_io` task에서 찍혔다. AIO이고 완료에 work가 필요하면 workqueue의 kworker에서 돈다(code로만 확인).

code: xfs_iomap.c, xfs_file.c, xfs_bmap.c, fs/iomap/direct-io.c

## block 일부 쓰기와 lock

새 block이나 unwritten block에 block보다 작게 쓰면 나머지를 0으로 채우고, 같은 block을 두 write가 함께 채우지 않도록 IOLOCK을 달리 잡는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### zeroing

새로 할당했거나(`IOMAP_F_NEW`) unwritten인 block에 block보다 작게 쓰면, iomap이 data bio 앞뒤로 block의 나머지를 0으로 채우는 bio를 따로 보낸다. 변환이 끝나면 그 block 전체가 written이 되므로, 0을 써 두지 않으면 쓰지 않은 부분에서 예전 내용이 보인다. EOF에 닿거나 EOF를 넘는 write는 written block이어도 뒤쪽을 0으로 채운다. 그러지 않으면 mmap으로 EOF block을 읽을 때 예전 내용이 보인다(code로만 확인).

**도식: block 일부 쓰기의 zero bio와 extent**

```text
(a) 52 KiB file의 EOF(53248)에 512 byte를 이어 쓴다. 새로 할당한
    file block 13 하나를 block 안의 byte 위치로 보면 이렇다

    0      512                                    4096
    +------+--------------------------------------+
    | data |         zero bio (3584 byte)         |
    +------+--------------------------------------+

    block 시작에서 쓰므로 앞쪽 zero bio는 없다
    쓰기 뒤: file block 0~13이 written extent 하나, size 53760

(b) 64 KiB를 unwritten으로 잡아 둔 file의 offset 4096에 512 byte를
    쓴다. file block 1에 data와 zero bio를 쓰고, 할당은 없다

    file block  0          1                 2                   15
    쓰기 전     [unwritten 16 block                               ]
    쓴 범위                [data|zero 3584 B]
    쓰기 뒤     [unwritten][     written    ][unwritten 14 block  ]
```

(a)에서는 새 block(disk block 53)이 앞 extent 바로 뒤에 잡혀, 변환 뒤 extent 하나(block 40부터 14 block)로 합쳐졌다.

### IOLOCK

block 정렬 write는 IOLOCK_SHARED로 들어간다. EOF 너머에서 시작하거나 `S_NOSEC`이 아직 서지 않았으면 IOLOCK_EXCL을 잡았다가 I/O 전에 SHARED로 내린다. pNFS layout을 깨야 하거나 reflink remap 중일 때도 그렇다. XFS는 attr fork가 없는 inode에만 `S_NOSEC`을 처음부터 세운다. attr가 있는 inode에서는 첫 write의 privilege 정리 끝에 VFS가 세운다.

block 정렬이 아닌 write에서는 `xfs_file_dio_write_unaligned`가 다음 규칙으로 IOLOCK을 잡는다.

- EOF에 닿거나 EOF를 넘는 write는 처음부터 IOLOCK_EXCL로 들어가, 다른 direct I/O가 끝나기를 기다린다(`inode_dio_wait`).
- 그 밖에는 IOLOCK_SHARED로 덮어쓰기만 시도한다(`IOMAP_DIO_OVERWRITE_ONLY`). 할당이 필요하거나 written이 아닌 extent에 block보다 작게 쓰면 `-EAGAIN`으로 물러났다가 IOLOCK_EXCL로 다시 한다.
- reflink file이면 `-ENOTBLK`를 내서 buffered write로 내려간다.

위 예의 write 다섯은 IOLOCK을 이렇게 잡았다.

| write | 처음 IOLOCK | 그다음 |
|---|---|---|
| 새 file의 offset 0에 52 KiB | SHARED | attr가 있는 file의 첫 write라 EXCL로 올렸다가 I/O 전에 SHARED로 내림 |
| 같은 범위 52 KiB 덮어쓰기 | SHARED | 그대로 |
| EOF(53248)에 512 byte | EXCL | 그대로 |
| unwritten file의 offset 4096에 512 byte | SHARED | `-EAGAIN` 뒤 EXCL로 다시. user에게는 512 byte를 썼다고 돌아감 |
| offset 100에 512 byte | 잡지 않음 | sector 정렬이 아니어서 그 전에 `EINVAL` |

첫 행의 file은 새로 만든 빈 file이지만 attr가 있었다(`fsxattr.xflags`의 X). 기본 mkfs가 parent pointer를 켜므로(`parent=1`) 새 file에도 parent pointer가 attr로 붙는다(추정). parent pointer는 [xattr와 parent pointer](08-xattr-parent.md)에 있다.

code: xfs_file.c, xfs_iomap.c, xfs_iops.c, fs/inode.c, fs/iomap/direct-io.c

## 정렬 조건

file offset과 길이는 device의 logical block size에, user buffer는 queue의 `dma_alignment`에 맞춘다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 정해지는 곳

| 조건 | 기준값 | 검사하는 곳 | 어기면 |
|---|---|---|---|
| write의 offset과 길이 | device의 logical block size(mount 때 정한다) | `xfs_file_dio_write` | `-EINVAL`. tracepoint가 찍히기 전에 돌아간다 |
| write의 offset과 길이 | fs block size | `xfs_file_dio_write` | 오류가 아니다. block 일부 쓰기 경로로 간다 |
| read와 write의 offset과 길이 | device의 logical block size. always-COW inode(zoned inode 등)의 write면 fs block size | `iomap_dio_bio_iter` | bio를 내는 구간이 맞지 않으면 `-EINVAL`. hole과 unwritten만 지나는 read는 검사하지 않는다(code로만 확인) |
| user buffer의 주소와 segment 길이 | queue의 `dma_alignment`(기본 511) | block layer가 bio를 나눌지 볼 때 | 검사를 거친 bio는 오류로 끝난다(code로만 확인) |

read에는 XFS 자체의 정렬 검사가 없고, iomap이 logical block 단위로만 본다. 그래서 reflink한 file도 logical block 단위(512 byte sector device면 512 byte)로 읽을 수 있다. buffer 주소는 block layer가 bio를 queue limit에 맞춰 나눌지 볼 때만 검사한다. 검사를 거치는 bio는 segment가 둘 이상이거나 page 끝을 넘는 bio다. queue에 `chunk_sectors`가 있으면 모든 bio가 거치고, 없으면 한 page 안의 segment 하나짜리 bio는 거치지 않는다(code로만 확인).

### 512 byte sector와 4096 byte sector

sector 크기만 다른 두 loop device에서는 결과가 이렇게 갈렸다. 4096 byte sector device에서는 mkfs가 `sectsz=4096`으로 만들었다.

| 항목 | 512 byte sector | 4096 byte sector |
|---|---|---|
| 512 byte direct write | 된다 | `EINVAL` |
| 512 byte direct read | 된다 | `EINVAL` |
| block 단위 direct write와 read | 된다 | 된다 |
| `dio_offset_align`(공유한 적 없는 file) | 512 | 4096 |
| `dio_read_offset_align` | 512 | 4096 |
| `dio_mem_align` | 512 | 512 |

loop device가 `dma_alignment`를 따로 정하지 않아 기본값 511이 남으므로, buffer 정렬은 512 그대로다.

### statx와 reflink

`statx`에 `STATX_DIOALIGN | STATX_DIO_READ_ALIGN`을 주면 `xfs_report_dioalign`이 값을 채운다. `dio_mem_align`은 `dma_alignment` + 1이고, `dio_read_offset_align`은 logical block size다. `dio_offset_align`은 COW inode(reflink flag가 있는 file, 또는 zoned inode 같은 always-COW inode)면 fs block 하나, 아니면 logical block size다. COW inode에서는 block보다 작은 write를 buffered로 내려야 하기 때문이다.

XFS 전용 `XFS_IOC_DIOINFO`도 같은 값에서 나오고, `xfs_io stat`에서는 `dioattr.*` 줄에 나온다. `d_miniosz`는 `dio_offset_align`, `d_maxiosz`는 `INT_MAX & ~(d_miniosz - 1)`, `d_mem`은 `dio_mem_align`을 pointer 크기로 올림한 값이다.

COW inode인지는 fs 기능이 아니라 inode flag(`XFS_DIFLAG2_REFLINK`)로 정한다. 그래서 기본 mkfs가 reflink를 켜도 공유한 적 없는 file에서는 512가 나온다. reflink는 사본뿐 아니라 원본에도 flag를 세운다.

| file | 상태 | `dio_offset_align`, `d_miniosz` | `d_maxiosz` |
|---|---|---|---|
| f1 | 공유한 적 없음 | 512 | 2147483136 |
| f2 | reflink 원본 | 4096 | 2147479552 |
| f3 | reflink 사본. extent가 공유 상태(flag 100000) | 4096 | 2147479552 |

`dio_mem_align`과 `dio_read_offset_align`은 셋 모두 512다. f3에 512 byte를 direct write하면 XFS가 이를 buffered write로 돌린다(`xfs_reflink_bounce_dio_write` 뒤 `xfs_file_buffered_write`). f1의 같은 write는 direct write로 끝나고, f3의 512 byte direct read도 된다.

code: xfs_file.c, xfs_buf.c, xfs_iops.c, xfs_ioctl.c, xfs_inode.h, fs/iomap/direct-io.c, block/blk-merge.c, block/blk.h

## hole과 unwritten 읽기

direct read는 할당도 transaction도 없이 data fork를 조회하고, mapping type에 따라 bio를 보내거나 user buffer를 0으로 채운다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 200 KiB direct read

앞 52 KiB만 written인 file 셋을 200 KiB direct read로 읽으면 다음과 같다.

| file | size | iomap이 받은 mapping | 돌려준 byte |
|---|---|---|---|
| h1 | 100 KiB | 0~52 KiB written(bio), 52~200 KiB hole(0으로 채움) | 102400 |
| h2 | 102401 byte | h1과 같다 | 102401 |
| u1 | 100 KiB | 0~52 KiB written, 52~100 KiB unwritten(0으로 채움), 100~200 KiB hole | 102400 |
| h1, offset 128 KiB에서 4 KiB | 100 KiB | 없음. EOF 너머에서 시작한다 | 0 |

hole mapping은 EOF에서 끊기지 않고 요청 끝까지 왔고, 돌려준 길이만 EOF에서 잘렸다. 읽은 내용은 hole과 unwritten 구간이 0, written 구간이 쓴 값(0xcd)이었다(구간마다 앞 32 byte만 확인).

- hole과 unwritten 구간에서는 `iomap_dio_hole_iter`가 device를 읽지 않고 user buffer를 0으로 채운다(device를 읽지 않는 것은 code로만 확인).
- delalloc을 만나면 "Direct I/O collision with buffered writes!" 경고를 남기고 `-EIO`를 돌려준다. read 전에 범위를 writeback하므로 mmap write와 겹칠 때만 난다(code로만 확인).
- EOF나 그 너머에서 시작하는 read는 바로 0을 돌려준다. EOF를 넘는 read는 끝까지 처리한 뒤, EOF 뒤로 진행한 만큼 되돌리고(`iov_iter_revert`) 반환값을 `i_size - offset`으로 자른다. 그래서 요청은 block 단위여도 결과는 byte 단위 EOF에서 끝난다. EOF가 든 block이 written이면 그 block은 device에서 읽는다.

### buffered read

buffered read도 같은 `xfs_read_iomap_ops`로 mapping을 받는다. page cache에 없는 folio는 readahead나 read_folio로 채운다. 이때 mapping이 written이 아니거나 새 block이거나 EOF 뒤면 그 범위를 0으로 채우고 uptodate로 표시한다(`iomap_block_needs_zeroing`). 나머지만 bio로 읽는다. u1을 buffered로 읽으면 readahead가 25 page(EOF인 100 KiB까지)로 들어가 written과 unwritten mapping 둘만 받았다.

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, fs/iomap/direct-io.c, fs/iomap/buffered-io.c

## 달라지는 곳

위 흐름은 zoned, DAX, realtime이 아닌 일반 file에서 extent size hint와 stripe unit이 없을 때에 해당한다.

| 경우 | 다른 점 |
|---|---|
| reflink(COW) file | direct write는 ILOCK_EXCL로 `->iomap_begin`에 들어가 `xfs_reflink_allocate_cow`를 부르고, 완료 때 `xfs_reflink_end_cow`를 부른다. block보다 작은 write는 buffered write로 내려간다. [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다 |
| atomic write | `xfs_file_dio_write_atomic`으로 간다. 할당이 2 block 이상 필요하면 `-ENOPROTOOPT`를 내서 COW 경로를 다시 탄다. [atomic write, large block size, DAX](27-atomic-lbs-dax.md)에 있다 |
| zoned | `xfs_file_dio_write_zoned`로 가고 완료는 `xfs_zoned_dio_write_end_io`가 처리한다. unwritten과 COW flag가 없어야 한다. [zoned XFS](29-zoned.md)에 있다 |
| realtime device | 정렬 단위(`xfs_inode_alloc_unitsize`)가 달라진다. [realtime, rtgroup, metadir](28-realtime.md)에 있다 |
| DAX | unwritten extent도 할당 쪽으로 간다. unwritten을 쓰지 않는 대신 commit 전에 zero를 쓴다 |
| stable write를 요구하는 device(data integrity 기능) | read와 write가 모두 `IOMAP_DIO_BOUNCE`를 쓴다 |

code: xfs_file.c, xfs_iomap.c

## 한계점

- direct write 경로가 unwritten을 쓰는 crash 쪽 이유는 그 경로 자체의 kernel 주석이나 문서에서 찾지 못했다.
- user buffer 주소가 `dma_alignment`에 맞지 않을 때 실패하는지와 user에게 가는 errno는 확인하지 못했다. `xfs_io`로는 이 경우를 만들 수 없었다.
- 동시 direct read가 변환 전 unwritten extent를 읽는 경쟁은 관찰하지 않았다. code 주석에 적힌 것까지만 확인했다.
- AIO 완료가 kworker로 가는 것, 한 번에 4 MiB로 자르는 것, delalloc에 걸린 direct read의 `-EIO`는 관찰하지 않았다.
- read 내용은 구간마다 앞 32 byte만 보았다. 나머지도 0과 0xcd라고 code와 `xfs_io`의 기본 pattern에서 추정했다. buffered read의 내용은 읽지 않았다.
- readahead가 범위를 EOF에서 자르는 code는 따라가지 않았다.
- EOF 뒤 hole mapping이 요청 끝까지 오는 것은 trace로만 봤고, 그 길이를 정하는 code는 따라가지 않았다.
- realtime device는 정렬 단위 말고는 확인하지 못했고, atomic write 경로는 다루지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 option 없는 `mkfs.xfs`로 새로 만들고 option 없이 mount했다. loop device의 logical block size는 512, `dma_alignment`는 511이다. 4096 byte sector 관찰만 loop device를 `losetup -b 4096`으로 만들었다.
- I/O는 모두 동기 I/O다. `xfs_io -d`로 O_DIRECT를 열고, buffered read는 `-d` 없이 했다. `xfs_io`는 buffer를 page 정렬로 잡는다.
- 경우마다 `trace_marker`로 표시를 남겼다. 4096 byte sector 관찰은 trace를 쓰지 않았고, read 관찰은 block event를 켜지 않았다.

### 관찰 목록

| 관찰 | 명령과 켠 event | 출처 | 본문의 예 |
|---|---|---|---|
| direct write 다섯 경우 | ```bash<br># event: xfs_file_direct_write xfs_ilock xfs_ilock_demote xfs_iomap_found<br>#        xfs_iomap_alloc xfs_end_io_direct_write xfs_unwritten_convert<br>#        xfs_setfilesize iomap_iter_dstmap iomap_dio_complete block_bio_queue<br>touch /mnt/dio19a/f1<br>xfs_io -f -c "resvsp 0 64k" -c "truncate 64k" /mnt/dio19a/f2<br>xfs_io -d -c "pwrite -b 52k 0 52k" /mnt/dio19a/f1<br>xfs_io -c "bmap -vp" -c "stat" /mnt/dio19a/f1<br>xfs_io -d -c "pwrite -b 52k 0 52k" /mnt/dio19a/f1<br>xfs_io -d -c "pwrite -b 512 53248 512" /mnt/dio19a/f1<br>xfs_io -c "bmap -vp" -c "stat" /mnt/dio19a/f1<br>xfs_io -d -c "pwrite -b 512 100 512" /mnt/dio19a/f1<br>xfs_io -d -c "pwrite -b 512 4096 512" /mnt/dio19a/f2<br>xfs_io -c "bmap -vp" -c "stat" /mnt/dio19a/f2<br>umount /mnt/dio19a<br>xfs_db -r -c "inode 131" -c "p core.size core.nblocks core.nextents u3.bmx" -c "inode 132" -c "p core.size core.nblocks core.nextents u3.bmx" <loop device><br>``` | fstests tests/xfs/164(`-f -d`와 `pwrite -b 52k`, resvsp와 truncate), common/tracing. man xfs_io(8)의 `-d`와 resvsp, man xfs_bmap(8)의 `-v -p` | 52 KiB 새 file, zeroing, IOLOCK |
| read 세 file | ```bash<br>xfs_io -f -d -c "pwrite -b 52k 0 52k" -c "truncate 100k" /mnt/dio19b/h1<br>xfs_io -f -d -c "pwrite -b 52k 0 52k" -c "truncate 102401" /mnt/dio19b/h2<br>xfs_io -f -d -c "resvsp 0 100k" -c "truncate 100k" -c "pwrite -b 52k 0 52k" /mnt/dio19b/u1<br># event: xfs_file_direct_read xfs_file_buffered_read xfs_iomap_found<br>#        iomap_iter_dstmap iomap_dio_complete iomap_readahead iomap_readpage<br>xfs_io -d -c "pread -b 200k 0 200k" /mnt/dio19b/h1<br>xfs_io -d -c "pread -b 200k 0 200k" /mnt/dio19b/h2<br>xfs_io -d -c "pread -b 4k 128k 4k" /mnt/dio19b/h1<br>xfs_io -d -c "pread -b 200k 0 200k" /mnt/dio19b/u1<br>xfs_io -c "pread -b 200k 0 200k" /mnt/dio19b/u1<br>xfs_io -d -c "pread -v 52k 512" /mnt/dio19b/h1 \| head -2<br>xfs_io -d -c "pread -v 64k 512" /mnt/dio19b/u1 \| head -2<br>xfs_io -d -c "pread -v 0 512" /mnt/dio19b/u1 \| head -2<br>umount /mnt/dio19b<br>xfs_db -r -c "inode 133" -c "p core.size core.nblocks core.nextents u3.bmx" <loop device><br>``` | fstests tests/xfs/164와 그 .out(EOF에서 잘린 read 길이), man xfs_io(8)의 pread와 resvsp | 200 KiB direct read, buffered read |
| statx와 reflink | ```bash<br>cat /sys/block/<loop>/queue/logical_block_size /sys/block/<loop>/queue/dma_alignment<br>xfs_io -f -c "pwrite -S 0x61 -b 64k 0 64k" -c "fsync" /mnt/dio19c/f1<br>xfs_io -f -c "pwrite -S 0x62 -b 64k 0 64k" -c "fsync" /mnt/dio19c/f2<br>xfs_io -f -c "reflink /mnt/dio19c/f2" /mnt/dio19c/f3<br>xfs_io -c "statx -r -m +dioalign,+dio_read_align" -c "stat" /mnt/dio19c/f1   # f2, f3도<br>xfs_io -c "bmap -vp" /mnt/dio19c/f3<br># event: xfs_file_direct_write xfs_reflink_bounce_dio_write<br>#        xfs_file_buffered_write xfs_file_direct_read<br>xfs_io -d -c "pwrite -S 0x63 -b 512 512 512" /mnt/dio19c/f1<br>xfs_io -d -c "pwrite -S 0x63 -b 512 512 512" /mnt/dio19c/f3<br>xfs_io -d -c "pread -b 512 512 512" /mnt/dio19c/f3<br>``` | fstests tests/generic/768(`statx -r -m`), tests/xfs/128(reflink 사본). man xfs_io(8)의 statx와 reflink | statx와 reflink, 512 byte sector와 4096 byte sector |
| 4096 byte sector device | ```bash<br>losetup -b 4096 -f --show /dev/shm/dio19d.img<br>blockdev --getss <loop device><br>mkfs.xfs -f <loop device><br>touch /mnt/dio19d/f1<br>xfs_io -c "statx -r -m +dioalign,+dio_read_align" -c "stat" /mnt/dio19d/f1<br>xfs_io -d -c "pwrite -b 512 0 512" /mnt/dio19d/f1<br>xfs_io -d -c "pwrite -b 4k 0 4k" -c "pread -b 512 0 512" -c "pread -b 4k 0 4k" /mnt/dio19d/f1<br>``` | man losetup(8)의 `-b`, fstests tests/generic/768 | 512 byte sector와 4096 byte sector |

### 출력 읽는 법

event 정의는 xfs_trace.h와 fs/iomap/trace.h에 있다.

| event | 읽는 법 |
|---|---|
| `xfs_file_direct_write` | lock과 write 검사를 마친 뒤 `iomap_dio_rw` 직전에 찍히고, `-EAGAIN` 재시도면 두 번 찍힌다. `disize`는 on-disk size |
| `xfs_iomap_alloc` | caller 칸이 없고 buffered write의 delalloc 예약에서도 나므로 direct write만 실행한 관찰에서 읽는다 |
| `xfs_iomap_found` | 할당 없이 기존 mapping을 쓴 경우. `startblock`이 큰 수(0xfffffffffffffffe)이면 hole이다 |
| `iomap_iter_dstmap` | mapping의 `type`(HOLE, MAPPED, UNWRITTEN)과 flag(NEW 등). extent의 unwritten 여부는 `xfs_iomap_alloc`과 `xfs_iomap_found`가 찍지 않으므로 여기서 본다 |
| `iomap_dio_complete` | `ret`은 `iomap_dio_rw`가 XFS에 돌려주는 값이다(-11은 `-EAGAIN`). XFS가 다시 시도하면 user에게 가지 않는다 |
| `block_bio_queue` | 512 byte 단위의 시작 sector와 sector 수. data bio와 zero bio가 따로 찍힌다 |
| `xfs_ilock` | lock 종류와 caller. 표시 사이에 close가 `xfs_can_free_eofblocks`에서 잡는 ILOCK_SHARED도 찍힌다 |

### 원본 출력: 52 KiB 새 file

(원본 출력 "새 file에 52 KiB direct write: tracepoint": `raw/19-direct-io-read.md`)

(원본 출력 "새 file에 52 KiB direct write 뒤의 extent와 stat": `raw/19-direct-io-read.md`)

(원본 출력 "같은 범위 덮어쓰기: tracepoint": `raw/19-direct-io-read.md`)

### 원본 출력: zeroing과 IOLOCK

(원본 출력 "EOF에 닿는 512 byte direct write: tracepoint": `raw/19-direct-io-read.md`)

(원본 출력 "EOF에 닿는 512 byte direct write 뒤의 extent와 on-disk 값": `raw/19-direct-io-read.md`)

(원본 출력 "unwritten extent에 512 byte direct write: tracepoint": `raw/19-direct-io-read.md`)

(원본 출력 "unwritten extent에 512 byte direct write 뒤의 extent와 on-disk 값": `raw/19-direct-io-read.md`)

(원본 출력 "sector 정렬이 아닌 direct write": `raw/19-direct-io-read.md`)

### 원본 출력: 512 byte sector와 4096 byte sector

(원본 출력 "4096 byte logical sector device의 direct I/O": `raw/19-direct-io-read.md`)

### 원본 출력: statx와 reflink

(원본 출력 "statx와 DIOINFO가 알리는 값": `raw/19-direct-io-read.md`)

(원본 출력 "reflink한 file의 direct write와 read": `raw/19-direct-io-read.md`)

### 원본 출력: 200 KiB direct read

(원본 출력 "hole과 EOF를 지나는 direct read: 결과": `raw/19-direct-io-read.md`)

(원본 출력 "hole과 EOF를 지나는 direct read: tracepoint": `raw/19-direct-io-read.md`)

(원본 출력 "unwritten 구간을 지나는 read: extent와 결과": `raw/19-direct-io-read.md`)

(원본 출력 "unwritten 구간을 지나는 direct read와 buffered read: tracepoint": `raw/19-direct-io-read.md`)

(원본 출력 "hole, unwritten, written 구간의 내용과 on-disk extent": `raw/19-direct-io-read.md`)
