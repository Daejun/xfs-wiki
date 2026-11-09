# iomap

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/17-iomap.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/17-iomap.md, 접힌 원본 출력은 raw/17-iomap.md.

## 개요

- iomap을 쓰는 filesystem은 "이 file 범위가 disk 어디에 어떤 상태로 있나"라는 물음에만 답한다. iomap은 그 답(mapping) 하나의 범위 안에서 page cache 복사나 bio 제출을 하고, 다 쓰면 다음 범위를 다시 묻는다.
- XFS는 경로마다 다른 ops를 준다. buffered write는 공간을 delalloc으로 예약만 하고, direct write는 그 자리에서 unwritten extent로 할당하며, read와 seek는 할당하지 않는다.
- writeback과 I/O 완료는 이 물음의 반복(`iomap_iter` loop)이 아니라 따로 둔 hook으로 처리한다. ioend는 이어지고 종류가 같은 구간을 bio 하나로 묶은 단위이고, 주로 writeback이 만든다. 완료에 transaction이 필요하거나 irq context에서 할 수 없는 일(unwritten 변환, on-disk 크기 갱신, COW 등)이 남은 ioend는 XFS가 workqueue로 넘긴다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. [inode](06-inode.md)의 extent와 [code를 읽기 위한 최소 모형](03-code-model.md)을 먼저 읽어 두면 따라가기 쉽다.

## 물음과 답

filesystem이 받는 질문은 하나뿐이다. "file offset `pos`부터 `len`까지 이 작업을 하려 한다. 앞에서부터 한 덩어리로 줄 수 있는 mapping은 무엇인가." 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

답은 `struct iomap`에 담아 돌려준다. 여기에는 disk 주소, file offset, 길이, type, flag와, 나중에 이 mapping이 아직 맞는지 볼 때 쓰는 `validity_cookie`가 들어 있다.

### type과 flag

mapping의 type에는 다섯 가지가 있다.

| type | 뜻 | 이 page의 예 |
|---|---|---|
| `IOMAP_HOLE` | 할당 없음 | data 없이 늘린 file의 read, FIEMAP의 끝 |
| `IOMAP_DELALLOC` | 예약만 됨, disk 주소 없음 | buffered write |
| `IOMAP_MAPPED` | 실제 block | 덮어쓰기, read |
| `IOMAP_UNWRITTEN` | 할당됐지만 아직 쓰지 않은 extent | 새 direct write, writeback이 받은 mapping |
| `IOMAP_INLINE` | inode 안에 있는 data |  |

| flag(일부) | 뜻 |
|---|---|
| `IOMAP_F_NEW` | 새로 할당하거나 예약한 block |
| `IOMAP_F_DIRTY` | datasync 때 함께 써야 할 inode 변경이 있음 |
| `IOMAP_F_SHARED` | COW 대상 |
| `IOMAP_F_STALE` | mapping이 낡았으니 다시 물어야 함 |

묻는 쪽은 `flags`로 작업 종류를 알린다. 같은 offset이라도 `IOMAP_WRITE`면 없는 block을 만들어 줘야 하고, `IOMAP_REPORT`(FIEMAP, seek)면 있는 그대로만 답하면 된다.

### iomap_ops

- `->iomap_begin`은 `pos`를 덮는 mapping 하나를 돌려준다. write면 없는 공간을 이때 예약한다. mapping의 길이는 요청과 다를 수 있고, iomap은 남은 길이보다 긴 mapping을 막지 않는다. 아래 256 KiB 예에서는 남은 64 KiB를 쓰는데 128 KiB짜리 mapping이 왔다.
- `->iomap_end`는 실제로 쓴 만큼(`written`)을 commit하고 나머지 예약을 푼다. 없어도 된다.
- COW에서는 읽어 올 원본의 mapping을 `srcmap`으로 따로 돌려준다. XFS는 COW fork의 mapping을 `iomap`에, data fork의 mapping을 `srcmap`에 넣는다. COW는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

code: include/linux/iomap.h, fs/iomap/iter.c

## iomap_iter loop

iomap의 진입점은 모두 같은 `iomap_iter` loop를 돈다. 한 바퀴에서 앞 mapping의 `->iomap_end`, 다음 mapping의 `->iomap_begin`, 그 mapping 안에서 일하는 body(진입점마다 다른 함수. buffered write는 `iomap_write_iter`, direct I/O는 `iomap_dio_iter`)가 차례로 불린다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 한 바퀴

같은 조건의 새 file system에서 첫 file(inode 131)에 1 MiB를 buffered write 한 번으로 쓰면 `iomap_iter`가 두 번 불린다.

| 호출 | pos | 한 일 |
|---|---|---|
| 첫 `iomap_iter` | 0 | 받은 mapping이 없어 바로 `->iomap_begin`을 부르고, DELALLOC 1 MiB(NEW, DIRTY, disk 주소 없음)를 받는다 |
| body | 0 → 1 MiB | folio 단위로 page cache에 복사하고 `pos`를 1 MiB까지 나아가게 한다 |
| 둘째 `iomap_iter` | 1 MiB | `->iomap_end`에 1 MiB를 썼다고 알리고(code로만 확인), 남은 길이가 0이라 loop를 끝낸다 |

1. 첫 호출이면 바로 4번으로 간다.
2. 둘째 호출부터는 이번 mapping에서 나아간 만큼(`pos - iter_start_pos`)을 `->iomap_end`에 `written`으로 넘긴다. `iomap_end`가 실패해도 진척이 있었으면 계속한다.
3. body가 오류를 냈으면 그 오류로 끝난다. 남은 길이가 0이거나, 진척이 없는데 `IOMAP_F_STALE`도 아니면 loop를 끝낸다.
4. `->iomap_begin`으로 다음 mapping을 받는다. mapping은 `pos`를 덮어야 한다(아니면 WARN).
5. body가 mapping 하나의 범위 안에서 일하고 `iomap_iter_advance`로 `pos`를 늘린다(남은 길이를 넘기면 `-EIO`). 1번으로 돌아간다.

write 하나가 begin을 몇 번 부를지는 mapping 수로 정해진다. direct로 offset 0과 128 KiB에 64 KiB씩 써 둔 file(inode 132)에 buffered로 256 KiB를 덮어쓰면 mapped와 delalloc mapping이 섞여 다섯 개 오고, begin도 다섯 번 불린다.

**도식: 256 KiB buffered write 한 번의 iomap_iter loop**

```text
xfs_io: inode 132에 256 KiB buffered write, pos 0부터

iomap_iter 1  ->iomap_begin(pos 0)        MAPPED    12 KiB
  body        12 KiB 복사, advance
iomap_iter 2  ->iomap_end(12 KiB 씀)
              ->iomap_begin(pos 12 KiB)   MAPPED    52 KiB
  body        52 KiB 복사, advance
iomap_iter 3  end, begin(pos 64 KiB)      DELALLOC  64 KiB  새 예약
  body        ...
iomap_iter 4  end, begin(pos 128 KiB)     MAPPED    64 KiB
  body        ...
iomap_iter 5  end, begin(pos 192 KiB)     DELALLOC 128 KiB  새 예약
  body        남은 64 KiB 복사, advance
iomap_iter 6  ->iomap_end(64 KiB 씀), 남은 길이 0 -> loop 끝
```

mapping 경계는 extent 경계와 delalloc 경계를 따라 끊긴다. pos 64 KiB의 빈 곳은 요청이 192 KiB 남았어도 다음 extent 앞에서 64 KiB로 잘렸다. pos 192 KiB의 예약은 남은 요청 64 KiB보다 큰 128 KiB였다. 이 write는 마지막 extent 뒤에서 file 크기(192 KiB)를 넘으므로 speculative preallocation이 붙는 조건에 든다.

### 진입점

진입점마다 `iomap_iter`에 넣는 flags가 다르다. trace의 `iomap_iter` event에는 ops 이름과 부른 함수가 찍히므로, 어느 진입점이 어느 XFS ops를 쓰는지 trace에서 바로 알 수 있다.

| 진입점 | 작업 | iter.flags | 관찰 |
|---|---|---|---|
| `iomap_file_buffered_write` | buffered write | `IOMAP_WRITE`. nowait, dontcache 요청이면 그 flag를 더한다 | 봄 |
| `__iomap_dio_rw` | direct I/O | read는 `IOMAP_DIRECT`, write는 `IOMAP_WRITE\|IOMAP_DIRECT`. nowait, overwrite-only, atomic 요청이면 그 flag를 더한다 | 봄 |
| `iomap_read_folio`, `iomap_readahead` | buffered read | 없음 | readahead만 봄 |
| `iomap_page_mkwrite` | mmap write fault | `IOMAP_WRITE\|IOMAP_FAULT` | 봄 |
| `iomap_zero_range`, `iomap_file_unshare` | zero range, unshare |  | 보지 않음 |
| `iomap_fiemap`, `iomap_bmap` | FIEMAP, bmap | `IOMAP_REPORT` | fiemap만 봄 |
| `iomap_seek_hole`, `iomap_seek_data` | seek | `IOMAP_REPORT` | 봄 |
| `iomap_swapfile_activate` | swapfile |  | 보지 않음 |

관찰한 읽기, FIEMAP, `SEEK_HOLE`에서는 다음 호출의 `pos`가 앞 mapping의 끝으로 나아갔다(FIEMAP의 마지막 HOLE은 남은 요청보다 길어 요청의 끝까지). `SEEK_DATA`는 달랐다. 앞의 256 KiB file은 fsync 뒤 12 KiB와 244 KiB의 extent 둘로 되어 있다. 여기에 `SEEK_DATA`를 하면 body는 첫 mapping(pos 0의 MAPPED 12 KiB)에서 data 위치 0을 기록하고 advance 없이 돌아오고, 둘째 호출은 진척이 없다고 보고 loop를 끝낸다.

code: fs/iomap/buffered-io.c, fs/iomap/direct-io.c, fs/iomap/fiemap.c, fs/iomap/seek.c

### buffered write의 body

`iomap_write_iter`는 mapping 하나 안에서 folio 단위로 돈다. 한 바퀴에서 다루는 양은 folio 경계와 mapping의 남은 길이로 자른다.

- 한 바퀴에서는 먼저 dirty page 양을 조절하고(`balance_dirty_pages_ratelimited_flags`) user page를 미리 fault-in한다. 그다음 `iomap_write_begin`으로 folio를 잡아 복사한 뒤 `iomap_write_end`로 마무리한다. 이어 in-core `i_size`를 올리고 `iomap_iter_advance`를 부른다. 복사가 짧게 끝나면 크기를 반으로 줄여 다시 한다.
- on-disk 크기는 여기서 올리지 않는다.
- `iomap_write_begin`은 folio를 잠근 뒤 `->iomap_valid`로 mapping이 아직 맞는지 묻는다. 거짓이면 `IOMAP_F_STALE`을 세우고, body는 멈추며, `iomap_iter`는 진척이 없어도 `->iomap_begin`을 다시 부른다. code 주석에 따르면 mapping을 받은 뒤 folio를 잡기까지 그 범위의 unwritten 상태가 바뀌거나, reclaim 뒤 같은 index에 다른 folio가 생길 수 있다.

### direct I/O의 body

`iomap_dio_iter`는 받은 mapping의 type으로 길을 고른다.

| type | write | read |
|---|---|---|
| `IOMAP_HOLE` | `-EIO` | 0으로 채운다 |
| `IOMAP_UNWRITTEN` | bio | hole처럼 0으로 채운다 |
| `IOMAP_MAPPED` | bio | bio |
| `IOMAP_DELALLOC` | "Direct I/O collision with buffered writes!" 경고와 `-EIO` | 같음 |

bio를 만들 때 mapping이 unwritten이면 `IOMAP_DIO_UNWRITTEN`을, `IOMAP_F_SHARED`면 `IOMAP_DIO_COW`를 `dio->flags`에 모은다. I/O가 끝나면 filesystem의 `end_io`가 이 flag를 보고 할 일(unwritten 변환, COW remap)을 정한다. loop 전체는 block plug 안에서 돈다.

### lock

lock 체계는 iomap이 아니라 filesystem이 정한다. iomap은 복사할 folio의 lock처럼 자기가 다루는 자료의 lock을 직접 잡는다. XFS의 buffered write는 `xfs_file_buffered_write`에서 IOLOCK(`i_rwsem`)을 exclusive로 잡은 뒤 iomap을 부르고, `->iomap_begin` 안에서 ILOCK을 잡는다. kernel 문서에 따르면 ILOCK처럼 mapping을 읽을 때 잡는 lock은 iomap이 folio lock을 잡기 전에 놓아야 한다.

- ILOCK은 `xfs_ilock_for_iomap`이 잡는다. read는 shared로, buffered write의 begin은 exclusive로 시작한다. extent list를 아직 메모리에 읽지 않았으면 exclusive로 잡는데, `IOMAP_NOWAIT`이면 그 대신 `-EAGAIN`을 돌려준다. NOWAIT에서 trylock이 실패해도 `-EAGAIN`을 돌려준다.

code: xfs_file.c, xfs_iomap.c, fs/iomap/buffered-io.c, fs/iomap/direct-io.c

## XFS의 ops

XFS에는 mapping을 답하는 `iomap_ops`가 여덟 벌 있고, 경로마다 다른 벌을 준다. 이와 별도로 direct I/O(`iomap_dio_ops`), writeback(`iomap_writeback_ops`), read(`iomap_read_ops`), mapping 재검사(`iomap_write_ops`)에 쓰는 ops가 있다. XFS의 read ops는 block device에 integrity checksum이 있을 때만 쓰고, 그 밖에는 iomap의 기본 read ops를 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### iomap_ops 여덟 벌

| ops | 쓰는 작업 | block이 없으면 begin은 | 돌려주는 type |
|---|---|---|---|
| `xfs_buffered_write_iomap_ops` | buffered write, mmap write fault, zero range, truncate, reflink unshare | delalloc으로 예약한다. end는 쓰지 못한 예약을 돌려준다 | DELALLOC(NEW) |
| `xfs_direct_write_iomap_ops` | direct write | transaction 안에서 unwritten extent로 할당한다. end는 없다 | UNWRITTEN(NEW) |
| `xfs_read_iomap_ops` | buffered read, direct read, FIEMAP의 data fork, bmap, swapfile, DAX read, DAX file의 reflink와 dedupe 준비 | 할당하지 않는다 | HOLE |
| `xfs_seek_iomap_ops` | `SEEK_HOLE`, `SEEK_DATA` | 할당하지 않는다 |  |
| `xfs_xattr_iomap_ops` | `FIEMAP_FLAG_XATTR` |  |  |
| 나머지 셋: `xfs_zoned_direct_write_iomap_ops`, `xfs_atomic_write_cow_iomap_ops`, `xfs_dax_write_iomap_ops` | zoned inode의 direct write, hardware 한도를 넘거나 `REQ_ATOMIC`이 안 되는 atomic write, DAX write |  |  |

같은 진입점이라도 받는 벌이 다르다. 예를 들어 `iomap_dio_rw`는 read면 `xfs_read_iomap_ops`를, write면 direct, zoned, atomic 셋 중 하나를 받는다. zoned, atomic, DAX ops는 [zoned XFS](29-zoned.md)와 [atomic write, large block size, DAX](27-atomic-lbs-dax.md)에 있다.

### ops가 정해지는 곳

ops는 VFS에서 XFS로 들어오는 함수에서 정해진다.

| 들어오는 곳 | 조건 | 가는 곳 |
|---|---|---|
| `xfs_file_write_iter` | DAX | `xfs_file_dax_write` |
|  | `IOCB_DIRECT` | `xfs_file_dio_write`. 정렬, zoned, atomic에 따라 다시 나뉜다([direct I/O와 read](19-direct-io-read.md)) |
|  | zoned inode | `xfs_file_buffered_write_zoned`. ops 벌은 같고, `xfs_buffered_write_iomap_begin` 첫머리에서 zoned용 begin으로 갈라진다 |
|  | 나머지 | `xfs_file_buffered_write` |
| `xfs_file_read_iter` | DAX, direct | DAX read, `xfs_file_dio_read` |
|  | buffered | `generic_file_read_iter`로 page cache에 간다. 없는 folio는 `read_folio`와 `readahead`로 와서 `xfs_read_iomap_ops`를 쓴다 |

direct write가 `-ENOTBLK`를 내면 buffered write로 다시 쓴다. XFS는 reflink file의 unaligned write에서 이 값을 내고, iomap은 같은 범위의 page cache를 비우지 못할 때 낸다.

### begin이 하는 일

begin이 extent record를 mapping으로 옮길 때는 공통 함수 `xfs_bmbt_to_iomap`을 쓴다. hole과 delalloc이면 type을 HOLE, DELALLOC으로 하고 disk 주소를 비워 둔다(`IOMAP_NULL_ADDR`). 실제 block이면 byte 주소로 바꾸고, unwritten extent면 UNWRITTEN, 아니면 MAPPED로 둔다. datasync 때 함께 써야 할 inode 변경이 있으면 `IOMAP_F_DIRTY`를 붙이고, `validity_cookie`에 fork의 sequence를 넣는다.

- read의 begin은 write나 zero 요청을 받지 않는다(ASSERT). ILOCK 아래에서 `xfs_bmapi_read`로 extent 하나를 읽고, FIEMAP이나 DAX면 공유 경계에서 자른다.
- direct write의 begin은 기존 extent를 읽고, COW가 필요하면 COW fork에 할당한다. 할당할 것이 없으면 읽은 mapping을 그대로 돌려주고, 할당이 필요한데 `IOMAP_NOWAIT`나 `IOMAP_OVERWRITE_ONLY`면 `-EAGAIN`을 돌려준다. 그 밖에는 `xfs_iomap_write_direct`가 [transaction](11-transaction.md) 안에서 기본으로 `XFS_BMAPI_PREALLOC`을 주어 unwritten extent로 할당한다.
- buffered write의 begin은 ILOCK exclusive 아래에서, reflink file이 아니면 메모리의 extent list만 본다. reflink file이면 refcount btree로 공유 여부를 확인해 mapping을 공유 경계에서 자른다. block이 없으면 delalloc으로 예약하는데, 그 뒤에 extent가 더 없으면서 write가 file 크기를 넘으면 speculative preallocation을 더한다. extent size hint가 있는 inode는 direct write의 begin으로 넘긴다.
- buffered write의 end는 이번에 새로 예약한 delalloc만 보고, fault는 건너뛴다. 쓰지 못한 뒷부분의 예약은 `xfs_buffered_write_delalloc_punch`로 되돌린다.

buffered write의 begin 상세는 [buffered write](18-buffered-write.md)에 있다.

direct write는 새로 쓸 때와 덮어쓸 때 begin이 하는 일이 다르다. 같은 file system의 새 file(inode 133)에 direct write로 1 MiB를 쓴 뒤 같은 범위를 다시 쓰면, begin의 event와 받는 mapping이 다음처럼 다르다.

| write | begin의 event | 받은 mapping | 완료 |
|---|---|---|---|
| 새로 쓰기 | `xfs_iomap_alloc`, block 341부터 256 block | UNWRITTEN(NEW, DIRTY) | `xfs_io` task가 written으로 바꾼다 |
| 덮어쓰기 | `xfs_iomap_found`, 같은 block 341 | MAPPED(DIRTY) | 변환 없음 |

code: xfs_iomap.c, xfs_file.c, xfs_iops.c, xfs_aops.c

### mapping 재검사

XFS는 buffered write ops를 쓰는 경로에서만 `->iomap_valid`로 mapping을 다시 확인한다. `xfs_iomap_write_ops`에는 `.iomap_valid = xfs_iomap_valid` 하나만 있고, buffered write, zero range, truncate, reflink unshare에서 buffered write ops와 함께 넘긴다. direct write에는 이런 재검사가 없다. `__iomap_dio_rw`에 write_ops를 받는 인자가 없기 때문이다.

- cookie에는 mapping을 만들 때의 fork 변경 sequence(`if_seq`)를 넣는다. xattr mapping이면 attr fork의 것을, 그 밖에는 data fork의 것을 쓴다. 공유(COW) mapping이면 COW fork의 `if_seq`를 상위 32 bit에 두어 bit OR로 합친다.
- `xfs_iomap_valid`는 hole mapping이면 늘 유효로 본다. cookie가 지금 값과 다르면 `xfs_iomap_invalid` event를 내고 거짓을 돌려준다.

buffered write가 mapping을 받은 뒤 folio를 잠그기 전에 그 범위의 data fork extent가 바뀌었다면, `if_seq`가 달라져 `xfs_iomap_valid`가 거짓을 돌려준다. 그러면 iomap이 `IOMAP_F_STALE`을 세우고 `->iomap_begin`을 다시 부른다(관찰하지 않음).

code: xfs_iomap.c, fs/iomap/buffered-io.c

## writeback과 완료

writeback은 `iomap_iter`를 쓰지 않는다. iomap이 dirty folio마다 `->writeback_range`로 mapping을 묻고, 이어지는 구간을 ioend로 묶어 bio로 보낸다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 1 MiB file의 writeback

앞의 inode 131에 fsync를 하면, write 때 예약만 한 1 MiB가 block 24부터 한꺼번에 unwritten으로 할당된다. I/O가 끝나면 kworker가 그 범위를 written으로 바꾸고, 같은 transaction에서 on-disk 크기도 1 MiB로 올린다.

| 시점 | task | mapping | on-disk 크기 |
|---|---|---|---|
| write | `xfs_io` | DELALLOC 1 MiB, disk 주소 없음 | 0 |
| fsync의 writeback | `xfs_io` | block 24부터 UNWRITTEN 1 MiB. `iomap_add_to_ioend` 한 번에 1 MiB를 붙인다 | 0 |
| 완료 | kworker | unwritten → written | 0 → 1 MiB |
| `drop_caches` 뒤 read | `xfs_io` | block 24부터 MAPPED 1 MiB | 1 MiB |

- delalloc 변환이 offset이 든 delalloc extent를 처음부터 통째로 할당하기 때문에 1 MiB가 한꺼번에 할당됐다. delalloc 변환은 data fork에도 새 extent를 늘 unwritten으로 만든다. code 주석에 따르면 crash 때 disk의 옛 내용이 보이지 않게 하려는 것이다.
- `xfs_setfilesize` event는 나오지 않았다. 변환 transaction이 on-disk 크기를 이미 올렸고, 완료 처리는 append 여부를 변환 뒤에 다시 보기 때문이다(code로만 확인. 불렸더라도 크기가 이미 올라 있으면 event 전에 돌아간다).
- mmap write fault도 같은 길을 간다. data 없이 64 KiB로 늘린 file(inode 134)에 mmap으로 4 KiB를 쓰면 fault가 `WRITE|FAULT` flag와 buffered write ops로 DELALLOC mapping을 받는다. msync의 writeback과 완료도 fsync 때와 순서가 같았다. mmap 쪽은 [fallocate와 mmap](21-fallocate-mmap.md)에 있다.

도식의 흐름은 code로 읽었고, task와 값은 관찰한 것이다.

**도식: 1 MiB file의 writeback과 완료를 나눠 맡는 context**

```text
xfs_io (fsync)            bio 완료               xfs-conv workqueue
------------------------  ---------------------  ------------------
xfs_vm_writepages
 iomap_writepages
  iomap_writeback_folio
   ->writeback_range
    xfs_map_blocks
     delalloc 1 MiB를
     block 24부터
     unwritten으로 할당
    iomap_add_to_ioend
  ->writeback_submit
   bi_end_io = xfs_end_bio
   submit_bio ----------->  xfs_end_bio
                            i_ioend_list에 넣고
                            work를 건다 ---------->  xfs_end_io
                                                     정렬, 병합
                                                     unwritten 변환,
                                                     on-disk 크기 1 MiB
                                                     iomap_finish_ioends
```

### writeback_range

XFS의 `.writepages`인 `xfs_vm_writepages`는 zoned inode면 `xfs_zoned_writeback_ops`를, 아니면 `xfs_writeback_ops`를 담아 `iomap_writepages`를 부른다. iomap은 dirty folio마다 `folio_start_writeback`을 부른 뒤, folio 안의 dirty 구간마다, 그 구간이 다 찰 때까지 `->writeback_range`를 되풀이한다. XFS의 `xfs_writeback_range`는 `xfs_map_blocks`로 mapping을 구해 `iomap_add_to_ioend`로 붙이고, 실패하면 그 folio의 delalloc을 지운다. 호출 tree는 [buffered write](18-buffered-write.md)에 있다.

- kernel 문서에 따르면 writeback은 folio를 잠그지만 `i_rwsem`과 `invalidate_lock`은 잡지 않는다. I/O가 실패해도 dirty bit는 지우고, `-EIO`는 fsync에서 돌려준다.
- `->writeback_range`는 DELALLOC이나 INLINE mapping을 돌려주면 안 된다. 그래서 `xfs_map_blocks`는 delalloc을 만나면 그 자리에서 `xfs_bmapi_convert_delalloc`으로 실제 block을 할당하고, 결과를 writeback context(`wpc`)에 둔 mapping에 넣으며 sequence도 갱신한다. 할당 과정은 [buffered write](18-buffered-write.md)에 있다.

### mapping 재사용

`xfs_map_blocks`는 앞 mapping을 다시 쓸 수 있으면 extent를 찾지 않는다. 재사용 판정(`xfs_imap_valid`)에는 buffered write의 `validity_cookie`가 아니라 writeback context에 따로 저장한 sequence를 쓴다.

- offset이 앞 mapping 범위 밖이면 다시 찾는다. COW mapping이면 범위만 본다.
- 그 밖에는 저장해 둔 `data_seq`, `cow_seq`를 fork의 지금 `if_seq`와 비교한다. 다르면 `xfs_wb_data_iomap_invalid`나 `xfs_wb_cow_iomap_invalid` event를 내고 다시 찾는다.
- 이 판정은 lock 없이 한 번, ILOCK shared 아래에서 한 번 더 한다. 다시 찾을 때는 COW fork, data fork 순서로 본다. delalloc이 아니면 mapping을 채우고 끝난다(`xfs_map_blocks_found` event).
- kernel 문서에도 writeback의 재사용 판정은 filesystem이 직접 해야 하고, 거기에 `validity_cookie`를 쓸 수 있는지는 분명하지 않다("unclear")고 되어 있다.

### ioend

`iomap_add_to_ioend`는 mapping type으로 ioend flag를 정한다.

| mapping | ioend |
|---|---|
| UNWRITTEN | `IOMAP_IOEND_UNWRITTEN` |
| MAPPED | flag 없음 |
| HOLE | 건너뛴다 |
| 그 밖(DELALLOC 등) | WARN과 `-EIO` |

`IOMAP_F_SHARED`면 `IOMAP_IOEND_SHARED`를, dontcache write의 folio면 `IOMAP_IOEND_DONTCACHE`를 더한다. 앞 ioend에 이어 붙이려면 file offset과 disk sector가 이어져야 한다(disk 주소를 아직 정하지 않은 `IOMAP_F_ANON_WRITE` mapping이면 sector는 보지 않는다). 또 병합을 막는 flag가 같고, bio 최대 크기와 folio 수 한도(`IOEND_BATCH_SIZE`)를 넘지 않아야 한다. 앞 mapping과 합치지 말라고 표시한 BOUNDARY mapping은 앞에 붙이지 않는다. 붙일 수 없거나 bio에 folio가 들어가지 않으면 앞 ioend를 `->writeback_submit`으로 보내고 새로 만든다.

### submit과 완료

ioend는 완료에 transaction이 필요하거나 irq context에서 할 수 없는 일이 남을 때만 workqueue로 간다.

- `xfs_writeback_submit`은 공유(COW) ioend면 보내기 전에 그 범위의 COW fork에 있는 unwritten extent를 written으로 바꾼다. ioend의 쓴 끝이 on-disk 크기를 넘거나(append), ioend가 unwritten이나 shared이거나 dontcache이면 `bi_end_io`를 `xfs_end_bio`로 바꾼다. 조건 표는 [buffered write](18-buffered-write.md)에 있다.
- 공통 helper `iomap_ioend_writeback_submit`은 `bi_end_io`가 비어 있으면 기본 handler(`ioend_writeback_end_bio`)를 넣고 `submit_bio`한다. 기본 handler는 folio마다 writeback을 끝내며, kernel 문서에 따르면 irq context에서 돌 수도 있다.
- `xfs_end_bio`는 ioend를 inode의 `i_ioend_list`에 넣고, 목록이 비어 있었을 때만 `xfs-conv/<dev>` workqueue에 work를 건다. 이미 걸려 있으면 목록에 더하기만 한다.
- work 함수 `xfs_end_io`는 목록을 통째로 가져와 offset 순으로 정렬하고(`iomap_sort_ioends`) 이어지는 것을 합친다(`iomap_ioend_try_merge`). transaction 수를 줄이려는 것이다.
- 그다음 `xfs_end_ioend_write`가 성공한 ioend마다 zoned면 `xfs_zoned_end_io`, shared면 `xfs_reflink_end_cow`, unwritten이면 `xfs_iomap_write_unwritten`을 부르고, direct가 아니면서 아직 on-disk 크기를 넘으면(append) `xfs_setfilesize`로 크기를 올린다. 마지막에 `iomap_finish_ioends`가 folio의 writeback을 끝낸다. kernel 문서에 따르면 filesystem은 자기 일을 한 뒤 `iomap_finish_ioends`를 불러야 하고, XFS도 이 순서대로 한다. I/O가 실패했을 때의 처리는 [buffered write](18-buffered-write.md)에 있다.

### 크기를 완료 뒤에 올리는 이유

iomap은 on-disk 크기를 쓰는 일을 filesystem에 맡긴다(code 주석에 따르면 되도록 I/O가 끝난 뒤에 쓴다). XFS는 data가 disk에 닿은 뒤에 올린다. 크기가 data보다 먼저 disk에 나가면 stale data가 드러날 수 있다. unwritten ioend면 변환 transaction이, 그 밖의 ioend(이미 written인 block, reflink의 COW, zoned)가 on-disk 크기를 넘기면 `xfs_setfilesize`가 올린다.

`xfs_iomap_write_unwritten`은 범위를 extent mapping 하나마다 transaction 하나로 written으로 바꾸고, 요청 범위의 끝(쓴 byte 끝으로 자름)이 on-disk 크기를 넘으면(in-core `i_size` 안에서) 첫 transaction에서 on-disk 크기를 그 끝까지 올린다. extent마다 그 extent의 끝까지 올리는 것이 아니다(code로만 확인, [direct I/O](19-direct-io-read.md)의 완료 절). 넷째 인자는 in-core `i_size`를 함께 올릴지만 정한다. direct write의 완료는 참으로, writeback의 완료는 거짓으로 부른다. buffered write는 in-core `i_size`를 이미 올려 두었으므로 거짓이어도 on-disk 크기가 오른다.

### direct I/O의 완료

direct I/O는 ioend 대신 `struct iomap_dio`가 완료를 모은다.

| I/O | 완료를 처리하는 곳 |
|---|---|
| 동기 | I/O를 낸 task. `iomap_dio_rw`가 `iomap_dio_complete`를 부르고, 이것이 filesystem의 `end_io`를 부른다 |
| 비동기 | 완료 때 할 일(unwritten 변환, i_size 증가, 오류 처리 등)이 있으면 `s_dio_done_wq` workqueue |

XFS의 `end_io`인 `xfs_dio_write_end_io`는 `IOMAP_DIO_COW`면 COW remap(`xfs_reflink_end_cow`, atomic write면 `xfs_reflink_end_atomic_cow`)을 먼저 하고, 이어서 `IOMAP_DIO_UNWRITTEN`이면 unwritten 변환을, 아니면 `xfs_dio_endio_set_isize`로 크기만 올린다. 변환은 넷째 인자를 참으로 부른다. 앞의 1 MiB 새 direct write에서는 `xfs_io` task가 변환까지 했고, in-core 크기와 on-disk 크기가 함께 0에서 1 MiB가 됐다. zoned inode는 `xfs_zoned_dio_write_end_io`를 쓴다. zoned write와 stable write가 필요한 mapping의 direct read(bounce)는 bio를 ioend로 감싸 XFS의 완료 workqueue를 거친 뒤 같은 dio 완료로 간다(code로만 확인).

code: xfs_aops.c, xfs_iomap.c, xfs_file.c, fs/iomap/buffered-io.c, fs/iomap/ioend.c, fs/iomap/direct-io.c

## 한계점

- mapping 재검사(`IOMAP_F_STALE`)와 writeback의 mapping 재사용 판정이 깨지는 경쟁은 일으켜 보지 않았다. 관련 event(`xfs_iomap_invalid`, `xfs_wb_data_iomap_invalid`, `xfs_wb_cow_iomap_invalid`)는 code로만 확인했다.
- fork의 `if_seq`를 어디서 올리는지는 따라가지 않았다. cookie가 fork의 extent 변경에 따라 바뀐다는 것은 code로만 읽었다.
- 256 KiB 예의 마지막 delalloc 예약이 왜 128 KiB인지는 따라가지 않았다.
- 완료를 맡은 kworker(`kworker/0:2`, `kworker/2:1`)가 `xfs-conv` workqueue의 것인지는 이름만으로 알 수 없다.
- `xfs_bmbt_to_iomap`이 rtgroup 시작 block에 세운 `IOMAP_F_BOUNDARY`와, zoned direct write의 begin이 세운 `IOMAP_F_DIRTY`는 바로 뒤의 대입으로 덮인다(code로만 확인). 의도인지, 영향이 있는지는 확인하지 못했다.
- atomic write, DAX, zoned, xattr ops와 unaligned direct write, zero range, unshare, swapfile은 code로 읽은 범위에서만 적었고 관찰하지 않았다. zero range, unshare, swapfile 진입점의 flags, mmap write fault의 body, direct I/O가 INLINE mapping을 받을 때의 동작은 확인하지 못했다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 한 번의 실행에서 guest의 `/dev/shm`에 1 GiB file을 만들어 loop device로 붙였다. 그 위에 `mkfs.xfs -q -f`로 file system을 새로 만들고 option 없이 mount했다. file 넷(a, b, c, e)을 차례로 만들었고 inode 번호는 131, 132, 133, 134였다.
- 켠 event는 아래 명령의 주석에 있다. 출력에는 mount한 device에서 난 event만 남겼다.
- STALE 경쟁은 errortag `wb_delay_ms`, `write_delay_ms`로 만들 수 있으나 debug image가 필요해 관찰에서 뺐다.

### 관찰 목록

| 관찰 | 절차 | 본문의 예 |
|---|---|---|
| buf-write | file a에 1 MiB buffered write | 한 바퀴, 1 MiB file의 writeback |
| writeback | file a에 fsync | 1 MiB file의 writeback, 완료 도식 |
| multi | file b에 64 KiB direct write 둘(offset 0과 128 KiB)과 fsync, 그 뒤 256 KiB buffered write | 256 KiB loop 도식 |
| dio-new, dio-over | file c에 1 MiB direct write, 같은 범위를 다시 | direct write의 새로 쓰기와 덮어쓰기 |
| buf-read, dio-read | `drop_caches` 뒤 file a buffered read, file c direct read | 진입점의 읽기, 1 MiB file의 writeback 표 마지막 줄 |
| report | multi 뒤 fsync한 file b에 `fiemap -v`와 `seek -a 0` | 진입점의 FIEMAP과 seek |
| fault | file e를 64 KiB로 늘리고 mmap, 4 KiB mwrite, msync | mmap write fault |

```bash
# event: iomap_iter iomap_iter_dstmap iomap_iter_srcmap iomap_add_to_ioend
#        iomap_writeback_folio iomap_dio_rw_begin iomap_dio_complete
#        xfs_iomap_alloc xfs_iomap_found xfs_map_blocks_alloc xfs_map_blocks_found
#        xfs_setfilesize xfs_unwritten_convert xfs_end_io_direct_write
xfs_io -f -c "pwrite -q -b 1m 0 1m" /mnt/p17io/a                      # buf-write
xfs_io -c fsync /mnt/p17io/a                                          # writeback
xfs_io -f -d -c "pwrite -q 0 64k" -c "pwrite -q 128k 64k" -c fsync /mnt/p17io/b
xfs_io -c "pwrite -q -b 256k 0 256k" /mnt/p17io/b                     # multi
xfs_io -c fsync /mnt/p17io/b
xfs_io -f -d -c "pwrite -q -b 1m 0 1m" /mnt/p17io/c                   # dio-new
xfs_io -d -c "pwrite -q -b 1m 0 1m" /mnt/p17io/c                      # dio-over
sync; echo 3 > /proc/sys/vm/drop_caches
xfs_io -c "pread -q -b 1m 0 1m" /mnt/p17io/a                          # buf-read
xfs_io -d -c "pread -q -b 1m 0 1m" /mnt/p17io/c                       # dio-read
xfs_io -c "fiemap -v" -c "seek -a 0" /mnt/p17io/b                     # report
xfs_io -f -c "truncate 64k" /mnt/p17io/e
xfs_io -c "mmap -w 0 64k" -c "mwrite 0 4k" -c "msync -s 0 64k" /mnt/p17io/e   # fault
```

### 출력 읽는 법

| event | 읽는 법 |
|---|---|
| 모든 event | 첫 칸은 event를 낸 task다. `xfs_io`는 write나 fsync를 부른 process, `kworker/...`는 workqueue worker다 |
| `iomap_iter` | `length` 칸은 지금 mapping에서 남은 길이라 첫 호출과 mapping을 다 쓴 뒤에는 0이다. `->iomap_end`에는 tracepoint가 없어 다음 호출의 `pos`로만 진척을 본다 |
| `xfs_iomap_alloc` | buffered write에서는 예약 뒤에 찍혀 `startblock`이 delalloc 표식(`0xffffffffe0008` 등)이다. `bytecount`는 요청 길이, `fsbcount`는 예약 길이다. direct write에서는 할당 뒤에 찍혀 실제 block 번호(`0x155`)가 나온다 |
| `iomap_iter_dstmap` | `addr 0xffffffffffffffff`는 disk 주소 없음(`IOMAP_NULL_ADDR`)이다 |
| `xfs_map_blocks_alloc` | 변환 전에 조회한 delalloc record를 찍어 `startblock`이 예약 때 값 그대로다. `bytecount`는 늘 block 하나(`0x1000`)라 범위가 아니다. 변환된 주소와 범위는 뒤 `iomap_add_to_ioend`의 `addr 0x18000`과 `dirty len 0x100000`에 나온다 |
| `xfs_end_io_direct_write`, `xfs_unwritten_convert` | 함수 첫머리에서 찍혀 크기를 올리기 전 값이 나온다. 첫 direct write에서는 둘 다 `isize 0x0 disize 0x0`이었고, 같은 범위를 다시 쓸 때의 `xfs_end_io_direct_write`는 `isize 0x100000 disize 0x100000`이었다 |
| fiemap 표 | 범위는 512 byte sector 단위다. `[0..23]`은 12 KiB, `[24..511]`은 244 KiB로 trace의 mapping 길이 `0x3000`, `0x3d000`과 같다 |

### 원본 출력

(원본 출력 "buf-write: 한 바퀴, 1 MiB file의 writeback": `raw/17-iomap.md`)

(원본 출력 "writeback: 1 MiB file의 writeback, 완료 도식": `raw/17-iomap.md`)

(원본 출력 "multi: 256 KiB loop 도식": `raw/17-iomap.md`)

(원본 출력 "dio-new, dio-over: direct write의 새로 쓰기와 덮어쓰기": `raw/17-iomap.md`)

(원본 출력 "buf-read, dio-read: 진입점의 읽기, 1 MiB file의 writeback 표 마지막 줄": `raw/17-iomap.md`)

(원본 출력 "report: 진입점의 FIEMAP과 seek": `raw/17-iomap.md`)

(원본 출력 "fault: mmap write fault": `raw/17-iomap.md`)
