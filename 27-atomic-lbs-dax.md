# atomic write, large block size, DAX

> XFS 전반 wiki, 묶음 "기능". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/27-atomic-lbs-dax.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/27-atomic-lbs-dax.md, 접힌 원본 출력은 raw/27-atomic-lbs-dax.md.

## 개요

- atomic write를 쓰면 쓴 범위가 전부 반영되거나 전혀 반영되지 않아, 일부만 반영되는 torn write가 생기지 않는다. XFS는 길이가 device가 광고하는 단위 안이면 bio 하나로 쓰기를 먼저 시도한다(hardware, code로만 확인). 길이가 더 길거나, 2 block 이상을 새로 할당해야 하거나, mapping이 정렬된 extent 하나가 아니면 COW fork의 새 block에 쓴 뒤 transaction 하나로 remap한다(software, reflink 필요). 1 GiB file system을 기본으로 mount하면 최대 길이가 2 MiB가 된다.
- block이 page보다 크면 page cache의 folio, direct I/O의 정렬 판정, atomic write의 최소 단위가 block 크기를 따른다. 16 KiB block에서는 4 KiB를 buffered로 써도 page cache에 page가 4개 올라왔다.
- DAX file에서는 page cache를 거치지 않고 storage device에 직접 읽고 쓴다. 새 block은 unwritten extent 대신 0으로 채운 written block으로 할당한다(code로만 확인).

예의 값은 option 없이 만든 1 GiB file system(4 KiB block, reflink 켬)과, 같은 크기로 만든 16 KiB block file system에서 얻었다.

## 세 기능의 관계

| 기능 | 들어온 kernel | 다른 기능과 함께 쓸 때 |
|---|---|---|
| hardware atomic write(6.13에는 한 block) | 6.13 | DAX file에서는 쓸 수 없다. `RWF_ATOMIC`을 주면 EOPNOTSUPP가 난다 |
| software atomic write(COW, 여러 block)와 `max_atomic_write` | 6.16 | reflink가 있어야 한다. 없으면 1 block을 넘는 `max_atomic_write`로 mount가 실패한다. DAX file에서는 쓸 수 없다 |
| block size가 page size보다 큰 mount | 6.12 | atomic write와 함께 쓸 수 있고, atomic write의 최소 단위가 block이 된다. DAX는 꺼진다(`dax=always`이면 경고하고 `dax=never`로 내린다) |
| DAX | 확인하지 못함 | block이 page와 같아야 한다. atomic write를 쓸 수 없다 |

code: xfs_inode.h, xfs_iops.c, xfs_mount.c, xfs_super.c

## 호출 순서

write는 아래 표의 위 줄부터 보아 처음 맞는 곳으로 간다(zoned inode는 이 표에서 뺐다). 기본 흐름은 [buffered write](18-buffered-write.md), [direct I/O와 read](19-direct-io-read.md), [iomap](17-iomap.md)에, COW fork와 remap은 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에 있다.

| write | 가는 곳 |
|---|---|
| `RWF_ATOMIC`인데 길이, offset, O_DIRECT 조건이 맞지 않음 | EINVAL이나 EOPNOTSUPP |
| DAX file | `xfs_file_dax_write`. page cache를 거치지 않는다 |
| O_DIRECT, fs block 비정렬 | unaligned 경로 |
| O_DIRECT, `RWF_ATOMIC` | `xfs_file_dio_write_atomic`. hardware 경로를 먼저 시도하고, 안 되면 COW 경로 |
| 그 밖의 O_DIRECT | 일반 direct write |
| 그 밖 | buffered write. block이 page보다 크면 folio도 block 크기 이상이다 |

code: xfs_file.c, fs/read_write.c

## atomic write

iomap 문서에 따르면, atomic write가 metadata를 바꿔야 할 때 file system은 그 갱신도 범위 전체를 한 번에 commit해야 한다. user는 `pwritev2`에 `RWF_ATOMIC`을 주어 atomic write를 요청한다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 한계값

software 경로가 있으면 atomic write의 최대 길이는 group마다 계산한 `awu_max`가 된다. 이 값은 아래 세 값 가운데 가장 작은 값이다. 세 값의 단위는 block이고, 셋 다 2의 거듭제곱이다.

| 값 | 정하는 것 | 기본 mount | `max_atomic_write=64k` |
|---|---|---|---|
| `max_write` | write 한 번의 상한(`MAX_RW_COUNT`) | 262144 | 262144 |
| `max_ioend` | remap transaction 하나가 끝낼 수 있는 block 수 | 512(계산값 814를 내림) | 16 |
| `max_gsize` | group(AG) 크기 | 65536 | 65536 |
| `awu_max` | 가장 작은 값 | 512 block(2 MiB) | 16 block(64 KiB) |

remap 한 번에 intent가 여러 종류(BUI, CUI, RUI, EFI) 생기므로 `max_ioend`는 그 transaction의 log reservation으로 정한다. 기본으로는 truncate transaction의 reservation을 그대로 쓴다.

XFS는 mount 때 block device가 광고하는 atomic write 단위(최소, 최대)를 `bt_awu_min`, `bt_awu_max`에 읽어 둔다. device가 광고하지 않거나 fs block 크기가 그 범위 밖이면 device의 atomic write를 쓰지 않는다. hardware 경로는 `bt_awu_max`가 0보다 클 때 쓸 수 있다. software 경로는 COW fork와 remap을 쓰므로 reflink가 있어야 한다.

statx는 `STATX_WRITE_ATOMIC`을 요청할 때만 아래 값을 채운다. 최소가 0이면 open 때 `FMODE_CAN_ATOMIC_WRITE`가 서지 않아 `RWF_ATOMIC`을 준 write는 VFS에서 EOPNOTSUPP로 실패한다.

| 쓸 수 있는 경로 | 최소 | 최대 | `atomic_write_unit_max_opt` |
|---|---|---|---|
| software(reflink 있음) | fs block | `awu_max`(byte로) | hardware 단위(`bt_awu_max`)와 최대 중 작은 값. 최대가 1 block 이하이면 0 |
| hardware만 | fs block | fs block | 0 |
| 둘 다 없음 | 0 | 0 | 0 |

`max_atomic_write` mount option으로 이 상한을 바꿀 수 있다. 값은 fs block에 정렬된 2의 거듭제곱이어야 한다. `max_write`, group 크기, group의 write 한계도 넘을 수 없다. reflink가 없으면 hardware가 있을 때 1 block까지만 된다. 통과하면 그 크기로 log reservation을 다시 잡고 `awu_max`를 다시 계산한다. log가 그 reservation을 담지 못하면 mount가 실패한다. 기본값과는 비교하지 않아서, log가 허락하면 기본값보다 큰 값도 받는다.

관찰한 file system에서는 mount에 따라 다음과 같았고, `atomic_write_unit_max_opt`는 모두 0이었다.

| mount | 결과 |
|---|---|
| 기본 | statx 최소 4096, 최대 2097152 |
| `max_atomic_write=64k` | 최소 4096, 최대 65536 |
| `max_atomic_write=12k` | mount 실패("not a power of 2") |
| reflink를 끈 file system | 최소와 최대가 0. `pwrite -A`는 "Operation not supported"로 실패 |
| reflink를 끈 file system에 `64k` | mount 실패("no reflink support") |

code: xfs_buf.c, xfs_inode.h, xfs_mount.h, xfs_mount.c, xfs_iops.c, xfs_file.c. 문서: Documentation/admin-guide/xfs.rst

### write 진입 검사

atomic write는 O_DIRECT로만 할 수 있다. 길이는 최소와 최대 사이의 2의 거듭제곱이어야 하고, offset은 그 길이로 정렬되어야 한다.

| 검사 | 어기면 | 관찰한 예(최대 2097152) |
|---|---|---|
| 길이가 최소 이상, 최대 이하 | EINVAL | 4194304 byte(최대의 두 배) |
| 길이가 2의 거듭제곱 | EINVAL | 12288 byte |
| offset이 길이로 정렬됨 | EINVAL | offset 4096에 8192 byte |
| O_DIRECT | EOPNOTSUPP | buffered로 4096 byte |

관찰한 예가 어느 검사에 걸렸는지는 code로만 확인했다. atomic write는 길이와 offset이 모두 fs block의 배수가 되므로 unaligned 경로로 가지 않는다(code에서 낸 추론).

code: xfs_file.c, fs/read_write.c

### hardware 경로와 COW 경로의 선택

XFS는 hardware offload가 더 빠를 것으로 보고 hardware 경로를 먼저 시도한다. 길이가 device의 atomic 최대(`bt_awu_max`)보다 길면 처음부터 COW 경로로 간다. hardware 경로가 `-ENOPROTOOPT`를 돌려주면 lock을 풀고 COW 경로로 다시 쓴다. 이 절의 동작은 code로만 확인했다.

hardware 경로에서는 write 하나를 bio 하나로 보내야 한다. iomap은 `IOMAP_F_ATOMIC_BIO`를 붙인 mapping이 write 전체를 덮으면 bio에 `REQ_ATOMIC`을 세우고, 못 덮으면 EINVAL을 낸다. 그래서 XFS는 `xfs_direct_write_iomap_begin`에서 미리 걸러 `-ENOPROTOOPT`를 돌려준다. 이미 있는 mapping이 통과하려면 시작 disk block이 길이로 자연 정렬되어 있고, mapping 하나가 write 전체를 덮고, 길이가 `bt_awu_max` 이하여야 한다.

reflink가 있는 4 KiB block file system에서 device의 atomic 최대가 16 KiB라면, atomic write는 이렇게 갈린다.

| write | 가는 곳 |
|---|---|
| 64 KiB | 처음부터 COW 경로 |
| 할당된 extent 하나에 16 KiB, 시작 disk block이 16 KiB로 정렬됨 | hardware 경로. `REQ_ATOMIC` bio 하나 |
| hole에 16 KiB(4 block 할당) | `-ENOPROTOOPT` 뒤 COW 경로. 2 block 이상을 할당해야 하면 거절한다. 할당이 모자라면 extent가 여럿이 되기 때문이다 |
| 공유된 extent에 16 KiB | COW mapping이 위 조건을 못 맞추면 `-ENOPROTOOPT` 뒤 COW 경로 |

code: xfs_file.c, xfs_iomap.c, fs/iomap/direct-io.c. 문서: Documentation/filesystems/iomap/operations.rst

### COW 경로

COW 경로에서는 I/O가 끝난 뒤 transaction 하나로 범위 전체를 remap하므로, crash가 나도 범위 전체가 옛 data로 남거나 새 data로 바뀐다(code에서 낸 추정, crash는 시험하지 않음).

**도식: atomic write의 COW 경로에서 바뀌는 두 fork**

```text
f1의 file block 0~15(64 KiB)에 atomic write 한 번.
숫자는 4 KiB block 번호다.

            data fork                COW fork
            -----------------------  -----------------------
write 전    0~15 -> 24~39            (없음)
begin 뒤    0~15 -> 24~39            0~31 -> 2592~2623
            아직 옛 data             0~15만 written, data를 씀
완료 뒤     0~15 -> 2592~2607        16~31 -> 2608~2623
            remap(transaction 하나)  unwritten으로 남음

COW fork는 기본 hint인 32 block 단위로 잡고, remap은 write 범위만
한다. trace에는 write 범위로 자른 0~15 -> 2592~2607만 나오므로,
COW fork 열의 0~31과 16~31은 추정이다.
crash가 나면 recovery 뒤 data fork는 "write 전"이거나 "완료 뒤"다
(code에서 낸 추정).
```

관찰한 file system에서는 device가 atomic 단위를 광고하지 않아 `bt_awu_max`가 0이었다(code로만 확인). 그래서 관찰한 atomic write 셋(64 KiB 둘, 4 KiB 하나)이 모두 처음부터 COW 경로로 갔다. 10 MiB file f1에서는 file block이 이렇게 옮겨졌다.

| atomic write | 옮겨진 file block | 새 disk block |
|---|---|---|
| f1의 offset 0에 64 KiB | 0~15 | 2592~2607 |
| f1의 offset 128 KiB에 4 KiB | 32 | 2624 |
| f1을 reflink한 f2의 offset 0에 64 KiB | f2의 0~15만. f1은 2592~2607 그대로 | 2656~2671 |

offset 128 KiB의 4 KiB write가 2608이 아니라 2624를 받았고 새 block의 시작(2592, 2624, 2656)이 32 block씩 떨어져 있어, COW fork를 32 block 단위로 잡는다는 추정과 맞는다.

- `xfs_atomic_write_cow_iomap_begin`은 reflink가 없으면 EINVAL을 돌려준다. 주석에 따르면 이 경로는 늘 block을 할당하므로 NOWAIT이면 EAGAIN을 돌려준다. COW fork가 없으면 만들므로 reflink하지 않은 file도 COW fork를 쓴다.
- COW fork에 이미 할당된 extent가 있으면 주석과 달리 할당 없이 write 범위로 잘라 쓴다. delalloc extent이면 이때 할당한다. 없으면 cowextsize hint에 맞춰 `tr_write` transaction에서 preallocation으로 잡는다. extent size로 정렬해 잡아 다음 atomic write가 정렬될 가능성을 높인다(주석).
- 잡은 extent가 unwritten이면 written으로 바꾸고 `IOMAP_F_SHARED`로 돌려준다. 그래서 iomap이 `IOMAP_DIO_COW`를 세운다.
- I/O가 끝나면 `IOCB_ATOMIC`인 COW write는 `xfs_reflink_end_atomic_cow`로 간다. 보통의 COW 완료는 extent마다 transaction을 하나씩 쓰지만, 이 함수는 `tr_atomic_ioend` transaction 하나에서 범위 전체를 remap한다. 한계값 절의 `max_ioend`는 이 transaction 하나로 끝낼 수 있는 block 수다.

code: xfs_iomap.c, xfs_reflink.c, xfs_file.c

## page보다 큰 block

kernel 문서에는 이 기능을 다룬 절이 없어서 code와 실험으로만 확인했다. 아래는 4 KiB page인 실험 kernel에서 16 KiB block과 4 KiB block file system에 같은 순서로 실행한 결과다.

| 실행 | 16 KiB block | 4 KiB block |
|---|---|---|
| 64 KiB file의 offset 4 KiB에 4 KiB buffered write 뒤 page cache의 page | offset 0부터 4 page | offset 4096의 1 page |
| 그 file이 차지한 크기(`st_blocks`) | 16 KiB | 4 KiB |
| offset 4 KiB에 4 KiB direct write, 보통 file | direct write | direct write |
| 같은 write, reflink한 file | buffered write로 내려감 | direct write |
| statx의 atomic 최소, 최대 | 16384, 33554432 | 4096, 2097152 |
| 4 KiB atomic write | EINVAL | 성공 |
| block 크기 atomic write | 16384 byte 성공 | 4096 byte 성공 |
| mount 때 잡은 metadata buffer | 16 KiB folio 3개, 512 byte kmalloc 19개 | 16 KiB folio 2개, 4 KiB folio 1개, 512 byte kmalloc 19개 |

### mount 조건

page보다 큰 block을 mount하려면 page cache가 그 크기의 folio를 만들 수 있어야 한다. `CONFIG_TRANSPARENT_HUGEPAGE`를 켜고 build한 kernel은 XFS 상한인 64 KiB까지 mount한다(관찰은 16 KiB, 64 KiB는 code로만 확인). 끈 kernel은 page보다 큰 block을 ENOSYS와 "block size (%u bytes) not supported"로 거절한다(끈 kernel은 code로만 확인). runtime 설정 `transparent_hugepage/enabled`는 이 판정과 상관없다. crc가 없는 V4 file system이면 block이 page보다 클 때 그보다 먼저 ENOSYS로 거절한다.

code: xfs_super.c, include/linux/pagemap.h

### page cache

block이 page보다 크면 XFS는 inode를 만들거나 재사용할 때 mapping에 최소 folio order(`min_folio_order` = `sb_blocklog - PAGE_SHIFT`)를 건다. 16 KiB block에 4 KiB page면 order 2다. 그래서 이 file의 page cache에는 block보다 작은 folio가 생기지 않고, buffered write, writeback, readahead의 단위가 block보다 작아지지 않는다. 표의 첫 줄처럼 4 KiB만 써도 16 KiB folio 하나(4 page)가 올라온다(추정).

code: xfs_ialloc.c, xfs_icache.c

### direct I/O

fs block 비정렬 판정은 block 크기로 하므로, 16 KiB block이면 4 KiB O_DIRECT write도 unaligned 경로로 간다. reflink file은 block 일부만 unshare할 수 없어서, 표처럼 4 KiB direct write도 buffered write로 내려간다. 4 KiB block에서 512 byte write로 같은 fallback을 본 예와 unaligned 경로의 lock은 [direct I/O와 read](19-direct-io-read.md)에 있다.

code: xfs_file.c

### metadata buffer

btree block 같은 block 크기 metadata buffer도 page보다 커진다. XFS는 buffer의 backing memory를 크기에 따라 다르게 잡는다.

| buffer 크기 | backing memory |
|---|---|
| page보다 작은 2의 거듭제곱(예: 512 byte) | kmalloc |
| 그 밖의 page 이하 크기 | page 하나짜리 folio(`__GFP_NOFAIL`). 실패해도 vmalloc으로 가지 않는다 |
| page보다 큰 2의 거듭제곱(예: 16 KiB) | high order folio를 `__GFP_NORETRY`로 한 번 시도하고, 안 되면 vmalloc |
| page보다 크고 2의 거듭제곱이 아님 | vmalloc |

code: xfs_buf.c

## DAX

DAX file을 mmap하면 storage device가 user 공간에 바로 mapping된다. 이 장의 동작은 code로만 확인했다. 실험 kernel은 `CONFIG_DAX`가 꺼져 있어 `CONFIG_FS_DAX`도 없다. 그래서 무엇이 거절되고 무엇이 저장되는지만 보았다.

### 켜지는 조건

inode마다 아래 표의 위 줄부터 보아 처음 맞는 줄을 따른다.

| 조건 | DAX |
|---|---|
| kernel에 `CONFIG_FS_DAX`가 없음 | 끔 |
| `dax=never` | 끔 |
| regular file이 아님, block이 page와 다름, daxdev 없음 | 끔 |
| `dax=always` | 켬 |
| option 없음, `dax=inode` | on-disk flag `XFS_DIFLAG2_DAX`를 따름 |

- `CONFIG_FS_DAX`가 없으면 mount option 처리 code가 그 `#ifdef` 안에만 있어서, `dax=never`까지 모든 `dax` mount option이 EINVAL("unknown mount option")로 실패한다.
- `dax=always`로 mount할 때 daxdev가 없거나 block이 page와 다르면 "Turning off DAX." 경고를 남기고 `dax=never`로 내린다. reflink가 있고 partition이면 EINVAL로 실패한다.
- user가 `FS_XFLAG_DAX`(`chattr +x`)를 세우면 on-disk inode에는 `XFS_DIFLAG2_DAX`로 남는다. directory에 세운 flag는 새로 만드는 child가 물려받는다. DAX를 쓸 수 없는 kernel이나 device에서도 XFS는 이 flag를 거절하지 않고 저장한다. flag를 받는 `xfs_ioctl_setattr_xflags`에 그런 검사가 없다.
- `S_DAX`는 inode를 초기화할 때만 세운다. 그래서 flag를 바꾸면 dentry를 dontcache로 표시해 다음 inode load 때 반영되게 한다.

실험 kernel에서는 `dax` mount option 넷이 모두 실패했다(실패 이유는 출력에 없다). `chattr +x`를 하면 on-disk flag만 남았다(statx의 DAX bit는 꺼진 채). chattr한 directory 아래에 새로 만든 file도 flag를 물려받았다.

code: xfs_super.c, xfs_iops.c, xfs_ioctl.c. 문서: Documentation/filesystems/dax.rst

### read와 write

DAX file에서는 unwritten extent를 쓰지 않는다. 새 block은 할당 transaction을 commit하기 전에 0으로 채운다. commit 뒤 crash하면 0으로 채우지 않은 block이 recovery 뒤 드러나기 때문이다. 이미 있는 unwritten extent도 할당이 필요한 것으로 본다.

|  | DAX가 아닌 경로(writeback, direct I/O) | DAX |
|---|---|---|
| 새 block | unwritten으로 할당 | 0으로 채운 written block(`XFS_BMAPI_ZERO`) |
| I/O가 끝난 뒤 | written으로 변환 | 변환할 것이 없다 |

- read와 write 모두 `IS_DAX`를 direct I/O보다 먼저 보므로, O_DIRECT를 줘도 DAX file이면 `dax_iomap_rw`로 간다. IOLOCK은 read에서 shared로, write에서 exclusive로 잡는다. write가 EOF를 넘으면 그 자리에서 `xfs_setfilesize`를 부른다. page cache를 거친 data writeback이 없으니 ioend도 없다.
- write의 iomap begin으로는 direct write와 같은 `xfs_direct_write_iomap_begin`을 쓴다. mapping을 iomap으로 바꿀 때 `IOMAP_DAX`이면 read와 write 모두 주소에 partition offset을 더하고 `bdev` 대신 `dax_dev`를 채운다.
- reflink한 DAX file이면 write 끝에 `xfs_reflink_end_cow`를 부른다(0 byte이면 cancel). remap 준비, unshare, zeroing, truncate도 `dax_remap_file_range_prep` 같은 DAX 함수로 간다.
- address space ops는 `xfs_dax_aops`로 바뀐다. `writepages`는 mapping을 persistent domain으로 flush한다. `dirty_folio`는 `noop_dirty_folio`라서 folio의 dirty bit만 세우고 mapping에 dirty 표시를 남기지 않는다.

code: xfs_file.c, xfs_iomap.c, xfs_aops.c, xfs_reflink.c, fs/dax.c

### mmap

- fault는 `IS_DAX`이면 `filemap_fault` 대신 `dax_iomap_fault`로 간다. `VM_FAULT_NEEDDSYNC`이면 `dax_finish_sync_fault`를 부른다.
- PMD 크기 fault는 DAX에서만 처리하고, 아니면 `VM_FAULT_FALLBACK`을 돌려준다.
- DAX file이 아니면 `MAP_SYNC`는 EOPNOTSUPP로 실패한다. `CONFIG_DAX`가 없는 kernel의 stub도 `MAP_SYNC`를 거절한다. 실험 kernel에서 `mmap -S`는 "Operation not supported"로 실패했다.
- truncate나 punch hole은 user mapping을 걷고, DAX page를 잡은 사용자(예: DMA 중인 pin)가 놓을 때까지 기다린다. 이 일은 `xfs_break_dax_layouts`가 한다.

code: xfs_file.c, xfs_inode.c, include/linux/dax.h

### media failure

device가 media failure를 알리면 XFS는 rmapbt로 그 block의 주인을 찾는다. `xfs_dax_notify_failure`가 log device와 data 또는 rt device를 나눠 받는다. data나 rt device이면 범위를 fs block으로 바꿔 rmapbt를 query한 뒤 주인마다 처리한다. 주인이 metadata, attr fork, bmbt block이면 shutdown 대상이고, 메모리에 있는 DAX file이면 그 mapping을 쓰는 process를 `mf_dax_kill_procs`로 죽인다. 이 code는 `CONFIG_FS_DAX`일 때만 build된다. rmap 구조는 [rmap과 refcount 구조](10-rmap-refcount.md)에 있다.

code: xfs_notify_failure.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| mkfs.xfs(8)(xfsprogs 7.1.1) | page size 이하 block만 mount할 수 있다 | THP를 켜고 build한 kernel은 64 KiB까지 mount한다. 실험 kernel에서 16 KiB block이 mount되었다. 낡은 문장일 것이다(추정) |

## 한계점

다음은 code로만 확인했다.

- hardware atomic write(`REQ_ATOMIC` bio)와 `-ENOPROTOOPT` 뒤의 재시도. atomic write를 광고하는 device가 없었다.
- DAX의 read, write, mmap, media failure 경로. 실험 kernel에 `CONFIG_DAX`가 없다.
- THP 없이 build한 kernel이 page보다 큰 block의 mount를 거절하는 것.

다음은 확인하지 못했다.

- atomic write의 crash 뒤 상태. remap을 transaction 하나로 commit한다는 데서 끌어낸 추정이고, crash를 일으켜 보지 않았다.
- 16 KiB block에서 statx 최대 33554432(2048 block)를 세 값 가운데 어느 것이 정했는지. 이 관찰에서는 `xfs_calc_atomic_write_unit_max`를 켜지 않았다.
- DAX mount option이 실패한 이유. dmesg를 거른 pattern으로는 option parse 때의 message를 잡지 못했다.
- `mincore`가 large folio의 page를 하나하나 resident로 표시하는지. 16 KiB block에서 4 page를 folio 하나로 본 해석은 이 가정에 기댄다.
- 4 KiB block file system이 mount 때 잡은 16 KiB folio buffer 2개와, 512 byte buffer 19개가 각각 어느 buffer인지.
- page보다 큰 folio를 쓰는 iomap buffered 경로. 이 page에서는 다시 읽지 않았다.
- DAX가 XFS에 들어온 시점, rmapbt가 없는 file system의 DAX media failure 처리.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB image를 새로 만들어 loop device로 쓰고, 끝나면 지웠다. 기본은 option 없는 mkfs(4 KiB block, reflink 켬)이고, 변형은 `-m reflink=0`과 `-b size=16384`다.
- I/O는 동기로 했다. O_DIRECT는 `xfs_io -d`, atomic write는 `pwrite -A`로 했고, `-b`를 길이와 같게 주어 atomic write 하나를 write 한 번으로 썼다.
- loop device는 atomic write를 광고하지 않았고(queue limit과 device statx의 atomic 값이 모두 0), 실험 kernel에는 `/sys/bus/dax`가 없다. 그래서 hardware atomic 경로와 DAX I/O 경로는 발동하지 않았다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| atomic write 한계값 | bdev와 file의 statx(`STATX_WRITE_ATOMIC`). 한계값 절 표의 mount 다섯. `max_atomic_write`로 받아들여진 값은 기본보다 작은 64k만 시험했다 | `xfs_calc_atomic_write_unit_max`, `xfs_calc_max_atomic_write_fsblocks`, `xfs_calc_max_atomic_write_log_geometry` | 한계값의 표 |
| software atomic write의 COW 경로 | 10 MiB file f1을 dd로 쓰고 fsync, 보통 direct write 4 KiB 둘 뒤, f1과 그것을 reflink한 f2에 COW 경로 표의 atomic write 셋, 이어서 거절되는 네 경우 | `xfs_iomap_atomic_write_cow`, `xfs_iomap_found`, `xfs_reflink_end_cow` 등 8개 | COW 경로의 도식과 표, write 진입 검사의 표 |
| 16 KiB block과 4 KiB block 비교 | 같은 image에 block 크기를 바꿔 만들고 같은 순서로 실행. metadata buffer는 mount 직전에 trace를 비우고 `xfs_info` 뒤에 읽은 구간에서 셈 | `xfs_buf_backing_*` 넷, `xfs_file_direct_write`, `xfs_file_buffered_write`, `xfs_reflink_bounce_dio_write` | page보다 큰 block의 표 |
| DAX를 못 켜는 kernel | `dax` mount option 넷, directory d와 file g에 `chattr +x`(d/f는 그 뒤에 만듦), statx, `mmap -S`, umount 뒤 `xfs_db`의 `v3.dax` | 없음 | DAX의 켜지는 조건과 mmap |

### 출력 읽는 법

event 정의는 xfs_trace.h에 있다.

- `xfs_calc_atomic_write_unit_max`는 mount 때 group 종류(`ag`, `rtg`)마다 한 줄 나오고 값은 block 수다. `xfs_calc_max_atomic_write_fsblocks`의 `blockcount`는 transaction 하나가 끝낼 수 있는 block 수로, `logres`로 계산한다.
- COW 경로의 trace에는 write마다 `xfs_file_direct_write`, `xfs_iomap_atomic_write_cow`, COW fork의 `xfs_iomap_found`, `xfs_reflink_end_cow`가 한 줄씩 이 순서로 나왔다. `pos`와 `bytecount`에는 요청한 값이 찍힌다. `xfs_file_direct_write`는 `iomap_dio_rw`를 부르기 직전에 찍히므로(bounce된 write에는 없다), hardware 경로를 먼저 시도했다면 write 한 번에 두 줄이 나와야 한다.
- `xfs_iomap_found`(`fork cow`)의 `startblock`과 `fsbcount`에는 write 범위로 자른 mapping이 찍힌다. 그래서 이 write가 쓴 새 block은 알 수 있지만, hint에 맞춰 더 잡은 block은 나오지 않는다. `xfs_reflink_end_atomic_cow`도 `xfs_reflink_end_cow` event를 쓰므로, trace만으로는 일반 COW 완료와 구분되지 않는다.
- `xfs_buf_backing_*`의 `bbcount`는 512 byte sector 수다(`0x20`이 16 KiB, `0x8`이 4 KiB).
- block 크기 비교의 trace 앞쪽에는 `xfs_file_buffered_write`가 세 줄 있다. 첫 줄은 `mincore`를 본 4 KiB write이고, 다음 두 줄은 direct write에 쓸 file 둘을 채운 64 KiB write다. mount 뒤 dmesg에서는 마지막 XFS 두 줄만 찍었으므로, 그 앞에 다른 줄이 있었는지는 알 수 없다.
- DAX 관찰의 dmesg("XFS (loop"가 든 마지막 여덟 줄)에는 이 mount 시도의 줄이 없다. 모두 앞 관찰이나 더 이른 mount의 줄이다(UUID나 시각이 같다). option을 parse할 때는 superblock이 아직 없어 message가 "XFS:"로 시작하므로 이 grep에 잡히지 않는다(code로만 확인).
- statx의 DAX bit(`0x200000`)는 `attributes_mask`(`0x203070`)에는 있고 `attributes`에는 없었다. umount 뒤 `xfs_db`의 `v3.dax`는 셋 다 1이었다.

### 원본 출력: atomic write 한계값

(원본 출력 "관찰: atomic write 한계값과 max_atomic_write mount option": `raw/27-atomic-lbs-dax.md`)

### 원본 출력: software atomic write의 COW 경로

(원본 출력 "관찰: software atomic write의 bmap 이동과 거절": `raw/27-atomic-lbs-dax.md`)

(원본 출력 "관찰: software atomic write의 trace 순서": `raw/27-atomic-lbs-dax.md`)

### 원본 출력: 16 KiB block과 4 KiB block 비교

(원본 출력 "관찰: 16 KiB block의 mount, page cache, direct I/O, atomic write": `raw/27-atomic-lbs-dax.md`)

(원본 출력 "관찰: 4 KiB block의 mount, page cache, direct I/O, atomic write": `raw/27-atomic-lbs-dax.md`)

### 원본 출력: DAX를 못 켜는 kernel

(원본 출력 "관찰: DAX를 못 켜는 kernel의 mount option, chattr, MAP_SYNC": `raw/27-atomic-lbs-dax.md`)
