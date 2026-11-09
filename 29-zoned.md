# zoned XFS

> XFS 전반 wiki, 묶음 "기능". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/29-zoned.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/29-zoned.md, 접힌 원본 출력은 raw/29-zoned.md.

## 개요

- zoned XFS에서는 rt device의 zone 하나가 rtgroup 하나가 된다. rtbitmap과 rtsummary가 없고, zone마다 write pointer와 살아 있는 block 수(used)로 공간을 관리한다. data는 제자리에 덮어쓰지 않고 늘 새 위치에 쓴다.
- write는 공간의 양만 예약하고, zone과 zone 안 위치는 bio를 내기 직전에 정한다. zone을 고를 때는 file의 write hint가 수명이 비슷한 data를 한 zone에 모으는 단서가 된다.
- 덮어쓰거나 지운 block의 자리는 zone을 reset해야 다시 쓸 수 있다. 그래서 공간을 "사용자가 더 채울 수 있는 양"과 "바로 쓸 수 있는 양" 둘로 세고, 바로 쓸 수 있는 block이 모자라면 zone GC가 살아 있는 block이 적은 zone을 비워 reset한다.

예의 값은 null_blk로 만든 host-managed zoned device(zone 16개, zone마다 16384 block)를 rt device로, 1 GiB loop device를 data device로 써서 만든 file system에서 얻었다.

**도식: zone 상태의 순환**

```text
  FREE
    |  첫 write가 zone을 연다 (GC zone도 free zone에서 연다)
    v
  OPEN   동시에 열 수 있는 zone 수에 한도가 있다
    |  written이 zone 끝(16384 block)에 닿는다
    v
  FULL
    |  덮어쓰기, truncate, unlink로 used가 준다
    v
  reclaimable   bucket = 10 x used / 16384
    |  GC가 used가 가장 적은 zone을 victim으로 골라
    |  살아 있는 block을 GC zone으로 옮긴다
    v
  used 0   (GC 없이 used가 0이 된 zone도 여기로 온다)
    |  reset: cache flush, rmap log force, ZONE_RESET
    v
  FREE
```

## zone과 rtgroup

XFS는 superblock의 incompat feature bit로 file system이 zoned인지를 보고, 그 안에서 realtime flag가 선 inode만 zoned 경로로 보낸다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### zoned inode

- data device에 있는 file은 보통의 XFS 경로를 탄다. mkfs는 zoned file system의 root에 rtinherit를 단다. 그 아래의 새 regular file은 realtime flag를, 새 directory는 rtinherit를 물려받는다. 그래서 root 아래의 file은 따로 정하지 않으면 zoned 경로로 간다(실험에서 root의 xflags는 `t`, 새 file은 `r`).
- XFS는 zoned inode를 늘 COW로 다루므로 fallocate의 preallocation은 `-EOPNOTSUPP`로 실패한다. close 때 EOF 뒤의 block을 정리하는 일도 건너뛰는데, 주석에 따르면 zoned rt device의 inode에는 preallocation이 없기 때문이다.

### mount 조건과 on-disk 구조

zoned file system을 mount하려면 rt device, rtgroup, rmapbt, metadir가 있어야 한다. 또 rt extent가 1 block이고 zone이 5개 이상이어야 한다. zone capacity가 rtgroup 크기(rgextents)가 되고, rtgroup과 metadir는 [realtime, rtgroup, metadir](28-realtime.md)의 것을 그대로 쓴다. reflink는 거부한다. zone GC가 data를 옮길 때 공유 block을 복제해 버리기 때문이다(주석). 실험에서 mkfs는 `-r rtdev=`만으로 `zoned=1`, `metadir=1`, `reflink=0`, `rgcount=16 rgsize=16384`를 골랐다.

zoned에는 rtbitmap, rtsummary, rtsb가 없고, rtgroup lock은 rmap inode lock만 쓴다. 1 MiB file 하나를 쓰고 umount한 뒤 `xfs_db`로 보면 다음과 같다.

| 본 곳 | 값 | 뜻 |
|---|---|---|
| superblock의 `rbmblocks` | 0 | bitmap과 summary가 없다(`rbmino`와 `rsumino`는 metadir에서 늘 0) |
| metadir의 `/rtgroups` | `0.rmap`, `1.rmap`, ... | rtgroup마다 rt rmap inode만 있다 |
| `rtsb` | CRC 오류 | rtsb가 없어 rt device의 첫 block은 zone 0의 첫 data block이다(f1을 그 자리에 썼다) |
| zone 0의 rmap inode | format `meta_btree`, metatype `rtrmap`, `v3.cowextsize = 256` | 256이 zone 0의 used다. zone 1과 15에서는 0 |

rt rmap inode에서는 일반 file의 `di_cowextsize` 자리를 union으로 같이 써서 used를 둔다(`di_used_blocks`). verifier는 이 값이 rtgroup 크기보다 크면 거부하고, zone GC는 이 값으로 victim을 고른다.

### mount 때 zone 상태

mount는 zone마다 write pointer와 used를 읽어 상태를 정한다. write pointer는 zoned block device면 장치가 알려 준 값이고, zone 크기나 capacity가 rtgroup 크기와 다르면 mount가 실패한다. conventional zone이나 일반 device면 rt rmap의 가장 높은 block 번호 + 1로 추정하는데, 이 추정은 전원이 나갔을 때 rmap에 기록되지 않은 block을 놓칠 수 있다(주석).

| write pointer | used | 상태 |
|---|---|---|
| 0 | 0 | free zone(`XFS_RTG_FREE` mark) |
| 중간 | write pointer 이하 | open zone으로 되살린다. hint는 `WRITE_LIFE_NOT_SET` |
| 아무 값 | write pointer나 zone 용량보다 큼, 또는 write pointer가 0인데 0이 아님 | 경고를 찍고 -EFSCORRUPTED로 mount가 실패한다(code로만 확인) |
| 끝 | 0 | reset 전에 전원이 나간 것으로 보고 그 자리에서 reset |
| 끝 | 1 이상, zone 크기보다 작음 | reclaimable(zone GC 후보) |

동시에 쓸 수 있는 open zone 수는 세 번에 걸쳐 정해진다.

| 단계 | 규칙 | 실험(zone 16개) |
|---|---|---|
| 처음 값 | mount option `max_open_zones`, 없으면 device 한도, 그것도 없으면 128. device 한도를 넘지 못한다 | null_blk는 한도를 주지 않는다 |
| zone 수 상한 | `(rgcount - 5) / 2 + 1 + 1`과 `rgcount / 4` 중 작은 값, 최소 2 | min(7, 4) = 4 |
| GC 몫 | 한 자리를 GC용으로 뗀다 | 사용자 몫 3(sysfs `zoned/max_open_zones`). `max_open_zones=2`로 mount하면 1 |

zone 크기의 배수로 쓰는 file이 쪼개져 GC가 필요해지지 않게 하려고 writeback 단위(`s_min_writeback_pages`)를 zone 크기까지 키운다(주석).

code: xfs_zone_alloc.c, xfs_zones.c, xfs_rtgroup.c, xfs_inode_util.c, xfs_inode_buf.c, xfs_super.c, xfs_file.c

## write 한 번의 경로

block 위치는 I/O가 끝난 뒤에야 data fork에 들어간다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 1 MiB direct write 하나

사용자 open zone 한도가 3인 file system에서 hint 없는 1 MiB file d1~d4를 차례로 direct로 쓰면, d1~d3이 zone 0, 1, 2를 하나씩 연다. 넷째 file d4는 이렇게 처리된다.

| 단계 | d4에서 |
|---|---|
| 공간 예약 | user free와 user available 두 counter에서 각각 예약한다. 양은 259 block(256 + 1 + 2, code로만 확인) |
| mapping | 주소 없는 mapping. block을 할당하지 않는다 |
| zone 고르기 | open 한도 3이 차서 list 맨 앞의 zone 0(이유는 code로 해석) |
| zone 안 위치 | allocated 256 -> 512, 시작 block 256 |
| 제출 | zone append. 주소는 장치가 정한다 |
| 완료 | transaction 하나로 기록 위치 256을 data fork에 걸고, used와 written 256 -> 512 |

**도식: zone 안의 두 위치**

```text
zone 0 (block 0 ~ 16383)

0            written      allocated                  16384
|============|............|------------------------------|

===  기록이 끝난 block (data fork와 rmap에 들어감, 기록할 때 used에 더함)
...  자리만 잡고 bio가 진행 중인 block
---  아직 쓰지 않은 곳
bio가 하나일 때의 그림이다. 여럿이 동시에 나가면
===와 ...가 섞일 수 있다(code로만 확인).

d4 bio를 낼 때:   written 256, allocated 256 -> 512
d4 I/O가 끝난 뒤: written 256 -> 512, used 256 -> 512
```

### write 시점: 주소 없는 mapping

- direct write는 block 정렬이면 공간을 예약한 뒤 `iomap_dio_rw`를 돈다. `->iomap_begin`은 주소 없는 mapping만 준다(rt device, `IOMAP_MAPPED`, `IOMAP_F_ANON_WRITE`).
- buffered write는 delalloc을 data fork가 아니라 COW fork에 넣는다. mapping은 `IOMAP_F_SHARED`로 시작한다. 넣은 delalloc만큼은 사용자 공간 예약에서 뺀다. block 정렬이 아닌 write나 zero에는 data fork의 extent를 srcmap으로 같이 줘서 read-modify-write를 하게 한다.

delalloc 일반론은 [buffered write](18-buffered-write.md), iomap ops의 틀은 [iomap](17-iomap.md), direct I/O 쪽은 [direct I/O와 read](19-direct-io-read.md)에 있다.

### bio를 낼 때: zone과 위치

buffered writeback은 COW fork의 delalloc을 지우고 주소 없는 mapping으로 bio를 만든다. direct write는 bio 크기만큼 예약에서 빼고(넘으면 shutdown) ioend를 만든다. 그다음 둘 다 `xfs_zone_alloc_and_submit`에서 위치를 정한다.

1. write context에 zone이 없으면 inode에 cache된 zone을 쓰고, 없으면 골라 cache한다. cache에 있는 동안 그 open zone 구조를 붙잡아 둔다.
2. `oz_allocated`를 올려 자리를 잡는다. 길이는 요청 길이, `XFS_MAX_BMBT_EXTLEN`, zone에 남은 양 중 가장 작은 값이고, 0이면 zone이 찬 것이라 다시 고른다.
3. 받은 양이 ioend보다 작으면 ioend를 잘라 앞부분을 내고, 나머지는 다시 고른다.
4. sequential zone이면 `REQ_OP_ZONE_APPEND`로 바꿔 zone 시작 sector로 낸다. 아니면 sector에 `oz_allocated` 위치를 더한다.

### I/O 완료: mapping 걸기

zone append가 끝나면 장치가 돌려준 sector를 ioend에 적는다. 이어서 extent마다 transaction 하나(`tr_write`, `XFS_TRANS_RESERVE`와 `XFS_TRANS_RES_FDBLKS`)를 열고, 그 안에서 다음을 한다.

- 지금 data fork의 mapping을 읽어 그 extent 길이로 자른다.
- 옛 block이 있으면 unmap하고 free를 미룬다. 이 free가 옛 zone의 used를 줄인다.
- rt rmap inode의 used와 zone의 `oz_written`을 올리고, 새 extent를 data fork에 건다.

그래서 zone 안 위치는 trace에 두 번 나온다. `xfs_zone_alloc_blocks`의 rgbno는 잡아 둔 위치, `xfs_zone_record_blocks`의 rgbno는 기록된 위치다. 같은 zone에 bio가 동시에 여러 개 나가면 둘이 다를 수 있다(code로만 확인). 실험에서는 닫힌 file 넷을 `sync`로 한꺼번에 낸 경우에도 둘이 같았다.

`oz_written`이 zone 크기에 닿으면 open 수를 줄이고 기다리는 writer를 깨운다. 그 zone에서 이미 free된 block이 있으면 바로 reclaimable로 넘긴다.

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_zone_alloc.c

## 공간 예약과 ENOSPC

zoned XFS는 공간을 counter 둘로 센다. `XC_FREE_RTEXTENTS`(mountstats의 user free)에는 사용자가 더 채울 수 있는 양이, `XC_FREE_RTAVAILABLE`(user available)에는 GC를 기다리지 않고 바로 쓸 수 있는 block 수가 들어간다. free된 block은 zone을 reset하기 전까지 다시 쓸 수 없어서 두 값이 달라진다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 두 counter의 움직임

rt device가 262144 block(zone 16개)인 file system에서 두 counter는 이렇게 움직였다. 둘째 행은 같은 조건의 새 file system에서 얻었다.

| 시점 | user free | user available | free zone |
|---|---|---|---|
| mount 직후 | 196608 | 212992 | 16 |
| 1 MiB를 하나 쓴 뒤 | 196352 | 212736 | 15 |
| 1 MiB direct write로 ENOSPC까지(767 MiB) | 256 | 16640 | 4 |
| 이어서 buffered 8 MiB write(1 MiB만 씀) | 0 | 16384 | 4 |
| 두 file을 지운 직후 | 0 | 16384 | 4 |
| 지우고 2초 뒤 | 196608 | 196864 | 15 |

- mount 때 available은 free zone 전부와 open zone의 남은 부분이고, rtextents는 거기에 reclaimable zone 몫을 더한 값이다. read-write mount 때는 GC와 RESERVED 예약(fallocate의 punch, zero, collapse와 truncate)이 쓰는 reserve pool로 rtextents에서 4 zone(GC 몫 3 zone + 1)과 `sb_rtreserved`(mkfs `-r reserved=`, 기본 0)를, available에서 GC 몫 3 zone을 떼어 둔다. 그래서 mount 직후 user free는 196608(= 262144 - 65536)이고, available은 user 212992와 reserved 49152를 더해 262144다.
- rt를 물려받는 directory나 rt file의 statfs에는 rt 값이 나온다. `f_bfree`는 rtextents이고 `f_blocks`는 `sb_rblocks`에서 rtextents 예약을 뺀 값이라, df 크기에서 이 예약 4 zone이 빠진다(root에서 `blocks=196608`). rtinherit를 뗀 directory에서는 data device 값(`blocks=245760`)이 나온다. available은 df에 나오지 않는다(code로만 확인).

### 예약 흐름

1. rtextents에서 뺀다. 모자라고 기다려도 되는 호출이면 inodegc를 flush한 뒤 한 번 더 시도한다. 지운 file의 block이 inactivation을 기다리느라 아직 free로 돌아오지 않았을 수 있기 때문이다(code로만 확인). inodegc는 [unlink 뒤의 흐름](25-unlink-inactivation.md)에 있다.
2. 그래도 모자라고 `XFS_ZR_GREEDY`면 남은 만큼만 받는다. 여기서도 실패하면 ENOSPC를 돌려준다.
3. available을 받는다. 기다리는 writer가 없거나 RESERVED 예약이면 먼저 바로 빼 본다. 빼지 못했거나, RESERVED가 아닌데 기다리는 writer가 있으면, NOWAIT는 `-EAGAIN`으로 끝내고 나머지는 대기 list 끝에 들어가 뺄 수 있을 때까지 GC를 깨우고 잔다. reclaimable zone도 없고 GC도 돌지 않으면 ENOSPC, kill signal이면 `-EINTR`, shutdown이면 `-EIO`로 끝난다.
4. 실패하면 앞서 뺀 rtextents를 돌려놓는다. 쓰고 남은 예약은 write 뒤에 돌려준다.

| 호출자 | 양 | flag | 모자랄 때 |
|---|---|---|---|
| direct write | write 길이를 block으로 올림 + 1(시작 offset을 보지 않고 올린 몫) + 2(옛 EOF block과 새 시작 block의 zeroing) | 0(`IOCB_NOWAIT`면 NOWAIT) | rtextents면 ENOSPC, available이면 기다림 |
| buffered write | 같음 | GREEDY | 받은 만큼 짧은 write. iomap 쪽 ENOSPC면 `xfs_flush_inodes` 뒤 한 번 더 |
| write fault | folio 크기 | 0 | fault error |
| fallocate의 punch, zero, collapse | 2 block | RESERVED | 가득 찬 file system에서도 reserve pool로 hole punch를 할 수 있다 |
| truncate | 1 block | NOWAIT, RESERVED | `-EAGAIN`을 ENOSPC로. VFS 호출 규약상 기다릴 수 없다(주석) |

위 ENOSPC 행에서 dd는 768번째 write에서 끝났다. 767 MiB(196352 block)를 쓰고 남은 256 block으로는 한 번의 예약 259 block(256 + 1 + 2)을 받지 못한다(trace의 `delta 259`). 이어진 buffered 8 MiB write는 2051 block(2048 + 3)이 필요했지만 GREEDY로 256 block만 받아 1 MiB만 썼다.

### 돌아오는 경로

- block을 free하는 처리(EFI)는 rtextents만 돌려준다. available은 zone reset이 끝날 때 zone 단위로 돌아온다.
- writeback 전에 지운 delalloc(truncate, punch)은 아직 zone에 쓰지 않았으니 둘 다 돌려준다. GC가 쓴 뒤 skip된 block은 rtextents만 돌려주고, rt grow는 available에 더한다.
- reserve pool이 덜 차 있으면 돌아오는 몫으로 pool부터 채운다. GC는 pool에서 바로 빼 쓰므로 GC 뒤의 첫 반환은 pool로 간다. GC 실험에서 2400 block을 옮기고도 reserved 두 값(65536, 49152)이 그대로였던 것과 맞는다.
- zoned rt file은 rt 공간이 모자랄 때 inodegc를 서두르는 경로에서 빠진다. 대신 예약 흐름의 1단계에서 flush한다.

그래서 file을 지워도 공간은 바로 돌아오지 않는다. block은 inactivation이 끝나야 user free로, available은 zone reset이 끝나야 돌아온다(code로만 확인). 위 표에서도 지운 직후에는 두 counter가 그대로였고 2초 뒤에 함께 바뀌어 있었다.

그때 available은 212992가 아니라 196864였다. code를 따라 계산하면 이렇다. dd가 쓴 196352 block은 가득 찬 zone 11개(180224 block, reset 11번)와 open zone 하나의 16128 block으로 나뉜다. 가득 차지 않은 open zone에서 free된 block은 zone이 찰 때까지 reclaimable로 넘어가지 않는다. 그래서 available은 free zone 15개(245760 block) + open zone에 남은 256 block - 예약 49152 = 196864다.

code: xfs_zone_space_resv.c, xfs_file.c, xfs_iops.c, xfs_super.c, xfs_mount.c, xfs_bmap.c

## zone을 고르는 순서

zone은 hint가 잘 맞는 open zone, 최근에 쓴 open zone(작은 닫힌 file), 새 zone, 아무 open zone 순으로 고른다. 순서는 code로 확인했다. 단계는 trace에 나오지 않아서, 예의 단계 칸은 code로 해석해 채웠다.

| 순서 | 시도 | 조건 |
|---|---|---|
| 1 | LRU (GOOD) | write hint 점수가 GOOD(2) 이상인 open zone |
| 2 | MRU | `pack_tight`일 때만. 점수 OK(1) 이상 |
| 3 | 새 zone | 사용자 open 수가 사용자 한도(실험에서 3)보다 작고 free zone이 2개 이상일 때 |
| 4 | LRU (OK) | 점수 OK 이상인 open zone |
| 5 | LRU (ANY) | 아무 open zone |

- LRU는 open zone list의 앞(가장 오래 쓰지 않은 것)부터, MRU는 뒤부터 본다. 쓴 zone은 list 끝으로 옮기므로 LRU가 사실상 round-robin처럼 돈다. GC zone과 끝까지 할당한 zone은 고르지 않는다.
- 새 zone으로는 free zone 중 번호가 가장 낮은 것을 연다. 다 실패하면 잠들었다가 zone이 차거나, 새로 열리거나, reset이 끝나거나, log가 shutdown되면 깨어난다.
- `pack_tight`는 file 크기가 zone보다 작고, write로 열려 있지 않고, append-only가 아닐 때 참이다. untar나 작은 file 복사로 생긴 file을 한 zone에 모으기 위해서다(주석). 닫힌 file의 writeback(`sync`, flusher)이 여기에 들고, direct write에서는 늘 거짓이다(code로만 확인).

사용자 open zone 한도가 3인 file system에서 세 가지로 쓰면 다음과 같다.

| file | 쓰는 법 | 단계 | zone | 시작 block |
|---|---|---|---|---|
| d1, d2, d3 | hint 없는 1 MiB를 하나씩 direct로 | 3 | 0, 1, 2 | 0 |
| d4 | 같음 | 4(한도가 차서 맨 앞) | 0 | 256 |
| b1~b4 | 256 KiB를 buffered로 쓰고 닫은 뒤 `sync` | 2(마지막에 쓴 zone) | 0 | 512, 576, 640, 704 |
| c1, c2 | 256 KiB를 buffered로 쓰고 연 채 fsync | 4(앞쪽부터) | 1, 2 | 둘 다 256 |

code: xfs_zone_alloc.c

## write hint

XFS는 file의 write hint와 zone의 hint가 얼마나 맞는지를 점수로 매겨 zone을 고른다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### hint가 오는 곳

- 사용자는 `fcntl(F_SET_RW_HINT)`로 VFS inode의 `i_write_hint`에 `WRITE_LIFE_NONE`, `SHORT`, `MEDIUM`, `LONG`, `EXTREME` 중 하나를 단다. 안 달면 `NOT_SET`으로 남는다.
- XFS는 hint를 disk에 저장하지 않고, XFS 안에서 이 값을 쓰는 곳은 zone 할당 code 한 곳뿐이다. 그래서 inode가 cache에서 빠지면 hint도 사라진다(code로만 확인).
- hint는 bio를 낼 때 읽는다(buffered는 writeback 때, direct는 submit 때). mount option `nolifetime`이면 늘 `NOT_SET`으로 보고, 이 option은 zoned가 아닌 file system에서는 거부된다.
- iomap은 같은 hint를 bio에도 싣는다(`bio->bi_write_hint`). GC의 write bio는 hint를 싣지 않는다(code로만 확인).

### 점수표

점수(`xfs_zoned_hint_score`)는 zone과 file의 hint가 같을 때와, 둘이 LONG과 EXTREME으로 섞일 때만 붙는다. SHORT와 MEDIUM은 같은 hint끼리 GOOD(2), NOT_SET과 NONE은 같은 hint끼리 OK(1), LONG과 EXTREME은 둘을 섞어도 OK다. 빈 칸은 0(ANY)이다.

| zone \ file | NOT_SET | NONE | SHORT | MEDIUM | LONG | EXTREME |
|---|---|---|---|---|---|---|
| NOT_SET | OK |  |  |  |  |  |
| NONE |  | OK |  |  |  |  |
| SHORT |  |  | GOOD |  |  |  |
| MEDIUM |  |  |  | GOOD |  |  |
| LONG |  |  |  |  | OK | OK |
| EXTREME |  |  |  |  | OK | OK |

- SHORT와 MEDIUM file은 1단계에서 같은 hint의 zone에 붙는다. LONG과 EXTREME file은 `pack_tight`이면 2단계에서 붙고, 아니면 새 zone을 열 수 있을 때 연다. 다른 hint끼리는 5단계에서만 섞인다.
- zone의 hint는 처음 연 file의 hint가 된다. hint 없이 열린 zone에 hint 있는 file이 들어오면 그때 hint를 달고 그 뒤로는 바꾸지 않는다. 주석에 따르면 hint별로 쓴 block 수를 따로 세지 않는 한 바꿔도 뜻이 없다. 이 일은 사실상 5단계에서만 생긴다(code로만 확인).
- mount 때 되살린 open zone과 GC zone의 hint는 `NOT_SET`이라, GC로 옮긴 data는 hint별로 나뉘지 않는다(code로만 확인).

### hint별 배치

사용자 open zone 한도가 3인 file system에 short, medium, long hint를 단 1 MiB file을 hint마다 10개씩 쓰면, lifetime(기본)에서는 hint별로 zone이 갈렸다. 1~3번은 zone마다 2560 block(10 MiB), 4번은 zone 0에 7680 block이 쌓였다.

| 실행 | mount | 쓰는 법 | short | medium | long | open zone의 hint |
|---|---|---|---|---|---|---|
| 1 | lifetime | buffered, file 순서 | 0 | 1 | 2 | SHORT, MEDIUM, LONG |
| 2 | lifetime | direct, hint 순서 | 0 | 1 | 2 | SHORT, MEDIUM, LONG |
| 3 | nolifetime | direct, hint 순서 | 0, 1, 2 | 0, 1, 2 | 0, 1, 2 | 모두 NOT_SET |
| 4 | nolifetime | buffered, hint 순서 | 0 | 0 | 0 | zone 0 하나, NOT_SET |

3번은 1단계가 실패해 한도까지 새 zone을 연 뒤 LRU OK로 돌았을 것이고, 4번은 닫힌 작은 file이라 MRU에서 한 zone에 모였을 것이다(둘 다 추정).

code: xfs_zone_alloc.c, xfs_super.c, fs/fcntl.c

## zone GC

zone GC는 바로 쓸 수 있는 block이 모자랄 때만 돌고, 이미 비어 있는 zone은 GC 없이 바로 reset한다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 회수할 것이 생기는 때

block을 free하면(truncate, unlink, 덮어쓰기의 옛 block) XFS는 bitmap 대신 rt rmap inode의 used를 줄이고, open zone이 아니면 reclaimable bucket을 갱신하고, rtextents를 돌려준다. 실제 공간은 zone을 reset해야 돌아오므로 zoned rtgroup은 busy extent를 추적하지 않는다(busy extent는 [해제의 흐름](23-free-flow.md)에 있다).

- 가득 찬 zone은 used 비율에 따라 10% 단위의 reclaimable bucket 10개 중 하나에 들어간다. bucket 번호는 `10 * used / zone 크기`라서 used 800인 zone은 bucket 0에 든다.
- used가 0이 되면 `xfs_zone_emptied` trace를 내고 reset list에 넣는다.
- open zone은 bucket에 넣지 않고, 다 쓰는 순간 그때까지 free된 양을 한꺼번에 넘긴다.

### GC가 도는 조건

reclaimable zone이 있고 다음 중 하나면 GC가 돈다.

- available이 사용자 open zone 한도만큼의 zone 분량보다 적다(한도 3이면 49152 block).
- sysfs `zoned/zonegc_low_space`가 0이 아니고, available이 user free의 그 퍼센트보다 적다. 기본값은 0이고 0에서 100까지 받는다.

XFS는 mount 때 GC thread(`xfs-zone-gc/<dev>`)를 만들어 park해 두고, read-write mount가 끝날 때 unpark한다. freeze와 read-only remount에서 park하고, 되돌릴 때 unpark한다([mount, unmount, freeze](16-mount-freeze.md)).

- thread는 zone이 비었을 때, GC zone이 찼을 때, GC의 bio가 끝났을 때 깨어난다. sysfs 값을 바꿀 때(read-only면 빼고)와 공간을 기다리는 writer가 GC가 돌지 않는 것을 볼 때도 깨어난다. 새 zone을 열 때와 가득 찬 zone에서 block이 처음 빠질 때는 GC가 필요할 때만 깨운다.
- 할 일이 없으면 running 표시를 지우고, 공간을 기다리는 writer를 모두 깨운 뒤 잔다.

### victim과 target

- victim을 고를 때는 bucket 0부터 보고, bucket 안에서는 used가 가장 적은 zone을 고른다. used가 같으면 번호가 낮은 zone을 고른다. GC 중인 zone과 used 0인 zone은 건너뛰고, bucket 0에서 1% 미만이면 바로 고른다.
- 쓰던 GC zone에 자리가 있으면 그 zone을 target으로 쓴다. 다 썼으면 그 zone의 write가 끝날 때까지 새 zone을 열지 않고, 새 zone은 hint 없이 GC 전용으로 연다.
- unclean shutdown 뒤 free zone이 없거나 open 한도가 차 있으면, mount 때 가장 덜 찬 open zone을 GC용으로 빼앗는다.

### 옮기는 방법

GC는 data를 옮기는 동안 file lock을 잡지 않는다. 일단 새 zone에 써 두고, I/O가 끝난 뒤 그 사이 다른 write가 없었는지 확인한다(code로만 확인).

1. victim의 rt rmap을 한 번에 최대 1024개 조회해 inode 번호와 file offset 순으로 정렬한다. 한 file의 조각을 target에 이어 붙이기 위해서다(주석의 "defragment the mappings"). 지워졌거나 일반 rt file이 아닌 inode는 건너뛴다.
2. rmap record 하나를 scratch(1 MiB 둘)의 남은 양까지 잘라 읽는다. 일반 writer의 공간을 빼앗아 deadlock이 나지 않게, 쓸 공간은 reserve pool에서 바로 받는다(주석).
3. 읽은 bio를 write로 바꿔 zone append 한도에 맞게 쪼개 낸다.
4. 완료 때는 IOLOCK과 MMAPLOCK을 잡았다 놓으며 layout을 깨고 진행 중인 direct I/O를 기다린다. 그 뒤 옛 startblock을 넘겨 write 완료와 같은 처리를 한다.
5. data fork의 지금 block이 옛 startblock과 다르면(사용자 write가 그 사이에 덮었으면) remap하지 않고 GC가 쓴 block을 skip한다. 주석에 따르면 exchange-range와 collapse, insert range도 같은 skip을 일으킨다([reflink, COW, dedupe, exchange-range](26-reflink-cow.md)).

### zone reset

GC는 매번 reset list를 먼저 처리한다. rt device의 cache를 flush하고, zone마다 rmap inode의 log를 force한 뒤 `REQ_OP_ZONE_RESET`을 낸다. 이 순서 때문에 zone을 가리키던 rmap 변경이 모두 commit된 뒤에야 zone이 비워진다. conventional zone이면 discard를 내고, discard도 안 되면 bio를 바로 끝낸다. reset이 끝나면 free mark를 다시 달고, available을 zone 하나만큼 돌려주고, 기다리던 writer를 깨운다.

### GC 전후

사용자 open zone 한도가 1인 file system에서 zone 0, 1, 2를 가득 채운 뒤 zone마다 살아 있는 block을 800개(약 5%)만 남기면, 세 zone이 모두 bucket 0에 든다. 여기서 `zonegc_low_space`를 100으로 올리면 GC가 돈다.

| 항목 | GC 전 | GC 뒤 |
|---|---|---|
| zone 0, 1, 2 | FULL, used 800씩 | reset되어 free zone |
| zone 3 | free zone | GC zone, written과 used 2400 |
| free zone 수 | 13 | 15 |
| user available | 163840 | 210592(= 262144 - 49152 - 2400) |
| stats `zoned`(read, write, reset) | 0 0 0 | 9 9 3 |
| stats `gc xpc`(옮긴 byte) | 0 | 9830400(= 3 x 800 x 4096) |

**도식: GC가 zone 셋을 zone 3으로 옮김**

```text
victim: used가 적은 zone부터, 같으면 낮은 번호부터

zone 0  FULL, used 800  ---+
zone 1  FULL, used 800  ---+--->  zone 3  GC zone, used 0 -> 2400
zone 2  FULL, used 800  ---+      (free zone 중 가장 낮은 번호)

zone 0, 1, 2  used 800 -> 0 -> reset -> FREE   (free zone 13 -> 15)
```

- victim마다 `xfs_zone_emptied` 뒤에 `xfs_zone_reset`이 따랐다. 세 file은 모두 zone 3으로 옮겨졌고 내용(0)은 그대로였다.
- read-only로 remount한 상태에서는 같은 값을 쓰고 1초를 기다려도 user available이 212992 그대로였다. read-only에서는 GC thread를 park하기 때문이다. 이 값은 read-write 때보다 49152(reserved available) 큰데, read-only remount가 reserve pool을 counter에 돌려주기 때문일 것이다(추정).

code: xfs_zone_gc.c, xfs_zone_alloc.c, xfs_extfree_item.c, xfs_sysfs.c, xfs_super.c

## 한계점

- conventional zone이 앞에 있는 zoned 장치 하나로 만든 "내부 rt" 배치는 확인하지 못했다. 문서에는 그 zone을 data device로, sequential zone을 rt device로 쓴다고 나오고, code에서는 `sb_rtstart`가 그 경계다.
- zoned direct write가 block 정렬이 아니면 unaligned 경로로 가는데, 그 경로의 zoned 처리는 확인하지 못했다.
- GC의 read와 write가 zone마다 몇 조각으로 나뉘었는지는 보지 않았다(쪼갠 write 조각을 call 수에 따로 세지 않는다는 것은 code로만 확인).
- 지운 뒤 두 counter 가운데 어느 쪽이 먼저 바뀌었는지는 sample 둘로 구분하지 못했다.
- `xfs_freecounter_enospc`의 caller offset 둘이 code의 어느 곳인지는 확인하지 못했다.
- rtsb 자리의 block 내용은 dump하지 않았고, `/rtgroups`의 가운데 항목은 보지 않았다.
- null_blk는 실제 zoned 장치가 아니라서, 장치 쪽 성능과 전원 장애 동작은 이 실험으로 알 수 없다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- rt device는 null_blk configfs로 만든 host-managed zoned device다(1024 MB, zone 64 MB 16개, conventional zone 없음, `memory_backed`, 장치 open zone 한도 `max_open_zones=0`). data device는 1 GiB image의 loop device다.
- mkfs는 `-r rtdev=`만, mount는 `-o rtdev=`로 했고 관찰마다 file system을 새로 만들었다. GC 전후 관찰에는 `max_open_zones=2`를, write hint의 3번과 4번에는 `nolifetime`을 더했다.
- GC 전후 관찰에서는 block 경계를 맞추려고 file을 800 block으로 잘랐다. 경계가 안 맞는 truncate는 마지막 block을 zeroing하며 새로 써서 zone 배치가 달라질 수 있다(code로만 확인).
- write hint의 2~4번은 hint 순서로 썼다. file 순서로 쓰면 nolifetime direct도 file 3개마다 zone 0, 1, 2를 돌아 hint 효과와 구별되지 않는다(code로 계산). 1번은 buffered라, hint가 없을 때의 모양(4번, 모두 zone 0)과 비교해 읽는다.

### 관찰 목록

명령은 줄여 적었고, 켠 event는 명령 칸의 `# event:` 줄에 있다.

| 관찰 | 명령과 켠 event | 출처 | 본문의 예 |
|---|---|---|---|
| zone과 rtgroup | ```bash<br># null_blk configfs p29z<br>echo 1024 > size; echo 4096 > blocksize; echo 1 > memory_backed<br>echo 1 > zoned; echo 64 > zone_size; echo 0 > zone_nr_conv; echo 1 > power<br>truncate -s 1G /dev/shm/p29d.img; losetup -f --show /dev/shm/p29d.img<br>xp71 mkfs.xfs -f -r rtdev=/dev/p29z /dev/loop0<br># event: xfs_zones_mount xfs_zone_opened<br>mount -o rtdev=/dev/p29z /dev/loop0 /mnt/p29<br>dmesg; cat /sys/fs/xfs/loop0/zoned/max_open_zones; xp71 xfs_info /mnt/p29<br>xp71 xfs_io -f -d -c "pwrite -b 1m 0 1m" /mnt/p29/f1<br>xp71 xfs_io -c stat /mnt/p29 /mnt/p29/f1; cat /proc/self/mountstats<br>mkdir /mnt/p29/d; xp71 xfs_io -c "chattr -t" /mnt/p29/d; stat -f /mnt/p29 /mnt/p29/d<br>umount /mnt/p29<br>xp71 xfs_db -r -R /dev/p29z -c "sb 0" -c print /dev/loop0<br>xp71 xfs_db -r -R /dev/p29z -c "ls -m /rtgroups" /dev/loop0<br>xp71 xfs_db -r -R /dev/p29z -c rtsb -c print /dev/loop0<br>xp71 xfs_db -r -R /dev/p29z -c "path -m /rtgroups/0.rmap" -c print /dev/loop0<br>``` | blktests common/null_blk, man xfs_db(8)의 `-R`과 `rtsb` | zoned inode의 xflags, on-disk 표, open zone 수 표, 두 counter 표의 둘째 행 |
| 1 MiB direct write 하나와 zone을 고르는 순서 | ```bash<br># event: xfs_zone_opened xfs_zone_alloc_blocks xfs_zone_record_blocks<br>#        xfs_zone_full xfs_zoned_map_blocks<br>for f in d1 d2 d3 d4; do xp71 xfs_io -f -d -c "pwrite -b 1m 0 1m" /mnt/p29/$f; done<br>for f in b1 b2 b3 b4; do xp71 xfs_io -f -c "pwrite 0 256k" /mnt/p29/$f; done; sync<br>for f in c1 c2; do xp71 xfs_io -f -c "pwrite 0 256k" -c fsync /mnt/p29/$f; done<br>xp71 xfs_bmap -v /mnt/p29/d4; cat /proc/self/mountstats<br>``` | fstests tests/xfs/638 | 1 MiB direct write 하나, zone을 고르는 순서의 표 |
| hint별 배치 | ```bash<br># hint(short medium long)마다 file 10개, 실행마다 mkfs<br># 3번과 4번: mount -o rtdev=/dev/p29z,nolifetime<br>touch $f; /opt/fstests-d98a5b5/src/rw_hint $f short<br>xp71 xfs_io -c "pwrite 0 1m" $f        # 2번과 3번은 xfs_io -d<br>sync; xp71 xfs_bmap -v $f; cat /proc/self/mountstats<br>``` | fstests tests/xfs/638 | hint별 배치 표 |
| GC 전후 | ```bash<br># event: xfs_zones_mount<br>mount -o rtdev=/dev/p29z,max_open_zones=2 /dev/loop0 /mnt/p29<br># event: xfs_zone_opened xfs_zone_full xfs_zone_emptied<br>for i in 1 2 3; do<br>dd if=/dev/zero of=/mnt/p29/data_$i bs=1M oflag=direct count=64<br>truncate -s 3276800 /mnt/p29/data_$i<br>done; sync<br>mount -o remount,ro /mnt/p29<br>echo 100 > /sys/fs/xfs/loop0/zoned/zonegc_low_space; sleep 1<br>echo 0 > /sys/fs/xfs/loop0/zoned/zonegc_low_space; mount -o remount,rw /mnt/p29<br># event: xfs_zone_gc_select_victim xfs_zone_gc_target_opened<br>#        xfs_zone_gc_target_stolen xfs_zone_emptied xfs_zone_reset<br>#        xfs_zone_opened xfs_zone_full<br>echo 100 > /sys/fs/xfs/loop0/zoned/zonegc_low_space<br># mountstats의 "RT GC required: 1"이 풀릴 때까지 1초씩 기다림<br>cmp -n 3276800 /mnt/p29/data_1 /dev/zero<br>``` | fstests tests/xfs/668(truncate 크기만 바꿈) | GC 전후 표 |
| 두 counter의 움직임 | ```bash<br>mkdir /mnt/p29/d; xp71 xfs_io -c "chattr -t" /mnt/p29/d<br>stat -f /mnt/p29 /mnt/p29/d; df -B4096 /mnt/p29<br># event: xfs_freecounter_enospc xfs_freecounter_reserved<br>dd if=/dev/zero of=/mnt/p29/fill bs=1M oflag=direct<br>xp71 xfs_io -f -c "pwrite -b 8m 0 8m" /mnt/p29/g<br>rm /mnt/p29/fill /mnt/p29/g; sleep 2; sync; sleep 1<br>``` | fstests common/xfs의 mountstats 읽기 | 두 counter 표, ENOSPC 예 |

### 출력 읽는 법

- mountstats의 open zone 줄은 "zone N, wp X, written Y, used Z, hint H"이고, GC zone에는 "(GC)"가 붙는다.
- stats의 `zoned` 줄은 GC의 read, write, reset call 수, `gc xpc` 줄은 GC가 옮긴 byte다.
- zone trace의 값은 16진 block 수다. `xfs_db`는 rmap inode의 used를 `v3.cowextsize`로 찍는다.
- rtsb 출력의 `0xcdcdcdcd`는 `xfs_io pwrite`의 기본 채움 값이라 zone 0에 쓴 file의 내용일 것이다(추정).
- `xfs_freecounter_enospc`는 한 요청에 caller offset만 달리해 두 번 나온다. 첫 시도와 inodegc flush 뒤의 재시도일 것이다(추정).
- `/rtgroups`는 앞 6줄과 끝 3줄, `rtsb`는 앞 4줄만 남겼다.

### 원본 출력: zone과 rtgroup

(원본 출력 "mkfs, mount, sysfs": `raw/29-zoned.md`)

(원본 출력 "superblock, /rtgroups, rmap inode": `raw/29-zoned.md`)

(원본 출력 "1 MiB를 쓴 뒤의 xflags, mountstats, statfs": `raw/29-zoned.md`)

### 원본 출력: 1 MiB direct write 하나와 zone을 고르는 순서

(원본 출력 "d1~d4, b1~b4, c1, c2의 zone과 위치": `raw/29-zoned.md`)

### 원본 출력: hint별 배치

(원본 출력 "write hint별 rtgroup": `raw/29-zoned.md`)

### 원본 출력: GC 전후

(원본 출력 "GC 전: 세 zone을 채우고 truncate한 상태": `raw/29-zoned.md`)

(원본 출력 "read-only와 read-write에서 GC 시작": `raw/29-zoned.md`)

(원본 출력 "GC 뒤의 상태": `raw/29-zoned.md`)

### 원본 출력: 두 counter의 움직임

(원본 출력 "mount 직후의 counter와 statfs": `raw/29-zoned.md`)

(원본 출력 "ENOSPC까지 채우기와 greedy 예약": `raw/29-zoned.md`)

(원본 출력 "지운 뒤 counter가 돌아오는 순서": `raw/29-zoned.md`)
