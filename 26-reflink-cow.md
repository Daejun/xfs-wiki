# reflink, COW, dedupe, exchange-range

> XFS 전반 wiki, 묶음 "기능". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/26-reflink-cow.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/26-reflink-cow.md, 접힌 원본 출력은 raw/26-reflink-cow.md.

## 개요

reflink를 하면 두 file이 같은 physical block을 가리키게 된다. data는 복사하지 않고 mapping만 바꾼다. 이를 위해 XFS는 data fork 구조는 그대로 두고 AG마다 refcountbt만 더했다.

- 공유된 block에 쓰면 XFS는 COW fork에 새 block을 잡아 그곳에 쓰고, I/O가 끝나면 그 block을 data fork로 옮긴다. 옛 block의 refcount는 이때 줄어든다.
- dedupe는 reflink와 같은 code로 block을 공유하고, 그 전에 VFS가 두 범위의 내용을 비교한다. dest를 쓰기로 열 필요가 없고 dest의 시간도 바꾸지 않는다.
- exchange-range는 두 file의 extent mapping을 step 단위로 맞바꾼다. 다른 thread는 lock으로 막고, crash에는 step마다 log에 남기는 intent로 대비한다. 그래서 요청 하나는 전부 바뀌었거나 하나도 바뀌지 않은 상태로만 보인다.

예의 값은 새로 만든 1 GiB file system에서 얻었다. reflink와 COW의 예는 AG를 하나로, dedupe와 exchange-range의 예는 option 없이 만들었다.

## 구조

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 공유 block의 기록

free space btree와 refcountbt만 보면, block이 어디에 기록되는지는 쓰임새로 정해진다. 비어 있는 block은 free space btree에, 두 file 이상이 함께 쓰는 block은 refcountbt에 refcount 2 이상으로 올라간다. file 하나만 쓰는 block은 free space btree에도 refcountbt에도 없다. COW용으로 잡아 둔 staging block만 예외로, refcount 1에 cowflag를 달고 refcountbt에 들어간다. record의 구조는 [rmap과 refcount 구조](10-rmap-refcount.md)에 있다.

reflink와 dedupe의 remap, COW write가 끝날 때의 remap, exchange-range는 mapping을 deferred operation으로 바꾼다. 두 remap은 refcount 변경과 bmap 변경을 함께 걸고, exchange-range는 bmap 변경만 건다. 이 intent가 log에 남는 방식은 [deferred operation, intent, log recovery](13-intent-recovery.md)에 있다.

### COW fork

COW fork는 memory에만 두는 extent list(inode의 `i_cowfp`)이고 log에도 남지 않는다. 그래서 crash 뒤에는 COW fork가 사라진다. disk에는 실제 block을 받은 COW extent만 refcountbt의 staging record와 rmap의 `OWN_COW` record로 흔적을 남긴다. COW fork는 btree format으로 바꾸지 않는다.

xfs_reflink.c 위쪽 주석에 따르면, write가 성공한 것을 확인하기 전에는 data fork의 mapping을 건드리지 않으려고 COW mapping을 별도 fork에 둔다. "the mappings must be stored in a separate CoW fork because we do not want to disturb the mapping in the data fork until we're sure that the write succeeded." 같은 주석에 따르면 COW는 네 단계를 거친다. 먼저 delalloc으로 예약하고 unwritten extent로 실제 block을 잡는다. 그다음 쓰는 범위만 written으로 바꾸고, 끝나면 data fork로 remap한다.

staging record는 startblock의 최상위 bit(`XFS_REFC_COWFLAG`)로 표시한다. 주석에 따르면 이렇게 하면 staging record가 모두 tree의 오른쪽 끝에 모여, mount 때 낡은 staging extent를 빨리 지울 수 있다.

640 KiB file1을 file2로 reflink하고 file2의 block 0에 4 KiB를 쓰면 두 fork는 아래처럼 바뀐다. COW fork는 사용자 공간에서 보기 어렵다. DEBUG kernel에서만 `bmap -c`로 볼 수 있다.

**도식: 공유 block에 쓸 때의 data fork와 COW fork**

```text
(1) reflink 직후
    file1 data fork  block 0~159 --+
                                   +--> 16408~16567  refcount 2
    file2 data fork  block 0~159 --+
    file2 COW fork   비어 있음

(2) file2 block 0에 4 KiB write, 그리고 writeback
    file2 COW fork   block 0~31  ---> 16568~16599  staging
                     block 0만 written으로 바꿔 I/O, 1~31은 unwritten
    file2 data fork  그대로 (block 0 -> 16408, 공유)

(3) I/O 완료: COW fork의 block 0을 data fork로 remap
    file2 data fork  block 0     ---> 16568        file2만 씀
                     block 1~159 ---> 16409~16567  공유, refcount 2
    file1 data fork  block 0     ---> 16408        file1만 씀
    file2 COW fork   block 1~31  ---> 16569~16599  남은 예약
```

code: xfs_reflink.c, xfs_inode_fork.c, xfs_inode_item.c, xfs_bmap.c, xfs_format.h

## reflink

reflink는 extent마다 transaction 하나로 dest의 mapping을 src의 block으로 바꾸고 refcount를 올린다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 입구와 준비

`FICLONE`은 VFS를 거쳐 XFS의 `->remap_file_range`인 `xfs_file_remap_range`로 들어온다. reflink feature가 없으면 여기서 `-EOPNOTSUPP`를 돌려준다. 그 뒤 준비, extent별 remap, dest inode 갱신을 차례로 한다. 준비 단계에서는 `xfs_reflink_remap_prep`이 remap transaction을 열기 전에 다음을 한다.

1. 두 inode의 IOLOCK과 MMAPLOCK을 잡는다.
2. realtime과 data를 섞지 않는지 보고, VFS 공통 검사를 한다.
3. dest에 쓸 위치가 EOF 뒤이면 옛 EOF와 그 위치 사이의 post-EOF preallocation을 0으로 채운다.
4. 두 inode에 REFLINK flag를 켜고, dest 범위의 page cache를 flush한 뒤 비운다.

REFLINK flag는 없을 때만 `tr_ichange` transaction 하나로 켠다. 한 번 켠 flag는 공유가 모두 사라져도 저절로 꺼지지 않는다. unshare, data fork와 COW fork를 다 비우는 truncate, file 전체를 바꾸는 exchange-range, scrub repair 같은 몇몇 경로에서만 끈다(code로만 확인).

### extent 하나의 remap

`xfs_reflink_remap_blocks`는 src의 extent를 하나씩 읽어 dest의 같은 위치에 같은 block을 map한다. src에 delalloc이 남아 있으면 corruption으로 본다. 준비 단계에서 flush했으므로 있을 수 없기 때문이다. extent 하나마다 `xfs_reflink_remap_extent`가 `tr_write` transaction 하나를 열고 그 안에서 다음을 한다.

1. dest의 현재 mapping을 읽는다. 두 mapping이 이미 같은 physical block이면 transaction을 cancel한다. 둘 다 unwritten이어도 그대로 둔다.
2. dest에 실제 block이 있었으면 그 unmap과 refcount 감소를 건다.
3. src extent가 written이면 refcount 증가와 map을 건다.
4. commit한다. refcount 변경은 CUI, bmap 변경은 BUI intent가 된다.

### refcountbt의 변화

640 KiB file1(block 160개, physical block 16408~16567)을 두 번 reflink하고, 한 사본을 다시 쓰고, 사본 둘을 지우면 refcountbt는 이렇게 바뀐다. record는 [startblock, blockcount, refcount, cowflag] 순서로 적었다.

| 단계 | refcountbt record | REFLINK flag | 사용 block |
|---|---|---|---|
| reflink 전 | 없음 | file1 0 | 13083 |
| file1을 file2로 reflink | `[16408,160,2,0]` | file1, file2 모두 1 | 13083 |
| file1을 file3으로 reflink | `[16408,160,3,0]` | 세 file 모두 1 | 13083 |
| file3 전체를 다시 쓰고 fsync | `[16408,160,2,0]` | 세 file 모두 1 | 13243 |
| file2, file3을 지움 | 없음 | file1 1 | 13083 |

- reflink는 공간을 더 쓰지 않고 refcount만 올린다.
- file3을 전부 다시 쓰면 새 block 160개가 잡히고 refcount가 3에서 2로 내려간다.
- 공유가 모두 사라지면 record도 빠지지만, file1의 REFLINK flag는 1로 남는다.

code: xfs_file.c, xfs_reflink.c, xfs_bmap.c, xfs_refcount.c, fs/remap_range.c

## 공유된 file에 쓸 때

공유된 block에 buffered write를 하면 XFS는 COW fork에 예약하고, writeback에서 새 block을 할당하고, I/O가 끝나면 그 block을 data fork로 옮긴다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. direct I/O와 crash 뒤의 회수는 code로만 확인했다.

### 4 KiB를 두 번 쓴 예

같은 조건의 새 file system에서 640 KiB file1을 file2로 reflink하고, file2의 block 0과 block 2에 4 KiB씩 쓴 뒤 fsync, freeze, umount를 차례로 하면 두 fork와 refcountbt는 이렇게 바뀐다.

| 시점 | file2 COW fork | file2 data fork | refcountbt |
|---|---|---|---|
| reflink 직후 | 비어 있음 | block 0~159 = 16408~16567, 공유 | 읽지 않음(refcountbt 예에서는 `[16408,160,2,0]`) |
| 첫 write(block 0) | block 0~31에 delalloc 32 block 예약 | 그대로 | 그대로 |
| writeback | block 0~31 = 16568~16599, unwritten. block 0만 written으로 바꿔 I/O(code로만 확인) | 그대로 | staging 32 block 추가 |
| I/O 완료 | block 1~31(31 block)이 남음 | block 0 = 16568 | 16408의 refcount 감소, staging 16568 해제 |
| 둘째 write(block 2)와 완료 | 남은 예약을 써서 block 1과 3~31(30 block)이 남음 | block 2 = 16570 | 16410의 refcount 감소, staging 16570 해제 |
| fsync 뒤 freeze 중 | 그대로 | 그대로 | 공유: `[16409,1,2,0]`(block 1), `[16411,157,2,0]`(block 3~159). staging: `[16569,1,1,1]`, `[16571,29,1,1]`(COW fork에 남은 30 block) |
| umount 뒤 | 30 block 모두 취소 | 그대로 | `[16409,1,2,0]`, `[16411,157,2,0]` |

- write 한 번에 COW fork에 32 block(128 KiB)을 예약했다. cowextsize hint(COW 예약을 넓히는 단위)를 주지 않아 기본값 32 block을 썼다.
- writeback은 write가 아니라 뒤이은 `bmap -v`가 일으켰다(추정). `bmap -v`는 delalloc이 남은 inode면 조회 전에 writeback을 한다. COW fork의 block은 COW fork에서 빠질 때까지 delalloc 수(`i_delayed_blks`)에 남으므로, 예약이 남아 있는 동안에는 `bmap -v`가 부를 때마다 writeback을 한다. 둘째 write도 그렇게 fsync 전에 끝나서 fsync 때는 COW에 관한 event가 없었다.
- 예약 32 block은 extent 하나로 할당됐다. remap transaction 안의 일은 unmap, refcount 감소, staging 해제, map 순서로 일어났다. 끝으로 COW fork에서 그 extent를 빼는 일은 memory에서만 한다. 이 일을 찍는 event는 켜지 않았고, `bmap -c`에서 block 0이 hole이 된 것으로 알 수 있다.
- 둘째 write는 새로 할당하지 않고 남은 31 block짜리 unwritten 예약을 썼다(`xfs_reflink_cow_found`의 `st 1`).
- umount 때 남은 예약 30 block(block 1 하나와 block 3~31의 29개)이 모두 취소되고 staging record도 사라졌다. 이 예약은 inactivation이 취소했다(추정). blockgc는 기본 5분 뒤에 도는데, 첫 예약에서 umount까지는 0.05초였다.

### 예약: buffered write

buffered write의 예약은 `xfs_buffered_write_iomap_begin`에서 한다. REFLINK flag가 있는 inode(또는 always_cow inode)이면 data fork에 extent가 없어도 COW fork를 함께 찾아 둔다. always_cow는 DEBUG kernel에서 sysfs로 켜는 설정이고, 켜고 mount하면 reflink file system은 덮어쓰기와 hole write를 COW fork로 보낸다. 주석에 따르면 이것으로 cowextsize를 쓰는 speculative preallocation을 구현해 공유 block의 이웃까지 unshare한다. 또 extent list를 보는 쪽이 refcountbt를 읽는 것보다 빠르다고 한다. REFLINK inode의 data fork extent에 쓰면 세 갈래로 나뉜다.

1. COW fork에 이미 offset을 덮는 extent가 있으면 그것을 쓴다(event `xfs_reflink_cow_found`).
2. 없으면 data fork extent가 공유인지 본다. written extent만 공유될 수 있고, refcountbt를 읽어 공유 구간의 경계로 mapping을 자른다(`xfs_reflink_trim_around_shared`). 공유가 아니면 data fork에 그대로 쓴다.
3. 공유이면 COW fork에 delalloc을 예약한다. post-EOF preallocation이 아니면 예약을 cowextsize hint(기본 32 block)에 맞춰 넓힌다. 넓힌 예약이 생기면 inode에 cowblocks tag를 붙이고, blockgc가 나중에 이 tag로 inode를 찾아 회수한다.

iomap에는 쓸 곳으로 COW mapping(`IOMAP_F_SHARED`)을, 읽을 곳(srcmap)으로 data fork의 mapping을 돌려준다. 그래야 block 일부만 쓰는 write가 read-modify-write를 할 때 옛 공유 block에서 읽어 온다. 이 밖에 zeroing은 hole을 덮은 COW extent가 있으면 그것을 쓸 수 있고, always_cow inode는 hole write도 COW fork에 예약한다. iomap의 호출 구조는 [iomap](17-iomap.md)과 [buffered write](18-buffered-write.md)에 있다.

### writeback: 할당과 written 변환

writeback은 COW fork의 delalloc을 unwritten extent로 할당하고, bio를 내기 직전에 이번 I/O 범위만 written으로 바꾼다.

- `xfs_map_blocks`는 offset이 COW fork의 extent에 덮이면 data fork를 보지 않고 COW fork를 쓴다. 그 extent가 delalloc이면 unwritten extent로 할당한다.
- COW fork extent를 할당할 때는 refcountbt에 staging record를 넣는 `XFS_REFCOUNT_ALLOC_COW` intent와, rmap에 `OWN_COW`를 더하는 변경을 따로 건다.
- bio 직전에는 `xfs_reflink_convert_cow`가 COW fork의 unwritten extent를 I/O 범위로 잘라 written으로 바꾼다. COW fork는 log하지 않으므로 transaction 없이 memory에서만 바꾼다. 같은 이름의 trace는 direct I/O 쪽에서만 나고, 예에서도 한 줄도 나오지 않았다.

### I/O 완료: remap

성공한 COW ioend는 transaction이 필요하므로 workqueue에서 `xfs_reflink_end_cow`로 끝낸다. I/O error이면 COW 범위와 data fork의 delalloc을 취소한다. remap은 extent마다 `tr_write` transaction을 하나씩 쓴다. 주석에 따르면 remap마다 bmbt block을 합치거나 나눌 수 있어서다. written인 COW extent만 옮기고, cowextsize로 넓힌 unwritten 부분은 건너뛴다. transaction 하나 안에서는 다음 네 가지를 한다.

1. 옛 data fork mapping을 unmap하고 그 block의 refcount를 줄인다. 2였던 refcount가 1이 되면 그 block은 refcountbt에서 빠진다.
2. staging record를 지우고(`XFS_REFCOUNT_FREE_COW` intent), rmap의 `OWN_COW`도 따로 지운다.
3. 새 block을 data fork에 map한다.
4. COW fork에서 그 extent를 뺀다. memory에서만 한다.

direct I/O도 같은 COW fork를 쓰되, delalloc 없이 곧바로 실제 block을 할당하고 완료 때 같은 `xfs_reflink_end_cow`를 부른다.

### 남은 예약의 회수

cowextsize로 넓힌 예약은 write가 끝난 뒤에도 COW fork에 남는다. 예약을 회수하는 길은 여럿이고, 대표적인 것은 아래와 같다.

| 경로 | 언제 | 취소하는 것 |
|---|---|---|
| blockgc | cowblocks tag가 붙은 inode에서, 기본 5분 뒤 | delalloc 예약과 unwritten extent. remap을 기다리는 written extent는 남긴다 |
| inactivation | VFS가 inode를 내보낼 때(umount 포함) | 전부 |
| truncate(close 때의 EOF 뒤 정리 포함), exchange-range, unshare 같은 동작 | 그 동작 때 | 자기 범위 |
| mount의 log recovery | crash 뒤 | refcountbt의 staging record 전부 |

collapse/insert와 I/O error 같은 몇몇 경로도 자기 범위의 예약을 취소한다(code로만 확인).

freeze는 inodegc와 blockgc를 멈출 뿐 예약을 취소하지 않는다. 그래서 freeze 중에 읽은 refcountbt에 staging record가 남아 있었다. crash 뒤에는 COW fork가 없으므로, log recovery 끝에서 staging record를 모두 찾아 해제한다(`xfs_reflink_recover_cow`). 주석에 따르면 mount 중에는 살아 있는 staging extent가 있을 수 없어 모두 지워도 된다. blockgc는 [fallocate와 mmap](21-fallocate-mmap.md)에, inactivation은 [unlink 뒤의 흐름](25-unlink-inactivation.md)에, log recovery는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.

code: xfs_iomap.c, xfs_aops.c, xfs_reflink.c, xfs_bmap.c, xfs_refcount.c, xfs_icache.c, xfs_inode.c, xfs_super.c, xfs_log_recover.c, xfs_file.c

## dedupe

dedupe는 reflink와 같은 XFS code로 block을 공유하며, 다른 점은 모두 VFS 공통부에 있다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

`FIDEDUPERANGE`는 VFS의 `vfs_dedupe_file_range`에서 dest마다 `->remap_file_range`를 `REMAP_FILE_DEDUP` flag와 함께 부른다. XFS에서는 reflink와 같은 `xfs_file_remap_range`로 들어오고, 그 아래의 block 공유는 reflink와 한 줄도 다르지 않다. 65573 byte(16 block과 37 byte)인 file로 dedupe와 reflink를 번갈아 하면 차이가 이렇게 나온다.

| 항목 | reflink | dedupe | 예 |
|---|---|---|---|
| 내용 비교 | 없다 | 두 범위를 write back한 뒤 page 단위로 비교한다. 다르면 `-EBADE`를 돌려주고, 사용자에게는 `FILE_DEDUPE_RANGE_DIFFERS`로 간다 | 내용이 다른 file에는 `Extents did not match.`가 나오고 XFS trace는 0줄이었다 |
| 길이 0 | EOF까지 | 바로 끝난다 |  |
| EOF의 부분 block | src 범위가 src EOF에서 끝나고 dest 범위가 dest EOF 이상에서 끝나면 공유한다. 아니면 `FICLONE`과 `FICLONERANGE`는 `-EINVAL`로 거절한다 | 두 범위가 모두 EOF에서 끝날 때만 공유한다. 아니면 길이를 block 단위로 줄인다 | 같은 크기의 두 file을 통째로 dedupe하자 37 byte짜리 마지막 block까지 17 block이 extent 하나로 공유됐다 |
| dest를 연 방식 | 쓰기로 열려 있어야 한다. 아니거나 `O_APPEND`이면 `-EBADF` | 쓰기로 열 필요가 없다. `CAP_SYS_ADMIN`, 쓰기 open, file 소유자, 쓰기 권한 중 하나면 된다 | O_RDONLY로 연 dest에 dedupe는 성공했고, reflink는 `Bad file descriptor`로 실패했다 |
| dest의 mtime과 ctime | 갱신하고 suid도 지운다(`file_modified`) | 그대로 둔다. 내용이 같아 수정으로 보지 않는다 | reflink 뒤에는 두 시간이 `00:31:48`에서 `00:31:50`으로 바뀌었고, dedupe 뒤에는 그대로였다 |

- 내용 비교는 XFS의 준비 단계에서 두 inode의 IOLOCK과 MMAPLOCK을 잡은 뒤에 한다. 그래서 비교와 공유 사이에 다른 write가 끼지 못한다(code로만 확인).
- XFS가 REFLINK flag를 켜는 transaction은 timestamp를 건드리지 않는다(code로만 확인).
- 이미 공유된 범위를 다시 dedupe하면 내용은 다시 비교하지만, extent마다 두 mapping이 같은 physical block이라 transaction을 cancel한다. 같은 dedupe를 되풀이하자 refcount와 bmap 변경 없이 remap trace 세 줄만 나왔다.
- dedupe로 공유된 block에 쓰면 COW 경로를 그대로 탄다.

code: fs/remap_range.c, fs/read_write.c, xfs_file.c, xfs_reflink.c

## exchange-range

exchange-range는 data를 복사하지 않고 두 file의 같은 길이 범위에 있는 extent mapping을 맞바꾼다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다. crash 뒤의 recovery는 code로만 확인했다.

### 입구와 조건

exchange-range는 `XFS_IOC_EXCHANGE_RANGE`, `XFS_IOC_START_COMMIT`, `XFS_IOC_COMMIT_RANGE` 세 ioctl로 들어온다. flag에는 `TO_EOF`, `DSYNC`, `DRY_RUN`, `FILE1_WRITTEN`이 있다. 범위를 EOF까지(`TO_EOF`) 잡으면 file 크기도 함께 바뀐다. 사용자에게 보이는 주요 조건은 다음과 같다.

- 두 file은 같은 mount의 regular file이어야 하고, 둘 다 읽기와 쓰기로 열려 있어야 한다. `O_APPEND`는 안 된다.
- 시작 offset과 길이는 allocation unit(realtime file이 아니면 fs block 하나, realtime file이면 rt extent 하나)에 맞아야 한다. 범위가 EOF에서 끝나면 끝의 조각은 받는다.
- exchange-range feature가 있어야 한다(없으면 `-EOPNOTSUPP`). reflink feature는 없어도 된다.

이 밖에 허용하지 않는 flag와 두 범위도 검사하고, fd나 inode가 막지 않으면 두 file의 시간을 갱신하라는 내부 flag(trace의 `CMTIME1|CMTIME2`)를 켠다.

한쪽에 REFLINK flag가 켜져 있으면 다른 쪽에도 켠다. 공유 block이 반대 file로 옮겨 갈 수 있기 때문이다. file 전체를 바꿀 때는 교환 뒤 원래 flag가 있던 쪽을 꺼서 flag도 맞바꾼다.

### 원자성

이 원자성은 다른 thread에게는 lock으로, crash에는 intent chain으로 지킨다. 설계 문서 xfs-online-fsck-design.rst에는 이렇게 적혀 있다. "it can guarantee that the caller never sees a mix of old and new contents even after a crash, and it can operate on two arbitrary file fork ranges."

- lock: `xfs_exchrange_contents`가 두 inode의 IOLOCK과 MMAPLOCK을 처음부터 끝까지 잡고 있다. 그 안에서 `tr_write` transaction 하나를 연다. 두 inode의 ILOCK은 lock flag 0으로 join하므로, commit이 transaction을 roll하는 동안에도 ILOCK이 풀리지 않는다. ILOCK은 commit이 deferred operation chain을 끝까지 실행한 뒤에야 놓는다.
- intent chain: 첫 transaction은 mapping을 하나도 바꾸지 않고 XMI(file mapping exchange intent, 두 inode 번호와 두 시작 offset, 남은 block 수, flag, 최종 크기를 담는다) 하나만 log에 넣는다. 그 뒤 step마다 `xfs_exchmaps_finish_one`이 다음에 바꿀 mapping 쌍을 찾아 두 mapping을 unmap하고 offset을 맞바꿔 다시 map한다. 이 변경은 BUI 네 개(unmap 둘, map 둘)가 된다. 남은 일이 있으면 `-EAGAIN`으로 새 transaction과 갱신된 XMI를 요구한다.
- crash 뒤의 log recovery는 마지막 XMI에서 이어 끝까지 간다. 중간 상태가 log에 남아도 mount가 끝나면 완료 상태가 되고, 되돌리지는 않는다.

TO_EOF이면 마지막 step에서 on-disk 크기를 바꾸고, commit 뒤에 memory에 있는 두 inode의 크기를 맞바꾼다. 두 file의 mtime과 ctime은 첫 transaction에서 바꾼다. durability는 이와 따로 봐야 한다. `DSYNC` flag가 있거나 wsync mount일 때만 commit을 동기로 한다. 아니면 commit이 CIL에 들어갈 뿐이라, crash하면 요청 전체가 없던 일이 될 수 있다(code로만 확인). 그래도 "all or nothing"은 지켜진다. intent와 recovery의 일반 구조는 [deferred operation, intent, log recovery](13-intent-recovery.md)에, log force는 [fsync](20-fsync.md)에 있다.

### 두 구간만 바꾸는 commit

1 MiB 원본(block 24~279에 연속)에 대해 temp file을 만들어 두 구간만 쓰고 commit하면 step 둘로 끝난다. temp file에는 file block 16~31(64 KiB~128 KiB)과 192~207(768 KiB~832 KiB)에만 data가 있다. commit은 `FILE1_WRITTEN`으로 해서 temp file의 hole과 unwritten 구간은 건너뛴다.

| 시점 | 바꾼 file block | 원본이 받은 block | temp file이 받은 block | 그 뒤의 XMI(시작 file block, 남은 block) |
|---|---|---|---|---|
| 첫 transaction | 없음 |  |  | 0, 256 |
| step 1 | 16~31 | 536~551 | 40~55 | 32, 224 |
| step 2 | 192~207 | 552~567 | 216~231 | 208, 48 |

- 원본은 64 KiB~128 KiB와 768 KiB~832 KiB만 temp file의 내용으로 바뀌었고, 나머지는 원래 block 그대로였다. temp file의 hole 구간 때문에 원본이 hole이 되지는 않았다.
- step마다 BUI 넷(unmap 둘, map 둘)이 나왔다. trace에는 step의 남은 XMI가 그 step의 BUI 넷보다 먼저 찍힌다. 남은 XMI는 step 끝에서 걸 때 찍히고 BUI는 처리할 때 찍히기 때문이다. BUI 넷은 처리 중에 새로 생긴 일이라 남은 XMI보다 먼저 처리된다(code로만 확인).
- 마지막 XMI에 남은 48 block은 temp file에서 hole이라 바꿀 것이 없었다.

### commit-range

commit-range는 START_COMMIT과 COMMIT_RANGE 사이에 원본에 쓴 사람이 없을 때만 성공한다.

- `XFS_IOC_START_COMMIT`은 freshness blob을 돌려준다. blob에는 원본의 inode 번호, generation, mtime, ctime이 들어 있다. 시간을 채우는 `fill_mg_cmtime`의 주석에 따르면, 그 뒤에 write가 있으면 ctime이 다른 값으로 남는다.
- `XFS_IOC_COMMIT_RANGE`는 blob을 되받아 magic과 fsid를 확인하고, 네 값 가운데 하나라도 지금과 다르면 `-EBUSY`로 거절한다. 이 검사는 IOLOCK과 MMAPLOCK을 잡은 뒤에 하므로, 검사와 교환 사이에 새 write가 끼지 못한다(code로만 확인).
- 검사를 통과한 뒤에는 다른 exchange-range 요청과 같이 두 범위를 flush하고 page cache를 비운 다음 두 file의 COW fork 예약을 취소한다. 주석에 따르면 flush 뒤에 남은 COW mapping은 speculative 예약뿐이다.

예를 들어 START_COMMIT 때 원본의 mtime은 `1792715698:545634651`이었다. 그 사이 다른 process가 offset 512 KiB에 4 KiB를 쓰자 commit 때는 `1792715700:129599450`이 되어 `EBUSY`로 거절됐다. 원본에서 바뀐 곳은 중간 write의 4 KiB뿐이고, temp file에 쓴 내용은 들어가지 않았다.

code: xfs_exchrange.c, xfs_exchmaps.c, xfs_exchmaps_item.c, xfs_trans.c, xfs_defer.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| `xfs_reflink_remap_prep` 위의 주석 | "For dedupe, we can simply round the length to dedupe down to the previous whole block and ignore the partial EOF block ... we can't dedupe the last block of a file" | 지금 VFS code는 두 file이 같은 크기로 EOF에서 끝나면 마지막 부분 block도 공유 대상에 넣는다. 65573 byte file 둘을 dedupe하자 17 block이 모두 공유됐다. |

## 한계점

- COW 예의 writeback을 `bmap -v`가 일으켰다고 본 근거는 code의 조건과 PID 순서뿐이다. trace에는 writeback을 시작한 함수나 명령 이름이 찍히지 않는다.
- 두 write를 따로 연 xfs_io로 해서 그 사이에 close의 EOF 뒤 정리가 끼었다. code로는 이 정리가 EOF 뒤만 다루고, 거기 있는 COW 예약도 함께 취소한다. 이 예의 예약은 모두 EOF 안이라 영향을 받지 않는다. 실제로 둘째 write는 첫 write의 예약을 그대로 찾았지만, close가 무엇을 했는지는 trace로 보지 않았다.
- delalloc 변환 때 할당 길이를 정하는 code는 읽지 않았다. 예에서 32 block이 extent 하나로 할당된 것만 확인했다.
- always_cow 변형, direct I/O의 COW 경로, crash 뒤의 staging record 회수와 XMI recovery는 관찰하지 않았다. crash를 만들려면 shutdown이나 DEBUG 전용 errortag가 필요하다.
- exchange-range의 마지막 XMI에 남은 48 block에서 step이 더 없었는지는 trace로 따로 확인하지 않았다. temp file의 `bmap`에서 그 구간이 hole인 것과 맞을 뿐이다.
- exchange-range 뒤 temp file이 받은 원본의 옛 block이 temp file이 닫힐 때 풀리는지는 보지 않았다.
- code의 lock 순서를 보고 dedupe의 비교와 공유 사이, commit-range의 검사와 교환 사이에 write가 끼지 못한다고 보았다. 경쟁 상황을 만들어 확인하지는 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 새로 만들었다. refcountbt 관찰과 COW 관찰은 refcountbt 전체를 `xfs_db`의 `agf 0` 하나로 읽으려고 `agcount=1`로, dedupe 관찰과 exchange-range 관찰은 기본값(`agcount=4`)으로 만들었다. 넷 모두 `crc=1 rmapbt=1 reflink=1`이다. refcountbt 관찰과 exchange-range 관찰에서는 mkfs 출력에 exchange도 찍었고, 둘 다 `exchange=1`이었다. exchange-range 관찰에서는 `parent=1`도 찍었다.
- COW 관찰만 debug image(`7.1.0-rc5-xdebug-g8fde5d1d47f6`)에서 했다. 관찰 image에서 처음 실행했을 때 `bmap -cv`(COW fork를 보는 `XFS_IOC_GETBMAPX`)가 모두 `Invalid argument`로 실패했다. DEBUG가 없는 kernel의 `xfs_getbmap`은 COW fork 조회를 거절한다. COW 관찰의 mount option은 `rw,relatime,inode64,logbufs=8,logbsize=32k,noquota`였다.
- 관찰마다 한 번 실행했다. COW 관찰과 exchange-range 관찰은 다시 실행한 출력을 썼다. exchange-range 관찰의 첫 실행에서는 중간 write가 낀 commit의 temp file을 `truncate 1m` 하지 않았다. 그래서 trace에는 `xfs_exchrange_prep`(temp file 크기 128 KiB, `isize 0x20000`)와 `error -22`만 남고, freshness 검사까지 가지 못했다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| refcountbt 관찰 | 640 KiB file1을 fsync하고 `cp --reflink=always`로 file2와 file3을 만들었다. 그 뒤 file3을 다시 쓰고 file2와 file3을 지웠다. 단계마다 unmount해서 `xfs_db`로 refcountbt의 root block(`refcntlevel`이 1이라 btree 전체)과 inode의 REFLINK flag를 읽고 다시 mount했다. 사용 block은 `df -B4096`의 사용량이다 | 없음 | refcountbt의 변화 |
| COW 관찰 | 같은 방법으로 만든 file2에 offset 0과 8 KiB에 4 KiB씩 쓰고, `fsync`, `xfs_freeze -f` 중의 `xfs_db`, `umount`를 차례로 했다. write와 fsync 뒤마다 `bmap -v`와 `bmap -cv`를 이 순서로 실행했다 | `xfs_reflink_trim_around_shared`, `xfs_reflink_cow_found`, `xfs_iomap_alloc`, `xfs_iomap_found`, `xfs_reflink_convert_cow`, `xfs_reflink_end_cow`, `xfs_reflink_cow_remap_from`, `xfs_reflink_cow_remap_to`, `xfs_refcount_deferred`, `xfs_bmap_deferred`, `xfs_reflink_cancel_cow_range`, `xfs_reflink_cancel_cow` | 도식, 4 KiB를 두 번 쓴 예 |
| dedupe 관찰 | 65573 byte file 다섯 개를 만들었다(file3만 byte 0x62, 나머지는 0x61). file1을 file2로 통째로 dedupe, 같은 dedupe 반복, 내용이 다른 file3으로 dedupe, `xfs_io -r`(O_RDONLY)로 연 file4로 dedupe, 같은 방식으로 연 file5로 reflink, 쓰기로 연 file5로 reflink를 차례로 했다. 시간은 `stat`의 mtime과 ctime이고, 첫 dedupe 전과 마지막 reflink 전에 1.1초씩 쉬었다 | `xfs_reflink_remap_range`, `xfs_reflink_remap_extent_src`, `xfs_reflink_remap_extent_dest`, `xfs_reflink_set_inode_flag`, `xfs_refcount_deferred`, `xfs_bmap_deferred` | dedupe 표 |
| exchange-range 관찰 | fstests xfs/792의 순서에서 crash 주입(`inject bmap_finish_one`)과 cycle mount를 뺐다. 1 MiB file a(전부 0x58)에 `startupdate -e`, `truncate 1m`, temp file의 64k에 0x59와 768k에 0x57 쓰기, `sync`, `commitupdate -h -k`를 했다. 중간 write가 낀 commit은 같은 1 MiB file b에 `startupdate -e`, `truncate 1m`, temp file 쓰기를 한 뒤 3초 기다리는 xfs_io를 띄워 두고, 1.5초 뒤에 다른 xfs_io가 b의 512k에 0x41을 4 KiB 쓰게 했다 | `xfs_exchrange_prep`, `xfs_exchrange_freshness`, `xfs_exchrange_flush`, `xfs_exchrange_mappings`, `xfs_exchrange_error`, `xfs_exchmaps_defer`, `xfs_exchmaps_update_inode_size`, `xfs_bmap_deferred` | 두 구간만 바꾸는 commit, commit-range의 예 |

### 출력 읽는 법

- `bmap -v`의 주소와 길이는 512 byte 단위라 4 KiB block 하나가 8로 나온다. 본문의 block 번호는 이 값을 8로 나눈 것이다(`132544`는 block 16568). 끝의 `100000`은 공유 extent를 뜻한다. `-p` 없이 찍으면 unwritten extent를 따로 표시하지 않는다.
- trace의 `startblock`, `gbno`, `fileoff`, `fsbcount`는 4 KiB block 단위의 16진수다(`0x40b8`는 block 16568). `st`는 extent 상태이고 1이 unwritten이다. `bmap -v`도 extent마다 공유 여부를 보느라 `xfs_reflink_trim_around_shared`를 낸다.
- COW 관찰의 phase A~E는 첫 write, 둘째 write, fsync, freeze, umount 구간이다. trace에는 명령 이름이 찍히지 않는다. 그래서 첫 write의 할당을 찍은 `xfs_io-972`는 write(`xfs_io-971`) 다음에 실행한 `bmap -v`로 보았다(명령 순서와 PID 순서로 본 추정).
- COW 관찰에서 write 뒤의 `bmap -cv`는 `bmap -v` 다음에 실행했고, 그때는 writeback이 이미 끝나 있어 delalloc 예약이 나오지 않는다. 4 KiB를 두 번 쓴 예의 표에서 첫 write 행은 `xfs_iomap_alloc` trace에서 읽었다.
- dedupe 관찰의 phase A~F는 관찰 목록의 여섯 동작을 차례로 가리킨다. xfs_io는 내용이 달라 dedupe가 거절돼도 종료 값 0을 돌려주므로 메시지로 판정했다. reflink 뒤에 찍는 `linked 0/0 bytes`는 xfs_io의 표시라 해석하지 않았다.
- exchange-range 관찰의 phase A는 두 구간만 바꾸는 commit이다. 중간 write가 낀 commit의 trace는 중간 write 직전에 찍은 표시(B2) 뒤에 있다. `xfs_exchrange_freshness`는 `->` 앞에 지금 원본 inode의 mtime과 ctime을, 뒤에 START_COMMIT 때 blob에 담긴 값을 찍는다.
- xfs_io의 `startupdate -e`는 같은 directory에 O_TMPFILE 빈 file을 만들고 원본을 file2로 삼아 START_COMMIT한다(`-e`가 없으면 원본을 복제한 file을 쓴다). `commitupdate`는 기본으로 TO_EOF를 켠다. `-h`는 `FILE1_WRITTEN`을 켜고 `-k`는 TO_EOF를 끈다.

### 원본 출력: refcountbt의 변화

(원본 출력 "reflink 전후의 refcountbt record와 inode의 REFLINK flag": `raw/26-reflink-cow.md`)

### 원본 출력: 4 KiB를 두 번 쓴 예

(원본 출력 "공유된 file2의 data fork와 COW fork 변화": `raw/26-reflink-cow.md`)

(원본 출력 "write, fsync, freeze, umount 구간의 tracepoint": `raw/26-reflink-cow.md`)

(원본 출력 "freeze 중과 umount 뒤의 refcountbt": `raw/26-reflink-cow.md`)

### 원본 출력: dedupe

(원본 출력 "dedupe와 reflink 명령의 실행 결과": `raw/26-reflink-cow.md`)

(원본 출력 "dedupe 구간별 tracepoint": `raw/26-reflink-cow.md`)

### 원본 출력: 두 구간만 바꾸는 commit과 commit-range

(원본 출력 "xfs/792 순서와 중간 write가 낀 commit의 실행 결과": `raw/26-reflink-cow.md`)

(원본 출력 "exchange-range 구간별 tracepoint": `raw/26-reflink-cow.md`)
