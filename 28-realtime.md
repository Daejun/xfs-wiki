# realtime, rtgroup, metadir

> XFS 전반 wiki, 묶음 "기능". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/28-realtime.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/28-realtime.md, 접힌 원본 출력은 raw/28-realtime.md.

## 개요

- realtime device(rt device)는 data device와 따로 붙이는 block device로, file의 data만 두고 inode와 btree 같은 metadata는 data device에 남긴다. free space는 bnobt와 cntbt 대신 고정 크기 file인 rtbitmap과 rtsummary로 관리한다.
- metadir(metadata directory)을 켜면 rtgroup도 함께 켜지고, rt device는 rtgroup 여러 개로 나뉜다. AG처럼 group마다 free space file(zoned가 아닐 때), lock, busy extent를 따로 둔다. rmapbt를 켜면 rt rmapbt가, reflink를 켜면 rt refcountbt가 group마다 생긴다.
- metadir은 이 metadata file을 사용자에게 보이지 않는 directory tree(`/rtgroups`, `/quota`)에 담는다.

예의 값은 1 GiB data device와 1 GiB rt device(4 KiB block, rextsize 1 block)로 만든 두 file system에서 얻었다. fs A에는 metadir과 rtgroup 4개가 있고, fs B에는 둘 다 없다.

## data device와의 비교

data device 쪽은 [free space 구조와 할당 정책](09-free-space.md)과 [할당 한 번의 흐름](22-allocation-flow.md)에 있다.

|  | data device (AG) | realtime device |
|---|---|---|
| free space index | bnobt와 cntbt (B+tree) | rtbitmap과 rtsummary (고정 크기 file) |
| 크기로 찾기 | cntbt lookup | rtsummary의 level 줄을 bitmap block 순서로 훑음 |
| 위치로 찾기 | bnobt lookup | rtbitmap을 앞뒤로 훑음 (`xfs_rtfind_back`, `xfs_rtfind_forw`) |
| metadata 위치 | 같은 AG 안 | 전부 data device |
| 할당 단위 | block | rtx (`sb_rextsize` block) |
| free 수 counter | AGF와 superblock | superblock만 (`sb_frextents`) |
| busy extent, EFI(extent free intent) | 있음 | rtgroup이 있을 때만 |

## realtime device의 공간 관리

이 page의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### rt device로 가는 것

rt device에는 realtime inode의 user data만 간다. `xfs_bmapi_allocate`는 user data이고 inode가 realtime일 때만 `xfs_bmap_rtalloc`을 부르므로, realtime inode라도 bmbt block과 attr fork block은 data device의 AG에서 나온다(code로만 확인).

inode가 realtime인지는 `XFS_DIFLAG_REALTIME` flag로 정해진다. directory에 rtinherit flag(`xfs_io`의 `chattr +t`)를 걸면 그 아래 새 file이 이 flag를 물려받는다. 실험에서도 directory에는 `t`가, 새 file에는 `r`이 붙었다.

rt device를 관리하는 file(rtbitmap, rtsummary, rt rmapbt, rt refcountbt)도 data device에 놓인다. rtbitmap과 rtsummary의 block은 일반 file처럼, rt rmapbt와 rt refcountbt의 block은 btree block으로 AG에서 할당한다.

code: xfs_bmap.c, xfs_rtalloc.c, xfs_rtbitmap.c, xfs_btree.c

### rt extent

rt device에서는 rt extent(rtx)가 할당 단위가 된다. rtx 하나의 크기는 superblock의 `sb_rextsize`(block 수)로 정해지고, rtx 수와 free rtx 수는 `sb_rextents`와 `sb_frextents`에 있다.

- block 번호를 rtx 번호로 바꿀 때 rextsize가 2의 거듭제곱이 아니면 shift 대신 나눈다. 그래서 rextsize는 2의 거듭제곱이 아니어도 된다(code로만 확인).
- rextsize가 1 block보다 크면 rt file은 rtx 단위로만 할당하고 해제한다. 해제할 범위가 rtx 경계에 맞지 않으면 `xfs_rtfree_blocks`가 -EIO를 돌려준다.
- xfsprogs 7.1.1의 mkfs는 `-r extsize=`를 주지 않으면, stripe가 없는 rt device에서 rextsize를 1 block으로 정한다(block이 4 KiB 이상일 때). 실험의 mkfs도 1 block(`extsz=4096`)을 골랐다.

code: xfs_format.h, xfs_rtbitmap.h, xfs_rtbitmap.c

### rtbitmap

rtbitmap에는 rtx 하나마다 bit 하나가 있고, 1은 free, 0은 할당을 뜻한다. word 안에서는 bit 0이 그 word의 가장 낮은 rtx가 된다.

rtgroup이 있으면 word가 host-endian 대신 big-endian 32-bit가 되고, rtbitmap과 rtsummary의 block마다 앞에 header(magic `BMPZ`나 `SUMY`, crc, owner, blkno, lsn, uuid)가 붙는다. header 크기 48 byte는 field 크기 4+4+8+8+8+16의 합으로 셌다(code로만 확인). 그만큼 bitmap block 하나가 다루는 rtx 수가 줄어든다.

|  | fs A (rtgroup 있음) | fs B (rtgroup 없음) |
|---|---|---|
| bitmap block 하나의 rtx | (4096-48)*8 = 32384개 | 4096*8 = 32768개 |
| bitmap 하나가 덮는 rtx | group 하나, 65536개 | rt device 전체, 262144개 |
| bitmap block 수 (관찰한 `rbmblocks`) | 3 (rtx 0~32383, 32384~64767, 64768~65535) | 8 |

fs A의 group 0에서는 rtx 0을 rtsb(rt device의 superblock)가, rtx 1~256을 1 MiB file이 쓴다. 그래서 첫 bitmap block의 word 0~7이 0, word 8이 `0xfffffffe`(bit 0만 0)였고 나머지는 모두 `0xffffffff`였다. 다른 group에서는 file이 rtx 0~255를 써서 word 0~7만 0이었다.

code: xfs_rtbitmap.c, xfs_rtbitmap.h, xfs_format.h

### rtsummary

rtsummary에는 free extent의 개수가 2차원 배열로 들어 있다. 칸 번호는 l x `sb_rbmblocks` + b로 정해지고(`xfs_rtsumoffs`), l은 길이의 level(log2 길이의 내림), b는 그 extent가 시작하는 bitmap block 번호다. 칸의 값은 길이가 [2^l, 2^(l+1)) rtx인 free extent의 개수다. level 수는 log2(rtx 수) + 1이고 칸 하나는 32-bit다. rtgroup이 있으면 summary도 group 크기(`sb_rgextents`)를 기준으로 한다.

위 fs A group 0의 free extent 하나를 따라가면 다음과 같다.

| 단계 | 값 |
|---|---|
| 할당된 rtx | 0(rtsb), 1~256(1 MiB file) |
| free extent | rtx 257~65535, 65279개 |
| level | 2^15 <= 65279 < 2^16이라 15 |
| 시작한 bitmap block | 0 |
| summary 칸 | 15 x 3 + 0 = 45, 값 1 |

group 1~3도 free extent가 rtx 256~65535(65280개)라 같은 칸 45만 1이었다. fs B에서는 칸 136(= 17 x 8 + 0)만 1이었고, 남은 free 261120 rtx(262144 - 1024)가 [2^17, 2^18) 안에 있으니 계산과 맞는다.

할당 한 번에 summary를 최대 세 번 고친다. 고른 free extent의 칸에서 1을 빼고, 앞과 뒤에 남는 부분의 칸에 1씩 더한 뒤 bitmap bit를 0으로 바꾼다(`xfs_rtallocate_range`). 위 상태에서 rtx 1000부터 16개를 할당한다면 다음과 같다.

| 부분 | rtx | 길이 | level | 칸 | 변화 |
|---|---|---|---|---|---|
| 고른 free extent | 257~65535 | 65279 | 15 | 45 | -1 |
| 앞에 남는 부분 | 257~999 | 743 | 9 | 27 | +1 |
| 뒤에 남는 부분 | 1016~65535 | 64520 | 15 | 45 | +1 |

free할 때는 거꾸로 bitmap bit를 1로 바꾸고(`xfs_rtfree_range`) 앞뒤의 free extent와 합친 길이로 summary를 고친다.

code: xfs_rtbitmap.h, xfs_rtbitmap.c, xfs_rtalloc.c

### 할당이 시작하는 곳

rt 할당은 시작 위치 근처를 찾는 near 할당과, 크기로 낮은 주소부터 찾는 size 할당 둘로 한다.

- 시작 위치는 `xfs_rtallocate_rtg`가 정한다. 이웃 extent의 끝(hint)이 있으면 그 rtx가 시작 위치가 된다.
- rtgroup이 없는 file system에서 file의 offset 0에 처음 할당할 때는 `xfs_rtpick_extent`가 고른다. bitmap inode의 atime 칸을 file 생성 순번으로 써서 rt 영역의 0, 1/2, 1/4, 3/4, 1/8, ... 위치를 차례로 고르고, 순번을 1 올린다.
- rtgroup이 있으면 rtpick을 쓰지 않는다. hint가 있으면 hint가 든 group에서, 없으면 group rotor(`m_rtgrotor`)가 돌린 group에서 시작하고, ENOSPC면 다음 group으로 간다.
- 시작 위치가 0이 아니면 near 할당(`xfs_rtallocate_extent_near`)을 부른다. near 할당은 그 rtx에 exact 할당을 먼저 해 보고, 안 되면 summary를 보며 bitmap block을 양쪽으로 넓혀 간다. ENOSPC면 error를 0으로 돌린다.

size 할당은 `if (!error)` 아래에 있어서, near 할당이 성공해도 이어서 실행되어 그 결과를 덮어쓴다(code로만 확인). bitmap은 그 뒤에야 바뀌므로 손상은 없지만, rtpick이 고른 위치와 이웃 hint가 group 안의 실제 위치에 반영되지 않을 수 있다.

size 할당(`xfs_rtallocate_extent_size`)은 필요한 길이의 level부터 위로 올라가며, level마다 bitmap block 0부터 summary를 본다. 그래서 같은 조건이면 낮은 주소부터 채운다. 빈 rt directory에 1 MiB file 4개를 차례로 쓰면 다음과 같이 놓였다.

| file | fs B의 시작 rtx | fs A의 위치 |
|---|---|---|
| f0 | 0 | group 0, rtx 1 |
| f1 | 256 | group 1, rtx 0 |
| f2 | 512 | group 2, rtx 0 |
| f3 | 768 | group 3, rtx 0 |

fs B에서는 file 4개가 rtx 0부터 빈틈없이 놓였고, 실험 뒤 bitmap inode의 atime은 1970-01-01 00:00:04(순번 4)였다. 순번은 올라갔지만 위치에는 반영되지 않았다. near 할당의 결과가 쓰였다면 fs B의 f1~f3은 rtx 131072(1/2), 65536(1/4), 196608(3/4) 근처에 놓였을 것이다(rtpick 위치는 code로 계산). fs A에서는 group rotor가 시작 group을 돌려 file이 group마다 하나씩 놓였다.

code: xfs_rtalloc.c, xfs_bmap.c

## rtgroup이 바꾼 것

### 켜는 조건

rtgroup에는 별도 feature bit가 없고, metadir을 켜면 함께 켜진다(`xfs_has_rtgroups`가 `xfs_has_metadir`를 그대로 돌려준다). rt device가 있으면 rtsb도 생긴다(zoned 제외).

- rt rmapbt와 rt reflink는 rtgroup이 있어야 쓸 수 있다. rt reflink는 rextsize가 1 block일 때만 되고, 어긋나면 mount가 거부된다.
- rtgroup이 있는 file system은 mkfs에 `-m metadir=1 -r rtdev=<dev>,rgcount=N`을 주어 만든다. xfsprogs 7.1.1의 mkfs는 metadir 없이 rtdev만 주면 rgcount와 rgsize를 0으로 두고, 따로 켜라고 하지 않은 reflink와 rmapbt를 끈다. 이 두 flag는 rt device만이 아니라 file system 전체의 feature다. fs B의 mkfs 출력도 `rmapbt=0`, `reflink=0`, `metadir=0`, `rgcount=0`이었다.

code: xfs_mount.h, xfs_reflink.c, xfs_super.c

### AG와 같은 group 틀

AG와 rtgroup은 같은 group 구조를 바탕으로 한다. `struct xfs_rtgroup`의 첫 member가 AG와 공통인 `struct xfs_group`이라서, group 번호, 크기, busy extent tree, health bit, intent drain이 rtgroup에도 AG와 같은 모양으로 들어 있다.

**도식: rtgroup 배치와 metadir tree**

```text
fs A: rt device 262144 block, rgcount 4, rgextents 65536, rextsize 1

rt block  0           65536       131072      196608      262144
          +-----------+-----------+-----------+-----------+
          | rtgroup 0 | rtgroup 1 | rtgroup 2 | rtgroup 3 |
          |rtsb  data |   data    |   data    |   data    |
          +-----------+-----------+-----------+-----------+
          rtsb는 rtx 0 하나이고 group 0에만 있다

data device: rt device의 metadata는 모두 여기에 있다
  superblock  metadirino 129, rgcount 4, rbmino 0, rsumino 0
  metadir tree                inode
    /                         129
    +-- rtgroups/             1179776
          +-- 0.bitmap        1179777   group 0의 free bit, block 3개
          +-- 0.summary       1179778   group 0의 level별 free extent 수
          +-- 0.rmap          1179779   rt rmapbt, root는 inode 안
          +-- 0.refcount      1179780   rt refcountbt, root는 inode 안
          +-- 1.bitmap        1179781
              ...
          +-- 3.refcount      1179792

fs B(metadir 없음): tree 없이 superblock의 rbmino 129, rsumino 130
```

- rtgroup 하나에는 rt device를 관리하는 metafile inode가 최대 넷(bitmap, summary, rmap, refcount) 딸린다. rmap은 rmapbt를, refcount는 reflink를 켰을 때만 있다.
- superblock에는 `sb_rgcount`, `sb_rgextents`(group 크기, rtx 단위), `sb_rgblklog`가 있다. 마지막 group은 나머지 크기가 되고, group 하나는 최대 2^31-1 block이다.

code: xfs_rtgroup.h, xfs_group.h, xfs_rtgroup.c, xfs_format.h

### sparse한 rt block 번호

rtgroup이 있으면 rt block 번호가 AG의 fsbno처럼 sparse해진다. 번호는 `rgno << sb_rgblklog | rgbno`이고, `sb_rgblklog`는 group 크기를 2의 거듭제곱으로 올린 log라서 group 끝과 다음 group 시작 사이에 번호 구멍이 생긴다. disk 주소로 바꿀 때는 `xfs_rtb_to_daddr`가 `rgno * g->blocks + (rtbno & g->blkmask)`로 구멍을 접는다.

rextsize 1 block에서 group 크기가 50000 block이라면 rgblklog는 16이 된다. 그러면 group 1의 첫 block은 번호가 65536, disk 위치가 block 50000이고, 번호 50000~65535는 쓰이지 않는다. 실험한 fs A는 group 크기가 2^16이라 구멍이 없었고, group 1의 첫 file은 rt block 65536에서 시작했다.

zone capacity가 zone size보다 작은 zoned device(`ZONE_GAPS`)에서는 device 쪽에 구멍이 있어서 번호를 그대로 쓴다. rtgroup이 없으면 rt device 전체가 group 하나가 되고, rt block 번호는 device 안의 선형 번호가 된다.

code: xfs_rtgroup.h, xfs_mount.h, xfs_sb.c

### rtsb

rtsb는 rt device의 첫 block에 하나만 있고, rtgroup 0의 rtx 0을 차지한다. rtsb에는 magic(`0x46726F67`, 'Frog'), crc, fname, uuid, meta_uuid만 들어 있고, rt geometry는 primary superblock에 둔다.

- group 0의 사용 가능한 첫 block이 rextsize만큼 뒤로 밀리므로, group의 최소 크기는 2 rtx가 된다.
- label을 바꾸면 primary superblock의 fname, uuid, meta_uuid를 rtsb로 복사해 log한다(code로만 확인).
- fs A의 rtsb는 crc가 맞았고 uuid가 primary superblock과 같았다. 빈 file system의 free rtx는 rtsb 몫 하나가 빠진 262143이었다.

code: xfs_format.h, xfs_rtgroup.c, xfs_sb.c

### group 단위 lock

rtgroup이 없으면 bitmap과 summary inode 한 쌍의 lock이 rt device 전체를 막는다. rtgroup이 있으면 group마다 그 쌍이 있어서 group끼리 동시에 할당할 수 있다(code로만 확인).

`xfs_rtallocate_rtg`의 주석에 따르면 rtgroup file system에서는 그 group에서 할당하기로 정해질 때까지 inode를 transaction에 join하지 않아서, type마다 inode 하나씩만 lock된다. 실패한 group은 unlock하고 다음 group으로 간다. rtgroup이 없으면 rtpick이 bitmap inode를 고치므로 처음부터 join한다. rtgroup이 있으면 할당할 때 busy extent도 피한다(tracepoint `xfs_rtalloc_extent_busy`, `xfs_rtalloc_extent_busy_trim`).

code: xfs_rtalloc.c, xfs_rtgroup.c

### free 경로

rtgroup이 있으면 data device처럼 rt extent의 해제를 EFI로 미루고, 해제한 extent를 busy extent에 넣는다. rtgroup이 없으면 같은 transaction에서 bitmap과 summary를 바로 고친다. code가 가는 길과, 1 MiB rt file 하나를 지우고 sync했을 때의 trace는 다음과 같다.

|  | fs A (rtgroup 있음) | fs B (rtgroup 없음) |
|---|---|---|
| `xfs_bmap_del_extent_real` 다음 (code) | `xfs_free_extent_later` -> `xfs_defer_extent_free` | `xfs_bmap_free_rtblocks` |
| intent | `rtrmap` 하나, `rtextent_free` 하나 | 없음 |
| bitmap과 summary를 고치는 때 (code) | intent를 마무리할 때 (`xfs_rtextent_free_finish_item`) | unmap한 transaction 안 |
| busy extent | group 0, block 1부터 256개 | 없음 |

- 함수 안의 주석에 따르면 옛날에는 rt extent를 EFI로 해제하지 않았다. 그런데 rmap이 켜지면 data device와 같은 순서(file mapping 제거, reverse mapping 제거, block 해제)를 지켜야 하고 rt reflink도 그렇다. 두 기능이 모두 rtgroup에 기대므로 rt EFI를 rtgroup 유무로 나눈다.
- rmap과 refcount의 변경도 rt 전용 intent(`rtrmap`, `rtrefcount`)로 간다.
- rtgroup이 없으면 busy extent에 넣지 않으므로, 해제한 rt 공간을 그 transaction이 log에 commit되기 전에 다시 할당할 수 있다(code로만 확인). rt file이 모두 비면 `xfs_rtfree_extent`가 rtpick 순번을 0으로 되돌린다.

intent와 deferred operation의 일반 구조는 [deferred operation, intent, log recovery](13-intent-recovery.md)에, busy extent는 [해제의 흐름](23-free-flow.md)에 있다.

code: xfs_bmap.c, xfs_alloc.c, xfs_extfree_item.c, xfs_rmap_item.c, xfs_refcount_item.c, xfs_rtbitmap.c

### rt rmapbt와 rt refcountbt

두 btree는 root를 metafile inode의 data fork 안에 둔다(inode-rooted btree). fork format은 `XFS_DINODE_FMT_META_BTREE`, block magic은 `MAPR`과 `RCNT`이고, record 형식은 data device와 같다. rt rmapbt는 rmapbt처럼 overlapping btree이고 root 안에 record를 둘 수 있다. verifier는 metadir과 parent pointer가 있고 metadata flag가 붙은 inode에서만 이 fork format을 받는다.

rt file `a`(inode 131)에 1 MiB를 쓰고 `b`(inode 132)로 reflink한 뒤 `b`의 [64 KiB, 128 KiB)를 덮어썼다. `b`의 그 구간은 group 0이 아니라 group 1의 새 자리로 옮겨 갔고, `a`의 extent는 세 조각으로 나뉘었다. 그 뒤 두 btree의 record는 다음과 같았다.

| group | btree | 시작 block | 길이 | owner 또는 refcount | 뜻 |
|---|---|---|---|---|---|
| 0 | rmap | 0 | 1 | FS (-3) | rtsb |
| 0 | rmap | 1 | 256 | 131 | `a` 전체 |
| 0 | rmap | 1 | 16 | 132 | `b`의 앞 64 KiB |
| 0 | rmap | 33 | 224 | 132 | `b`의 128 KiB부터 끝까지 |
| 0 | refcount | 1 | 16 | 2 | 공유가 남은 구간 |
| 0 | refcount | 33 | 224 | 2 | 공유가 남은 구간 |
| 1 | rmap | 16 | 16 | 132 | `b`가 덮어쓴 구간의 새 자리 |

`b`가 덮어쓴 block 17~32는 이제 `a`만 쓰므로 refcount record가 없다. group 1의 refcountbt와 group 2, 3의 두 btree는 비어 있었고, 모든 btree가 level 0인 root 하나에 record를 담고 있었다. reflink와 COW는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md)에, record 형식은 [rmap과 refcount 구조](10-rmap-refcount.md)에, inode-rooted btree는 [B+tree 공통 구조](05-btree.md)에, fork format은 [inode](06-inode.md)에 있다.

code: xfs_rtrmap_btree.c, xfs_rtrefcount_btree.c, xfs_format.h, xfs_inode_buf.c

## metadir

### root와 tree

metadir root directory의 inode 번호는 superblock의 `sb_metadirino`에 있고, mount 때 `xfs_mount_setup_metadir`가 root를 읽어 둔다. tree의 형식은 보통의 XFS directory와 같고, [directory와 symlink](07-directory.md)에 있다. 이름은 `xfs_metadir_lookup`이 찾는다. 찾는 쪽이 ftype을 주었는데 맞지 않으면 corrupt로 본다. mount는 ftype을 주지 않아서 실험의 lookup은 모두 type unknown이었다.

지금 code는 metadir 아래에 directory를 둘만 만든다.

- `/rtgroups/`: group마다 `<rgno>.bitmap`, `<rgno>.summary`, `<rgno>.rmap`, `<rgno>.refcount`가 있다. bitmap과 summary는 zoned가 아닐 때만, rmap은 rmapbt를, refcount는 reflink를 켰을 때만 만든다.
- `/quota/`: `user`, `group`, `project`가 있다. 실험의 file system은 quota를 켜지 않아 `/quota`가 없었다.

inode의 종류는 `di_metatype`(옛 `di_onlink` 자리)에 적고, 읽을 때 기대한 type과 다르면 corrupt로 본다. 주석에 따르면 metadir 아래 file은 아직 unlink할 수 없다.

code: xfs_mount.c, xfs_metadir.c, xfs_rtgroup.c, xfs_dquot_buf.c, xfs_format.h, xfs_icache.c

### metadata inode의 표지와 격리

metadata inode에는 `XFS_DIFLAG2_METADATA` flag가 붙는다. 이 flag를 정의한 곳의 주석에 따르면 metadata inode는 root directory로 찾아지면 안 되고, userspace가 접근해서도 안 되며, metadata directory entry는 ftype이 맞아야 한다. inode 자체의 제약은 아래와 같다. 값은 `xfs_metafile_set_iflag`가 채우고 `xfs_dinode_verify_metadir`가 검사한다.

| 항목 | 규칙 | fs A의 `0.bitmap` |
|---|---|---|
| mode | regular file이나 directory, 권한 bit 0 | `0100000` |
| uid, gid | 0 | 0, 0 |
| project ID | 32-bit 표지. rtgroup file이면 group 번호이고, load할 때 다르면 corrupt | 0 (group 1~3의 bitmap은 1~3, `2.rmap`은 2) |
| flag | immutable, sync, noatime, nodump, nodefrag 필수, dax 금지 | 다섯 다 1, `v3.metadata = 1` |
| nosymlinks | directory는 필수 | 0 (root와 `/rtgroups`만 1) |
| metatype | unknown 0, dir 1, quota 2~4, rtbitmap 5, rtsummary 6, rtrmap 7, rtrefcount 8 | 5. 다른 file과 directory도 규칙의 값과 같았다 |

metadata inode는 주로 `S_PRIVATE` flag로 user에게서 떼어 놓는다. XFS는 inode를 setup할 때 metadata inode에 이 flag를 세우고 xattr 연산 표지를 떼어, LSM과 ACL이 이 file에 닿지 않게 한다(code로만 확인).

- NFS file handle 경로(`xfs_nfs_get_inode`)는 이 flag가 있으면 `-ESTALE`을 돌려주고, bulkstat도 `XFS_IBULK_METADIR` flag가 없으면 같은 검사로 거른다(code로만 확인).
- 보통의 lookup이 metadata inode에 닿으면 corrupt로 끝난다. 이 검사로 일반 directory entry가 metadata inode를 가리키는 손상을 잡는다(code로만 확인).
- fs A를 mount한 root에는 `.`과 `..`뿐이었고 `rtgroups`는 없었다.

code: xfs_format.h, xfs_metafile.c, xfs_inode_buf.c, xfs_rtgroup.c, xfs_iops.c, xfs_export.c, xfs_itable.c, xfs_inode.c

### 옛 방식과의 관계

metadir 이전에는 rtbitmap, rtsummary, quota inode를 superblock이 직접 가리켰다(`sb_rbmino`, `sb_rsumino`). metadir file system에서는 이 두 field에 0을 쓰고, rt inode는 `/rtgroups/<n>.<type>`을 lookup해서 가져온다. fs A의 두 field는 0이었고, fs B는 `rbmino = 129`, `rsumino = 130`이었다.

mount는 metadir root를 읽은 뒤 `xfs_rtmount_inodes`에서 `rtgroups` directory를 먼저 찾고, group마다 bitmap, summary, rmap, refcount 순으로 찾는다. quota는 그 뒤에 mount한다(root와 quota의 순서는 code로만 확인). group 4개인 fs A에서는 `xfs_metadir_lookup`이 17번 찍혔다(`rtgroups` 한 번, group마다 네 번).

metadir 쪽 주석에는 옛 inode 다섯(rtbitmap, rtsummary, quota 셋)을 이 계층이 관리하지 않는다고 적혀 있다. 그러나 metadir file system에서는 같은 file들이 metadir 아래로 들어간다. 이 주석은 metadir 이전의 file system을 말한다(추정).

code: xfs_sb.c, xfs_rtgroup.c, xfs_rtalloc.c, xfs_mount.c, xfs_metadir.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs_format.h의 `XFS_DFL_RTEXTSIZE`, mkfs.xfs man page | 기본 rt extent 크기 64 KiB (non-striped volume) | mkfs 7.1.1과 실험에서는 1 block이었다. rt reflink도 rextsize 1을 요구한다. |
| realtime.asciidoc | rtsummary의 값은 free extent가 없으면 0, 있으면 양수 | 0이거나 양수라는 말은 맞다. 다만 값은 free extent의 개수라서 1보다 클 수 있다. |
| realtime.asciidoc | mkfs 예의 `-r rtgroups=1` | xfsprogs 7.1.1에는 이 suboption이 없다. |
| realtime.asciidoc | 같은 절에 free space 정보가 volume 전체에 file 하나로 있다는 문장과, rtgroup마다 따로 있다는 문장이 함께 있다 | 뒤의 문장이 맞다. group마다 `N.bitmap`과 `N.summary`가 따로 있다. |
| realtime.asciidoc | "The first block of each realtime group contains a superblock." | rtsb는 group 0에만 있다. 같은 문서의 "Group 0 has a super block"이 code와 맞다. |
| internal_inodes.asciidoc | metadir 예의 `rbmino = 130`, `rsumino = 131` | kernel은 metadir file system에서 이 값을 0으로 쓰고, fs A도 둘 다 0이었다. |
| rtrmapbt.asciidoc | xfs_db 예의 `core.format = 5 (rmap)` | format 이름은 `meta_btree`다. 번호 5는 같다. |

## 한계점

다음은 확인하지 못했다.

- near 할당의 결과를 size 할당이 덮어쓰는 것이 의도인지, upstream에서 이후 바뀌었는지. 읽은 tree에서는 git history를 쓸 수 없었다.
- rtgroup끼리 동시에 할당되는지. 동시성은 관찰하지 않았다.
- rtgroup이 없을 때 해제한 rt 공간이 commit 전에 실제로 다시 할당되는지. busy extent가 없다는 것만 읽었다.
- rt block 번호의 sparse 구멍과 접힌 disk 주소의 차이. 구멍이 없는 geometry라 bmap의 값이 어느 쪽인지 가르지 못했다.
- COW 쓰기가 group 0이 아니라 group 1로 간 이유.
- mount 중 fsmap에 group 1의 block 0~15가 "cow reservation"으로 있었지만 unmount 뒤 group 1 refcountbt에는 record가 없었다. 이 예약이 무엇이었는지.
- quota를 켠 뒤 `/quota` 아래의 구성. mount는 quota 없이만 했다.
- metadir 없이 rtdev만 주고 `rgcount`를 줄 때 mkfs가 이를 무시하는지. xfsprogs source만 읽었고 실행하지 않았다.
- label을 바꿀 때 rtsb가 갱신되는 경로. code로만 읽었다.

## 실험 환경

공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- data device와 rt device로는 각각 guest `/dev/shm`의 1 GiB image를 loop device로 붙였다. 관찰은 한 번씩 실행했다.
- fs A는 `mkfs.xfs -m metadir=1 -r rtdev=<rt loop>,rgcount=4`, fs B는 `mkfs.xfs -r rtdev=<rt loop>`로 만들었다.
- mount에는 `-o rtdev=<rt loop>`를 주었고, 옵션은 `rw,relatime,inode64,logbufs=8,logbsize=32k,noquota`였다. rt file은 directory에 `xfs_io -c "chattr +t"`를 걸고 그 아래에서 만들었다. on-disk 내용은 unmount 뒤 `xfs_db -r -R <rt loop>`로 읽었다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| superblock, rtsb, metadir tree | fs A를 mount하는 동안 event를 켜고, unmount 뒤 rtsb, superblock, metadir tree, metadir 아래 inode 여섯의 core를 읽음 | `xfs_metadir_lookup` | 배치도, rtsb, metadata inode 표, mount 때의 lookup |
| bitmap과 summary | fs A와 fs B의 rt directory에 1 MiB file 4개를 쓰고 fsync한 뒤 `bmap -v`로 위치를 보고, unmount 뒤 superblock, bitmap과 summary의 첫 block, bitmap inode를 읽음 | 없음 | rtbitmap 표, free extent 따라가기, 1 MiB file 4개의 위치, 켜는 조건의 fs B mkfs 출력, rt device로 가는 것의 flag(`t`, `r`), 옛 방식과의 관계의 fs B `rbmino`와 `rsumino` |
| free 경로 | 두 file system에서 1 MiB rt file을 쓰고 sync한 뒤, event를 켜고 rm과 sync | `xfs_itruncate_extents_start`, `xfs_bunmap`, `xfs_defer_create_intent`, `xfs_defer_finish_item`, `xfs_extent_free_deferred`, `xfs_extent_busy`, `xfs_rmap_deferred` | free 경로 표 |
| rt rmapbt와 rt refcountbt | fs A에서 `a`에 1 MiB를 쓰고 `b`로 reflink한 뒤 `b`의 [64 KiB, 128 KiB)를 덮어씀. 단계마다 `bmap -v`와 `fiemap -v`, 끝에 `fsmap -r -vv`. unmount 뒤 group마다 두 btree를 읽음 | 없음 | record 표 |

### 출력 읽는 법

- `fsmap`도 `bmap`처럼 512 byte 단위로 찍는다. 1 MiB file이 2048로, 4 KiB block 하나가 8로 나온다.
- fs B의 bitmap과 summary는 xfs_db가 hex dump로 찍는다. summary의 칸은 32-bit이고, rtgroup이 없으면 block 처음부터 놓인다. 그래서 줄 `220:`의 byte 0x220(544)은 칸 136이 된다. 이 줄의 첫 4 byte는 host-endian 1이다.
- fs A의 출력에는 bitmap에서 `0xffffffff`가 아닌 word만, summary에서 0이 아닌 칸만 남겼다. bitmap을 거른 `head -20`보다 출력이 짧아 잘린 줄은 없다.
- free 경로 trace에서 group 번호는 `rtgno`, group 안 block 번호는 free 줄이 `gbno`, busy 줄이 `rtgbno`로 찍힌다. `fsbcount 0x100`은 1 MiB다.
- `xfs_metadir_lookup`은 dir inode, 이름, 찾은 inode를 16진으로 찍는다. 0x81은 129, 0x120080은 1179776이다.

### 도구에서 생긴 일

bitmap과 summary 관찰의 출력은 fs B의 출력을 거르는 filter를 고친 뒤 다시 실행해 얻었다. 처음 filter를 짤 때 fs B의 bitmap과 summary가 hex dump로 찍히는 것을 고려하지 않아 줄 머리만 남았다.

### 원본 출력

#### superblock, rtsb, metadir tree

(원본 출력 "mkfs가 고른 rt extent 크기": `raw/28-realtime.md`)

(원본 출력 "fs A: superblock의 rt와 metadir 필드": `raw/28-realtime.md`)

(원본 출력 "fs A: rtsb": `raw/28-realtime.md`)

(원본 출력 "fs A: metadir tree": `raw/28-realtime.md`)

(원본 출력 "fs A: metadir 아래 inode의 core (일부)": `raw/28-realtime.md`)

(원본 출력 "fs A: mount한 file system의 root": `raw/28-realtime.md`)

(원본 출력 "fs A: mount 때 xfs_metadir_lookup의 순서": `raw/28-realtime.md`)

#### bitmap과 summary

(원본 출력 "fs A와 fs B의 mkfs 출력 (rt 쪽과 feature 줄)": `raw/28-realtime.md`)

(원본 출력 "rt directory와 새 file의 flag": `raw/28-realtime.md`)

(원본 출력 "fs A: group 0과 group 1의 첫 bitmap block": `raw/28-realtime.md`)

(원본 출력 "fs A: group 0의 summary 첫 block (group 1~3도 같은 칸)": `raw/28-realtime.md`)

(원본 출력 "fs B: summary에서 0이 아닌 줄": `raw/28-realtime.md`)

#### 1 MiB file 4개의 위치

(원본 출력 "fs B: 1 MiB file 4개의 위치, superblock, bitmap inode의 순번": `raw/28-realtime.md`)

(원본 출력 "fs A: 1 MiB file 4개의 위치": `raw/28-realtime.md`)

#### free 경로

(원본 출력 "fs A: rt file 하나를 지운 뒤의 trace": `raw/28-realtime.md`)

(원본 출력 "fs B: 같은 workload의 trace": `raw/28-realtime.md`)

#### rt rmapbt와 rt refcountbt

(원본 출력 "reflink 직후와 COW 덮어쓰기 뒤의 bmap, 그리고 fsmap": `raw/28-realtime.md`)

(원본 출력 "fs A: group 0~3의 rt rmapbt와 rt refcountbt": `raw/28-realtime.md`)
