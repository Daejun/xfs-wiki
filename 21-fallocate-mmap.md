# fallocate와 mmap

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/21-fallocate-mmap.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/21-fallocate-mmap.md, 접힌 원본 출력은 raw/21-fallocate-mmap.md.

## 개요

먼저 읽어 둘 page는 [free space 구조와 할당 정책](09-free-space.md)(speculative preallocation의 크기)와 [buffered write](18-buffered-write.md)(delalloc, writeback, unwritten 변환)이다. 여기서는 data device 하나에 internal log가 있는 일반 file system의 일반 file을 다룬다.

- fallocate는 대부분의 mode에서 data를 쓰지 않고 extent map만 바꾼다. ALLOCATE와 ZERO_RANGE는 unwritten extent를 만들고, PUNCH_HOLE은 온전한 block만 hole로 만들며, COLLAPSE_RANGE와 INSERT_RANGE는 disk block을 그대로 두고 file offset만 옮긴다.
- buffered write가 EOF 뒤에 붙여 둔 speculative preallocation은 EOF 뒤를 지울지 판정이 처음 참이 되는 close 한 번, background blockgc, 공간 부족 때의 scan, `xfs_spaceman`이 같은 판정과 같은 함수로 정리한다. 이 함수는 fallocate한 file의 EOF 뒤 real block을 남기므로, 그 block은 truncate나 punch로만 없어진다.
- MAP_SHARED로 mmap한 hole에 처음 store하면 XFS는 block 번호를 정하지 않고 delalloc 공간만 예약한다. block 번호와 unwritten 변환은 writeback 때 정해진다.

예의 값은 관찰마다 option 없이 새로 만든 1 GiB file system(block 4 KiB, reflink와 rmapbt 켜짐)에서 얻었다.

## fallocate mode

fallocate는 data를 옮기거나 0을 쓰는 대신 extent record의 상태와 file offset을 바꾼다. 예외는 부분 block의 zeroing과 UNSHARE_RANGE의 복사다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

| mode (xfs_io 명령) | extent map | file 크기 | 이유와 조건 |
|---|---|---|---|
| ALLOCATE (`falloc`) | hole과 범위 안의 delalloc을 unwritten extent로 만든다. 이미 있는 written과 unwritten은 그대로 둔다. 범위는 block 경계까지 넓힌다 | 범위 끝이 EOF를 넘으면 할당 뒤 늘린다 | unwritten extent를 읽으면 buffered read가 folio를 0으로 채우므로 disk에 0을 쓰지 않아도 된다. 이미 있는 extent는 건드리지 않으므로 있던 data도 그대로다. transaction 하나에 extent 하나씩 잡고, inode에 `XFS_DIFLAG_PREALLOC`를 세운다 |
| ALLOCATE+KEEP_SIZE (`falloc -k`) | 같다. EOF 뒤에도 unwritten extent가 생긴다 | 그대로 | `XFS_IOC_RESVSP64` ioctl도 이 mode로 들어온다 |
| PUNCH_HOLE (`fpunch`) | 범위를 먼저 writeback하고 page cache에서 뺀 뒤, 온전히 덮인 block만 unmap해 hole로 만든다. 양 끝 부분 block은 extent를 두고 내용만 0으로 쓴다 | 그대로 | 바꿀 범위의 앞뒤에 걸친 extent까지 clean하고 idle하게 두려고 먼저 writeback한다(code 주석). delalloc은 이때 실제 block이 된다. 풀린 block은 deferred free로 돌아간다([deferred operation, intent, log recovery](13-intent-recovery.md)) |
| ZERO_RANGE (`fzero`) | punch한 뒤 범위를 block 경계로 넓혀 다시 ALLOCATE한다. 온전한 block은 written이던 것도 새 unwritten extent가 되고, 양 끝 부분 block은 0이 쓰인 채 원래 extent에 남는다(written이면 written으로) | KEEP_SIZE가 없으면 늘린다 | unwritten 변환 대신 punch를 쓰면 부분 block을 0으로 만들 수 있고, ENOSPC가 나도 범위를 읽으면 0이 나온다(code 주석) |
| COLLAPSE_RANGE (`fcollapse`) | 범위를 punch하고 뒤 extent의 file offset을 len만큼 당긴다. disk block은 그대로다. 앞 extent와 file offset과 disk block이 모두 이어지고, 상태가 같고, 합친 길이가 extent 길이 한계 안이면 합친다 | len만큼 줄인다 | data를 복사하지 않고 record의 file offset만 바꾼다. offset과 len이 block 경계에 맞아야 하고, 범위가 EOF에 닿으면 사실상 truncate라 EINVAL을 돌려준다 |
| INSERT_RANGE (`finsert`) | offset에서 extent를 쪼개고, 뒤 extent를 마지막 것부터 len만큼 뒤로 민다. 비는 자리는 hole로 남기고 extent를 합치지 않는다 | EOF folio를 writeback하고 먼저 len만큼 늘린다 | 크기를 먼저 늘려 두면 crash가 나도 옮긴 extent가 EOF 밖에 남지 않는다(code 주석). offset이 EOF 이상이면 EINVAL을 돌려준다 |
| UNSHARE_RANGE (`funshare`) | reflink file이면 공유된 block의 folio를 읽어 dirty로 만들고, writeback이 COW fork의 새 block에 쓴 뒤 data fork로 옮긴다. 남은 hole은 unwritten으로 채운다. reflink file이 아니면 hole만 채운다 | KEEP_SIZE가 없으면 늘린다 | 공유된 extent가 남지 않으면 inode의 reflink flag를 끈다. COW는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다 |

### 320 KiB file의 세 구간

file의 0-320 KiB를 0-64, 64-192, 192-320 KiB 세 구간으로 나누어 보면, 명령마다 구간이 이렇게 된다. block 수는 4 KiB 단위다.

| file과 명령 | 0-64 KiB | 64-192 KiB | 192-320 KiB | block 수 |
|---|---|---|---|---|
| written, `fpunch 64k 128k` | written | hole | written | 48 |
| written, `fzero 64k 128k` | written | unwritten(다른 자리) | written | 80 |
| unwritten(`falloc 0 320k`), `fpunch 64k 128k` | unwritten | hole | unwritten | 48 |
| unwritten, `fzero 64k 128k` | unwritten | unwritten(새 extent) | unwritten | 80 |
| 0-128 KiB만 unwritten, `fpunch 64k 128k` | unwritten | hole | hole | 16 |
| 64 KiB만 쓴 file, `falloc -k 64k 256k` | written | unwritten(EOF 뒤) | unwritten(EOF 뒤) | 80 |

- written file의 fzero 뒤 가운데 구간은 block 104-135에 새로 잡혔다. disk에서는 오른쪽 extent(72-103) 바로 뒤지만 file 순서로는 그 앞이라, 양 옆(24-39, 72-103)과 이어지지 않는다. event도 `xfs_zero_file_space` 다음에 punch(`xfs_free_file_space`, `xfs_bunmap`), ALLOCATE(`xfs_alloc_file_space`) 순으로 찍혔다.
- unwritten file에서도 가운데는 오른쪽 extent(264-295) 바로 뒤인 block 296-327에 잡혔다. 새 extent는 옆 extent와 file offset과 disk block이 모두 이어지고 상태가 같을 때만 합쳐지므로, 셋 다 unwritten인데도 extent가 셋으로 남는다.
- `falloc -k`를 한 file은 크기가 65536 byte인데 `bmap`에는 EOF 뒤 256 KiB가 unwritten extent로 나온다. `bmap`은 PREALLOC flag가 있는 file이면 EOF 뒤 extent도 보고한다.

### block 경계에 맞지 않는 punch

written 320 KiB file에 `fpunch 66k 124k`(66-190 KiB)를 하면, 시작은 68 KiB로 올리고 끝은 188 KiB로 내려 block 17-46의 30 block만 hole이 된다. block 16(64-68 KiB)과 block 47(188-192 KiB)은 written extent에 남고, 그 안의 66-68 KiB와 188-190 KiB만 page cache에서 0으로 쓴다. zeroing은 hole을 건너뛰고 EOF를 넘어서는 하지 않는다. zeroing 범위가 EOF에 닿고 EOF가 page 중간이면 EOF page를 writeback한다. 그 page가 mmap되어 있을 수 있기 때문이다(code 주석).

### INSERT와 COLLAPSE

1 MiB file에 `finsert 512k 64k`, hole에 64 KiB 쓰기, `fcollapse 512k 64k`를 차례로 하면 뒤 절반의 disk block은 끝까지 그대로이고 file offset만 움직인다.

**도식: INSERT와 COLLAPSE에서 움직이는 extent**

```text
4 KiB block 번호. A는 file의 앞 512 KiB, B는 뒤 512 KiB다.

file offset(KiB) 0           512   576         1088
                 |           |     |           |
(1)              [A 24-151  ][B 152-279  ]
(2) finsert      [A 24-151  ] hole [B 152-279  ]
(3) pwrite       [A 24-151  ][N   ][B 152-279  ]
(4) fcollapse    [A 24-151  ][B 152-279  ]

(1) 처음: extent 하나(block 24-279), 크기 1 MiB. B는 512-1024 KiB
(2) finsert 512k 64k: B는 disk block 그대로, file offset만 64 KiB 뒤로
(3) hole에 64 KiB를 쓰면 새 block N(280-295)이 잡힌다
(4) fcollapse 512k 64k: N을 풀고 B를 앞으로 당긴다.
    A 끝(151) 다음이 B 시작(152)이라 extent 하나(24-279)로 합친다
```

- finsert에서는 unmap한 block이 없었다(`xfs_bunmap` 없음). hole에 64 KiB를 쓴 뒤 크기는 1114112 byte(1 MiB + 64 KiB)였고, fcollapse 뒤에는 1048576 byte로 돌아왔다.
- 둘 다 옮기기 전에 EOF 뒤 preallocation을 정리해, 옮긴 뒤 file 안으로 드러나지 않게 한다. 옮길 범위는 writeback한 뒤 page cache와 COW fork를 비운다. rmapbt에서는 옛 mapping을 지우고 새 offset의 mapping을 넣는다.

### UNSHARE

1 MiB file u1을 reflink해 u2를 만들고 u2에 `funshare 0 1m`을 하면 이렇게 된다.

| 시점 | u1의 block | u2의 block | `fiemap`의 shared 표시 |
|---|---|---|---|
| reflink 직후 | 296-551 | 296-551 | 둘 다 있음 |
| funshare 뒤 | 296-551 | 552-807 | 둘 다 없음 |

u1은 block이 그대로인데도 shared 표시가 사라졌다. XFS는 FIEMAP에서 refcount btree로 공유 block을 찾았을 때만 shared로 표시한다. u2가 새 block(552-807)으로 옮기면서 u1과 함께 쓰던 block 296-551의 refcount가 줄어, 공유 block이 없다고 판정되었다(추정). COW 완료는 u2의 옛 mapping을 REMAP flag로 지워 block을 free하지 않고, 그 block의 refcount를 따로 줄인다(code로만 확인).

### lock과 log

XFS는 모든 mode에서 먼저 IOLOCK_EXCL과 MMAPLOCK_EXCL을 잡고 pNFS layout을 회수한다. 이어 진행 중인 direct I/O를 기다리고 mtime과 ctime을 바꾼 뒤 mode별 처리에 들어간다. 그래서 fallocate가 도는 동안 같은 file의 mmap fault는 기다린다. wsync로 mount했거나 file을 O_SYNC나 O_DSYNC로 열었거나 inode가 sync 속성이면 끝에 log force를 한다. 그 밖에는 extent 변화가 CIL에 들어가 다음 log force 때 disk에 쓰인다([log](12-log.md)).

code: xfs_file.c, xfs_bmap_util.c, libxfs/xfs_bmap.c, xfs_reflink.c, fs/iomap/buffered-io.c

## EOF 뒤 preallocation의 정리

XFS는 쓰지 않은 EOF 뒤 preallocation을 바로 돌려받지 않고, 지울지 판정이 처음 참이 되는 close 한 번, 일정 시간이 지난 뒤, 공간이 모자랄 때, 관리자가 요청할 때 돌려받는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

buffered write가 EOF를 넘겨 쓰고 그 뒤에 extent가 없으면, XFS는 쓰는 양보다 더 많이 EOF 뒤까지 예약할 수 있다(speculative preallocation). file이 최소 크기(예의 file system에서는 64 KiB)보다 작으면 붙이지 않는다. 이 예약은 처음에 delalloc이고, writeback이 delalloc extent 전체를 할당하면 unwritten real block이 된다. data fork에 이 예약을 잡을 때 inode에 `XFS_IEOFBLOCKS`를 세우고, AG에 blockgc(EOF 뒤 block과 COW block을 거두는 background 작업) tag를 단다. AG에 tag가 처음 붙으면 그 AG의 blockgc work를 sysctl `fs.xfs.speculative_prealloc_lifetime`초(기본 300초) 뒤로 예약한다.

### 네 file의 정리

sysctl lifetime을 5초로 낮추고 새 file system에 네 file을 만들면 이렇게 된다. block 수는 4 KiB 단위다.

| file | 한 일 | close 뒤 block 수 | 그 뒤 | 정리한 경로 |
|---|---|---|---|---|
| a | 한 번 열어 1 MiB를 한 번에 write | 256 | 그대로 | close(EOF 뒤는 비어 있었다) |
| d | `falloc -k 0 1m` 뒤 64 KiB write와 fsync | 256 | 그대로 | 없음 |
| b | 32 KiB씩 아홉 번 열고 쓰고 닫은 뒤 128 MiB write | 65520 | spaceman 뒤 32768 | 첫 close(EOF 뒤는 비어 있었다), spaceman |
| e | b와 같은 순서, 마지막은 8 MiB | 4080 | spaceman 뒤 그대로, blockgc worker 뒤 2048 | 첫 close(EOF 뒤는 비어 있었다), blockgc worker |

- a: 빈 file에 1 MiB를 한 번에 쓰면 EOF 뒤 preallocation이 생기지 않는다. write할 때 file 크기가 0이라 최소 크기(64 KiB)보다 작기 때문이다. 그래도 close는 지우는 함수를 한 번 거쳤다. close 때 EOF 안쪽에 delalloc이 남아 판정이 참이 되었다(추정).
- b와 e: 첫 close 때 file은 32 KiB라 EOF 뒤 preallocation이 없었지만, EOF 안쪽 delalloc 때문에 판정이 참이 되어(추정) 지우는 함수가 돌고 flag가 섰다. 그래서 그 뒤 여덟 번의 close와 큰 write의 close는 지우지 않았고, b는 close한 뒤에도 EOF 뒤에 약 128 MiB가 남았다.
- spaceman: `xfs_spaceman -c "prealloc -m 64m -s"`로 scan하면 kernel이 크기가 64 MiB보다 작은 file을 건너뛰어, b만 잘리고 8 MiB인 e는 남았다.
- blockgc worker: e를 fsync해 clean하게 만든 뒤 worker가 돌아 e의 EOF 뒤를 잘랐다. lifetime이 5초인데 worker는 한 번만 돌았다. 돈 뒤 tag가 남은 inode가 없으면 다시 예약하지 않기 때문이다.
- d: close에서는 PREALLOC flag가 있고 delalloc이 없어 판정이 거짓이었다. 64 KiB를 쓴 자리가 이미 unwritten extent 안이라 예약이 없었고 blockgc tag도 붙지 않아, background worker와 spaceman의 scan도 d를 보지 않았다.

### 지울지 판정하는 조건

아래 조건을 위에서부터 보아, 앞의 다섯 가운데 하나에 걸리면 지우지 않고 마지막 조건에 맞으면 지운다.

1. 일반 file이 아니다.
2. 크기가 0이고 page cache가 비었고 delalloc도 없다.
3. inode의 extent map을 아직 disk에서 읽어 오지 않았다.
4. `XFS_DIFLAG_PREALLOC`가 있고(fallocate한 file) delalloc이 없다.
5. EOF가 이미 file 최대 크기(s_maxbytes)에 닿아 있다.
6. delalloc이 하나라도 있거나 EOF block 자리에 extent가 있다. EOF 안쪽의 delalloc만 있어도 참이 된다.

지우는 함수 `xfs_free_eofblocks`는 direct I/O를 기다린 뒤 `tr_itruncate` transaction에서 EOF 뒤 data fork를 자르고 tag를 지운다. discard는 하지 않고 on-disk size도 바꾸지 않는다. PREALLOC이나 APPEND flag가 있으면 EOF 뒤 delalloc만 지우고 real block은 남긴다.

### 정리하는 경로

| 경로 | 언제 지우나 | 건너뛰는 때 |
|---|---|---|
| close | 쓰기로 연 file을 닫을 때 판정이 처음 참이 되면 그 한 번만 지우고 `XFS_EOFBLOCKS_RELEASED`를 세운다. truncate로 크기를 줄이면 다음 close가 이 flag를 지우고 처음처럼 다시 판정한다 | 읽기 전용 open, read-only mount, shutdown, nlink 0, APPEND flag, zoned. IOLOCK을 nowait로 잡지 못할 때(mmap_lock과의 deadlock을 피한다) |
| background blockgc | AG마다 delayed work 하나가 lifetime 뒤에 돌고, 돈 뒤에도 tag가 남아 있으면 다시 lifetime 뒤로 예약된다. 판정이 거짓인 inode는 tag만 지운다 | dirty page가 있는 inode(동기 scan이 아닐 때), IOLOCK을 nowait로 잡지 못한 inode |
| 공간 부족 | buffered write가 ENOSPC를 받으면 dirty inode를 flush하고 동기 scan을 한 뒤 한 번 재시도한다(EDQUOT이면 그 quota의 inode만). transaction 예약이 ENOSPC를 받으면 모든 AG의 work를 지금으로 당겨 flush한다. 둘 다 끝에 inodegc를 flush한다([inode 수명](15-inode-lifecycle.md)) | 없음. 동기 scan은 dirty page가 있어도 본다 |
| `XFS_IOC_FREE_EOFBLOCKS` | `xfs_spaceman -c "prealloc [-s] [-m minlen]"`이 부른다. CAP_SYS_ADMIN이 필요하고, `-s`를 주면 동기 scan을 한다 | 크기가 minlen보다 작은 file |
| 그 밖 | inactivation(마지막 참조가 사라질 때. nlink가 0보다 크면 EOF 뒤만), collapse와 insert 전, read-only remount(cache된 inode 전체를 동기 scan, 관찰하지 않음), unmount(nlink가 0보다 큰 inode도 판정이 참이면 inactivation 경로로) |  |
| truncate | 크기를 줄이거나 같게 두면 새 크기 뒤를 직접 자르고 tag를 지운다. 판정과 지우는 함수를 거치지 않아 PREALLOC 예외가 없다 | 크기를 늘리는 truncate |

- log처럼 열고 쓰고 닫기를 되풀이하는 file은 preallocation을 남겨 두어야 조각이 줄고, tar로 푼 file처럼 한 번 쓰고 마는 file은 첫 close에서 바로 돌려받는 편이 낫다. 그래서 close는 한 번만 지운다(code 주석).
- sysctl을 바꿔도 이미 예약된 blockgc work는 당겨지지 않는다.

code: xfs_bmap_util.c, xfs_file.c, xfs_icache.c, xfs_iops.c, xfs_ioctl.c, xfs_inode.c

## mmap write fault

MAP_SHARED 영역에 store하는 시점에는 data가 page cache에만 있으므로, XFS는 write fault에서 block 번호를 정하지 않는다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

fault는 buffered write와 같은 `xfs_buffered_write_iomap_begin`을 불러 공간을 예약만 하고 folio를 dirty로 표시한다. 할당과 unwritten 변환은 writeback이 한다([buffered write](18-buffered-write.md)).

### 두 file의 extent map

fallocate로 잡은 12 MiB file의 처음, 6 MiB, 마지막 4 KiB 자리에 store하고 fsync하면 extent map이 다섯 줄로 쪼개진다.

| file block | disk block | 상태 |
|---|---|---|
| 0 | 24 | written |
| 1-1535 | 25-1559 | unwritten |
| 1536 | 1560 | written |
| 1537-3070 | 1561-3094 | unwritten |
| 3071 | 3095 | written |

write fault는 세 번이었고 할당(`xfs_iomap_alloc`)은 없었다. fault가 iomap에 물은 범위가 셋 다 4 KiB였으므로 store가 닿은 folio도 4 KiB였고, writeback 완료도 4 KiB씩 세 번 변환했다.

1 MiB sparse file의 첫 4 KiB에 store하면 write fault가 1 block의 delalloc을 예약한다. `msync` 전에는 file block 0이 delalloc이고 나머지는 hole이다. `msync -s` 뒤에는 file block 0이 disk block 13의 written extent가 된다. 그 사이 writeback이 delalloc을 unwritten으로 할당했고, I/O 완료가 이 4 KiB를 written으로 한 번 바꾸었다. 이때 `xfs_unwritten_convert`가 찍혔다.

### fault가 거치는 순서

1. 진입: page가 아직 map되지 않은 MAP_SHARED 영역에 쓰면, VM은 먼저 `->fault`로 folio를 읽어 채우고 이어 `->page_mkwrite`를 부른다. hole과 unwritten 자리는 읽으면 0이 나온다. DAX가 아니면 write fault여도 앞 event는 `xfs_read_fault`다.
2. lock: `sb_start_pagefault`로 freeze와 맞물리고([mount, unmount, freeze](16-mount-freeze.md)), `file_update_time`으로 mtime과 ctime을 바꾼 뒤 MMAPLOCK_SHARED를 잡는다. fallocate와 크기를 바꾸는 truncate는 MMAPLOCK_EXCL을 잡으므로 fault와 서로 막는다. reflink remap이 도는 중이면 EXCL로 기다린다. XFS의 MMAPLOCK은 VFS의 `invalidate_lock`을 가리킨다.
3. 길이: fault가 iomap에 묻는 범위는 folio 하나이고, EOF에서 잘린다. folio가 이미 EOF 밖에 있으면 fault는 VM_FAULT_NOPAGE로 끝난다. 그래서 buffered write와 달리 mmap write로는 EOF 뒤 speculative preallocation이 생기지 않는다(code로만 확인).
4. dirty 표시: folio를 dirty로 표시하고, folio가 stable해지기를 기다린 뒤 folio lock을 쥔 채 돌아간다.

fault 자리의 extent 상태에 따라 공간을 예약할지가 갈린다.

| fault 자리 | fault 때 | 그 뒤 |
|---|---|---|
| hole | delalloc을 예약한다. free counter에서 미리 빼므로 공간이 모자라면 fault에서 ENOSPC가 난다. 길이는 최대 1024 page다 | writeback이 unwritten으로 할당하고 I/O 완료가 written으로 바꾼다 |
| unwritten(fallocate한 범위) | 할당하지 않고 unwritten mapping을 돌려준다 | I/O 완료가 dirty folio가 덮는 범위만 written으로 바꿔 extent가 쪼개진다 |
| written | 할당하지 않는다 | dirty 표시만 한다 |
| reflink로 공유된 block | COW fork에 delalloc을 예약한다 | writeback이 새 block에 쓰고 COW 완료가 data fork에 넣는다 |

fault는 실패해도 예약한 delalloc을 되돌리지 않는다. buffered write는 쓰지 못한 범위의 새 delalloc을 `->iomap_end`에서 지우지만 fault는 건너뛴다. code 주석에 따르면 `iomap_page_mkwrite`는 delalloc을 철회해야 하는 식으로 실패하지 않는다. fault에서 난 ENOSPC와 EDQUOT은 VM_FAULT_SIGBUS로 바뀐다.

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_inode.c, fs/iomap/buffered-io.c, mm/memory.c

## 갈라지는 곳

| 조건 | 달라지는 점 |
|---|---|
| mode 조합 | VFS가 먼저 검사한다. PUNCH_HOLE은 KEEP_SIZE와 함께 써야 하고, COLLAPSE_RANGE, INSERT_RANGE, WRITE_ZEROES는 KEEP_SIZE와 함께 쓸 수 없다. XFS는 WRITE_ZEROES를 지원하지 않아 EOPNOTSUPP을 돌려주고, 일반 file이 아니면 EINVAL을 돌려준다 |
| zoned inode | PUNCH_HOLE, ZERO_RANGE, COLLAPSE_RANGE 때 경계 block 둘을 먼저 예약하고 같은 함수로 간다. fault도 folio 크기만큼 예약한다. preallocation이 없어 close 정리를 건너뛴다([zoned XFS](29-zoned.md)) |
| DEBUG kernel의 errortag `force_zero_range` | ZERO_RANGE가 punch 뒤 ALLOCATE 대신 실제로 0을 쓴다. 관찰 image는 DEBUG가 꺼져 있다 |
| always_cow(DEBUG) | ALLOCATE는 EOPNOTSUPP을 돌려준다. ZERO_RANGE와 UNSHARE_RANGE가 끝에 부르는 ALLOCATE 단계는 아무것도 하지 않는다 |
| extent size hint | ALLOCATE의 예약량을 hint 경계로 넓히고, `bmap`은 EOF 뒤도 보고한다. buffered write와 mmap fault도 delalloc 대신 바로 unwritten으로 할당한다 |
| realtime device, rt extent가 큰 경우 | punch는 온전한 rt extent만 지우고, 정렬 단위가 rextsize가 된다([realtime, rtgroup, metadir](28-realtime.md)) |
| DAX | fault가 page cache 없이 `xfs_dax_fault_locked`로 간다. huge fault는 DAX일 때만 쓴다 |
| 읽기 전용 open | close가 EOF 뒤를 정리하지 않는다. 이 문제는 fstests xfs/632의 주석에 있다 |

code: xfs_file.c, xfs_bmap_util.c, xfs_inode.c, fs/open.c

## 문서와 code가 다른 곳

| 문서 | code |
|---|---|
| kernel admin-guide의 xfs 문서에는 workqueue 표에 blockgc의 workqueue 이름이 `xfs-gc`로 적혀 있다 | code가 만드는 이름은 `xfs-blockgc/<dev>`다 |

## 한계점

다음은 확인하지 못했다.

- unmap의 unwritten과 realtime 처리(`__xfs_bunmapi`)는 줄 단위로 읽지 않았다. 해제의 흐름은 [deferred operation, intent, log recovery](13-intent-recovery.md)와 [free space 구조와 할당 정책](09-free-space.md)에 있다.
- ZERO_RANGE가 다시 잡은 block을 할당기가 원래 자리와 다른 곳에서 고른 이유.
- block 경계에 맞지 않는 punch 뒤 남은 부분 block의 내용이 0이 되었는지.
- funshare 뒤 refcount btree의 record. shared 표시가 사라진 것만 보았다.
- 1 MiB를 한 번 쓴 file a의 close 때 delalloc이 실제로 남아 있었는지.
- mmap write가 EOF 뒤 preallocation을 만들지 않는다는 것. 두 관찰 모두 `xfs_iomap_prealloc_size`가 찍히지 않았지만, fault 자리가 EOF 안쪽이어서 buffered write였어도 preallocation 조건이 거짓이었다.
- large folio가 쓰이는 환경에서 mmap write fault의 delalloc 길이와 written 범위가 어떻게 변하는지.
- VM_FAULT_SIGBUS가 process에 SIGBUS로 전달되는 arch 쪽 fault code.

## 실험 환경

기준 kernel과 xfsprogs, 관찰 image, 저장 장치는 [XFS 전반](00-start.md)의 실험 환경에 있다.

- 관찰마다 guest `/dev/shm`의 1 GiB sparse image를 loop device로 붙이고 option 없는 `mkfs.xfs`로 새로 만들어 option 없이 mount했다. 그래서 관찰 사이에는 inode와 block 번호가 이어지지 않는다. block은 4 KiB이고 reflink와 rmapbt가 켜져 있다.
- EOF 뒤 preallocation 관찰에서는 sysctl `fs.xfs.speculative_prealloc_lifetime`을 300에서 5로 낮췄다가 끝에 되돌렸다.
- 경우마다 `trace_marker`로 표시를 넣어 event를 구분했다.

### 관찰 목록

| 관찰 | 명령과 켠 event | 본문의 예 |
|---|---|---|
| fallocate mode | ```bash<br># event: xfs_alloc_file_space xfs_free_file_space xfs_zero_file_space xfs_bunmap<br>xfs_io -f -c "truncate 320k" -c "pwrite -S 0x61 0 320k" -c fsync -c "fzero 64k 128k" -c "bmap -vvp" c2z<br>xfs_io -f -c "truncate 320k" -c "pwrite -S 0x61 0 320k" -c fsync -c "fpunch 64k 128k" -c "bmap -vp" c2p<br>xfs_io -f -c "truncate 320k" -c "falloc 0 320k" -c "fzero 64k 128k" -c "bmap -vp" -c stat c3z<br>xfs_io -f -c "truncate 320k" -c "falloc 0 320k" -c "fpunch 64k 128k" -c "bmap -vp" c3p<br>xfs_io -f -c "truncate 320k" -c "falloc 0 128k" -c "fpunch 64k 128k" -c "bmap -vp" c8p<br>xfs_io -f -c "pwrite -S 0x61 0 64k" -c fsync -c "falloc -k 64k 256k" -c stat -c "bmap -vp" ck<br>xfs_io -f -c "truncate 320k" -c "pwrite -S 0x61 0 320k" -c fsync -c "fpunch 66k 124k" -c fsync -c "bmap -vp" cu<br>``` | 320 KiB file의 세 구간, block 경계에 맞지 않는 punch |
| INSERT, COLLAPSE, UNSHARE | ```bash<br># event: xfs_free_file_space xfs_collapse_file_space xfs_insert_file_space xfs_bunmap<br>#        xfs_itruncate_extents_start xfs_reflink_unshare xfs_reflink_end_cow<br>xfs_io -f -c "falloc 0 1m" -c "pwrite -S 0x68 -b 1m 0 1m" -c fsync -c "bmap -elpv" s1<br>xfs_io -c "finsert 512k 64k" -c "bmap -elpv" -c "pwrite -S 0x69 512k 64k" -c fsync -c "bmap -elpv" s1<br>xfs_io -c "fcollapse 512k 64k" -c "bmap -elpv" s1<br>xfs_io -f -c "pwrite -S 0x61 -b 1m 0 1m" -c fsync u1<br>xfs_io -f -c "reflink u1" -c "fiemap -v" -c "bmap -elpv" u2<br>xfs_io -c "fiemap -v" u1<br>xfs_io -c "funshare 0 1m" -c "fiemap -v" -c "bmap -elpv" u2<br>xfs_io -c "fiemap -v" u1<br>``` | INSERT와 COLLAPSE, UNSHARE |
| EOF 뒤 preallocation | ```bash<br># event: xfs_iomap_prealloc_size xfs_inode_set_eofblocks_tag xfs_inode_clear_eofblocks_tag<br>#        xfs_inode_free_eofblocks_invalid xfs_itruncate_extents_start xfs_blockgc_worker<br>#        xfs_ioc_free_eofblocks xfs_blockgc_free_space<br>echo 5 > /proc/sys/fs/xfs/speculative_prealloc_lifetime<br>xfs_io -f -c "pwrite -b 1m 0 1m" -c stat a<br>xfs_io -f -c "falloc -k 0 1m" -c "pwrite 0 64k" -c fsync d<br># b: fstests xfs/014의 open-write-close 반복 뒤 큰 write. e는 마지막이 8m<br>for i in $(seq 0 32768 262144); do xfs_io -f -c "pwrite $i 32k" b; done<br>xfs_io -c "pwrite -b 1m 0 128m" b<br>xfs_spaceman -c "prealloc -m 64m -s" <mount point><br>xfs_io -c fsync e; xfs_io -c fsync b; sleep 15<br>``` | 네 file의 정리 |
| mmap write fault | ```bash<br># event: xfs_alloc_file_space xfs_read_fault xfs_write_fault xfs_iomap_found xfs_iomap_alloc<br>#        xfs_iomap_prealloc_size xfs_unwritten_convert<br># fstests xfs/166이 쓰는 program. XFS_IOC_RESVSP64로 잡고 mmap해 처음, 가운데, 끝에 store하고 fsync<br>src/unwritten_mmap 12582912 t<br>xfs_io -c "bmap -vp" t<br>xfs_io -f -c "truncate 1m" -c "mmap -rw 0 1m" -c "mwrite -S 0x62 0 4096" -c "bmap -elpv" -c "msync -s 0 1m" -c "bmap -elpv" -c munmap h<br>``` | 두 file의 extent map |

### 출력 읽는 법

- `bmap -vp`의 FLAGS 열에서 `010000`은 unwritten이다. `fiemap -v`의 flags에서 `0x2000`은 shared, `0x1`은 마지막 extent다.
- `xfs_inode_clear_eofblocks_tag`의 iflags `0x40`은 `XFS_EOFBLOCKS_RELEASED`다. close가 이 flag를 세운 뒤 지우는 함수를 부르므로 a에서도 이 값이 찍혔다.
- read fault 뒤 `xfs_iomap_found`의 startblock이 `0xfffffffffffffffe`이면 hole이다. write fault 뒤의 `xfs_iomap_found`는 찾은 extent를 자르지 않고 찍으므로 12 MiB file에서는 전체 extent가 나온다.
- DAX huge fault가 아니면 `xfs_read_fault`와 `xfs_write_fault`의 order 값은 늘 0이라, 이 값으로는 folio 크기를 알 수 없다.
- fcollapse 끝의 `xfs_itruncate_extents_start`는 disize가 이미 새 크기라서, 크기를 줄이는 마지막 단계에서 찍힌 것으로 보았다. 마지막 단계는 on-disk 크기를 먼저 새 값으로 바꾼 뒤 truncate하고, 준비 단계의 EOF 정리는 on-disk 크기를 바꾸지 않는다(code로만 확인).
- b에서는 `xfs_iomap_prealloc_size`를 찍은 process를 pid 순서로 write와 짝지었다(추정). 128 MiB write에서는 128 block에서 32768 block까지 찍혔다.
- blockgc worker는 기다리기 시작한 표시의 약 5.4초 뒤에 한 번 찍혔다.

### 원본 출력

#### 320 KiB file의 세 구간

(원본 출력 "written, unwritten file에 fzero, fpunch, falloc -k, 부분 block fpunch를 한 뒤 extent map": `raw/21-fallocate-mmap.md`)

(원본 출력 "fzero, fpunch, falloc -k의 event 순서": `raw/21-fallocate-mmap.md`)

#### INSERT와 COLLAPSE, UNSHARE

(원본 출력 "finsert, fcollapse, funshare 전후 extent map": `raw/21-fallocate-mmap.md`)

(원본 출력 "finsert, fcollapse, funshare의 event 순서": `raw/21-fallocate-mmap.md`)

#### 네 file의 정리

(원본 출력 "close, spaceman, background blockgc worker의 정리 결과와 event 순서": `raw/21-fallocate-mmap.md`)

#### 두 file의 extent map

(원본 출력 "unwritten 범위와 sparse file의 mmap write fault": `raw/21-fallocate-mmap.md`)
