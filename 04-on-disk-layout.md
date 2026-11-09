# on-disk 배치

> XFS 전반 wiki, 묶음 "기초". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/04-on-disk-layout.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/04-on-disk-layout.md, 접힌 원본 출력은 raw/04-on-disk-layout.md.

## 개요

- XFS는 data device를 크기가 같은 AG(allocation group)로 나누고, AG마다 처음 네 sector에 superblock 사본, AGF, AGI, AGFL을 둔다. 이 header 넷은 다른 metadata가 가리키지 않고, kernel이 AG 번호로 자리를 계산해 바로 읽는다.
- feature를 모르는 kernel에서 superblock의 compat bit는 경고만 하고, ro_compat bit는 rw mount를, incompat bit는 mount 자체를 막는다. log_incompat bit는 log가 dirty하면 recovery를 막고, clean하면 mount는 되지만 superblock을 처음 쓸 때 shutdown된다.
- v5 metadata block에는 CRC, magic, UUID, 자기 주소나 AG 번호, owner, LSN이 들어 있다. verifier는 block 하나만 보고 CRC로 block 안의 손상을, 나머지 field로 엉뚱한 자리에 놓인 block을 가려낸다.

이 page를 읽기 전에 [관찰 도구](02-observation-tools.md)의 xfs_db 사용법과 [code를 읽기 위한 최소 모형](03-code-model.md)의 주소 type을 먼저 알아 두면 좋다. 예의 값은 1,025,000 KiB와 1 GiB file system(기본 mkfs와 4096 byte sector), 공통 예제 image, debug image에서 얻었다.

## 구조

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### AG로 나누기

data device는 크기가 같은 AG `sb_agcount`개로 나뉘고, 나누어떨어지지 않으면 마지막 AG만 짧아진다. AG 하나의 block 수는 superblock의 `sb_agblocks`에 있다. 예를 들어 1,025,000 KiB file system(4 KiB block 256250개)은 AG 4개로 나뉘어, AG 크기가 64062.5를 올림한 64063 block이 되고 마지막 AG 3만 64061 block(256250 - 3 × 64063)이 된다.

- kernel은 AG 하나가 16 MiB 이상 1 TiB 이하이고 마지막 AG가 64 block 이상인 geometry만 받아들인다. superblock을 읽을 때 `dblocks`를 `agblocks`로 나눠 올림한 값이 `sb_agcount`와 같은지, `sb_agblklog`가 `agblocks - 1`의 최고 bit + 1인지도 본다. 다르면 "SB sanity check failed"를 찍고 -EFSCORRUPTED로 거부한다.
- AG 수와 크기는 mkfs가 정한다(code로만 확인).

| mkfs의 조건 | AG 수 |
|---|---|
| `-d agcount=`나 `agsize=`를 줌 | 그 값 |
| `-d concurrency=`를 줌 | 그 값을 목표로 하되, AG가 4 GiB보다 작아지면 수를 줄이고 기본 계산의 수보다 적게는 만들지 않는다. 32 GiB device에 16을 주면 8개 |
| 지정 없음, 비회전 device(SSD) | CPU 수를 목표로 위와 같은 두 하한을 적용 |
| 지정 없음, 회전 disk 등, stripe unit 없음 | 128 MiB 이상 4 TiB 미만이면 4개, 4 TiB 이상이면 최대 AG 크기 |
| 지정 없음, 회전 disk 등, stripe unit 있음 | 32개에서 device 크기에 따라 줄인다 |

그래서 CPU가 많은 host의 큰 SSD에서는 AG가 4개보다 많을 수 있다.

agblocks가 2의 거듭제곱이 아니면 fsblock 번호에 빈 구간이 생긴다. fsblock 번호는 AG 번호를 `sb_agblklog` bit만큼 왼쪽으로 민 값에 AG 안의 block 번호를 더해 만든다. 1,025,000 KiB file system은 `sb_agblklog`가 16이라 AG 1이 fsblock 65536에서 시작하고, 64063~65535는 어느 block도 가리키지 않는다. 주소 type 사이의 변환은 [code를 읽기 위한 최소 모형](03-code-model.md)에 있다.

code: xfs_ag.c, xfs_sb.c, xfs_fs.h, xfs_format.h

### 각 AG의 앞

AG마다 처음 네 sector에는 superblock 사본, AGF, AGI, AGFL이 차례로 놓인다(`XFS_AGF_DADDR` 등). 단위는 512 byte가 아니라 file system sector(`sb_sectsize`)라서, 4096 byte sector로 만들면 header 하나가 4096 byte를 차지한다. kernel은 다른 metadata를 거치지 않고 AG 번호로 자리를 계산한다(`XFS_AG_DADDR`). XFS에서 자리가 고정된 metadata는 이 넷뿐이다.

새 AG를 만들 때는 header 뒤의 고정 번호에 AG btree의 첫 root block이 bnobt, cntbt, inobt, finobt, rmapbt, refcountbt 순으로 놓인다(꺼진 feature의 btree는 빠진다). 그 다음 block부터는 free space가 된다(`xfs_prealloc_blocks`). 1,025,000 KiB file system의 AG 1에서는 mkfs가 root block 바로 뒤의 6 block(7~12)을 AGFL에 넣었다.

**도식: AG 배치와 AG 1의 앞**

```text
data device: 4 KiB block 256250개를 AG 4개로 나눔

   AG 0          AG 1          AG 2          AG 3
+-------------+-------------+-------------+-------------+
| 64063 block | 64063 block | 64063 block | 64061 block |
|             |             | (log 포함)  |             |
+-------------+-------------+-------------+-------------+
              |
              v  AG 1의 앞 (512 byte sector)

block 0                 block 1~6         block 7~12   block 13~64062
+-----+-----+-----+-----+-----------------+------------+----------------
| SB  | AGF | AGI | AGFL| btree root 여섯 | AGFL에     | free space
|     |     |     |     | bnobt, cntbt,   | 넣은 예비  | 13부터
|     |     |     |     | inobt, finobt,  | block 6개  | 64050 block
|     |     |     |     | rmapbt, refcnt  |            |
+-----+-----+-----+-----+-----------------+------------+----------------
  sector 0~3 (512 byte씩)
```

sector 크기에 따라 root block의 번호가 밀린다. 512 byte sector 열은 위 file system에서 본 값이고(root는 AG 0, AGFL과 free space는 AG 1), 4096 byte sector 열은 `-s size=4096`으로 만든 1 GiB file system의 AG 0에서 본 값이다.

| 자리 | 512 byte sector | 4096 byte sector |
|---|---|---|
| header 넷 | block 0 안에 512 byte씩 | block 0~3에 하나씩 |
| btree root 여섯 | block 1~6 | block 4~9 |
| mkfs가 AGFL에 넣은 block | 7~12 | 10~15 |
| free space 시작 | 13 | 찍지 않았다 |

이 고정 번호는 새 AG를 만들 때 root를 처음 놓는 자리일 뿐이고, tree의 높이가 바뀌면 root는 다른 block으로 옮겨 간다. kernel은 root를 언제나 AGF와 AGI의 root field(`agf_bno_root`, `agi_root` 등)로 찾는다. 공통 예제 image의 AG 0에서는 bnobt가 level 2로 자라 root가 block 4724에 있고, 덜 쓴 AG 1에서는 root가 처음 자리(bnobt 1, cntbt 2, rmapbt 5, refcountbt 6)에 그대로 있다.

이 배치는 `xfs_ag_init_headers`가 만든다. kernel에서는 growfs가 새 AG에 이 함수를 부르고, mkfs도 같은 libxfs 함수를 부른다.

code: xfs_format.h, xfs_ag.c, xfs_alloc.c, xfs_ialloc.c

### AG마다 다른 것

- root inode chunk는 AG 0에만 있다. 1,025,000 KiB file system에서는 `sb_rootino`가 128이고 AG 0의 AGI는 count 64, freecount 61이다. AG 0의 freeblks(64042)는 AG 1의 free extent(64050 block)보다 8 block 적다. 이 차이는 root inode chunk 하나(inode 64개 × 512 byte = 8 block)만큼이다(추정).
- internal log는 AG 하나 안에 있고, 그 AG의 free space는 log 길이만큼 줄어든다. mkfs는 기본으로 AG `agcount / 2`의 header와 root block 바로 뒤에 log를 둔다. 위 file system에서는 logstart 131079가 AG 2의 block 7이고, log 16384 block 뒤의 free extent는 block 16397에서 시작한다.
- AGFL을 처음 채우는 쪽이 다르다. kernel growfs는 AGFL을 비운 채 만들고(flfirst 1, fllast 0, flcount 0), mkfs는 header를 만든 뒤 `libxfs_alloc_fix_freelist()`로 채운다. 언제 얼마나 채우는지는 [free space 구조와 할당 정책](09-free-space.md)에 있다.

code: xfs_ag.c, xfs_ag.h

### superblock과 AG header

header의 크기는 compile 때 `static_assert`로 고정된다. superblock은 304 byte, AGF는 224 byte, AGI는 344 byte, AGFL header는 36 byte다.

#### superblock

superblock에는 file system 전체의 geometry, 식별자, feature, 전역 counter가 들어 있다. on-disk 형은 big-endian `struct xfs_dsb`, in-core 형은 `struct xfs_sb`이고, `xfs_sb_from_disk`와 `xfs_sb_to_disk`가 둘을 바꾼다. 아래 값은 공통 예제 image의 primary superblock에서 읽었다.

| 묶음 | field (xfs_db 이름) | 공통 예제 image의 값 |
|---|---|---|
| geometry | blocksize, dblocks, agblocks, agcount, sectsize, inodesize, logstart, logblocks, log2 값(blocklog, sectlog, inodelog, inopblog, agblklog) | 4096, 524288, 131072, 4, 512, 512, 262151, 16384, log2 값은 12, 9, 9, 3, 17 |
| 식별 | uuid, meta_uuid, fname(label) | uuid는 0ef0b5c2-41de-48a9-9829-c3d46154a46e, meta_uuid는 0 |
| 시작점 inode | rootino, 그 밖에 rtbitmap, rtsummary, quota inode(rbmino, rsumino, uquotino, gquotino, pquotino) | 128, 그 밖은 129~133 |
| 전역 counter | icount, ifree, fdblocks | 40576, 2020, 500447 |
| format 판과 feature | versionnum, features2, features_compat, features_ro_compat, features_incompat, features_log_incompat | 0xb4f5, 0x18a, 0, 0xf, 0xeb, 0 |
| v5 무결성 | crc, lsn | 0xb1475487 (correct), 0x10000780f |
| 작업 중 표시 | inprogress | 0 |

- `sb_inprogress`가 켜져 있으면 mount를 거부한다("Offline file system operation in progress!").
- 전역 counter는 lazy counter다. v5에서는 lazysbcount가 늘 켜져 있고, superblock을 log할 때마다 percpu counter의 합을 넣는다(`xfs_log_sb`). 주석에 따르면 이 값은 unmount 때만 정확하고 그 밖에는 어림값("ballpark")이며, 비정상 종료 뒤에는 AGF 값으로 다시 맞춘다.

superblock 사본은 모든 AG의 sector 0에 있지만, kernel은 평소에 AG 0의 primary 하나만 고친다. kernel은 growfs, label 변경, realtime device의 growfs(`xfs_update_secondary_sbs`)와 online repair(`xrep_superblock`) 때만 secondary를 primary 내용으로 다시 쓴다. 이런 일이 없으면 secondary의 counter와 inprogress에는 mkfs가 처음 쓴 값이 남는다. unmount된 file system에서 xfs_admin으로 label이나 UUID를 바꾸면 xfs_db가 모든 superblock을 고쳐 쓰지만 counter는 그대로 둔다. mount된 file system에서 xfs_admin -L은 kernel의 label 변경을 부르므로 secondary의 counter도 primary 값으로 바뀐다. xfs_repair가 깨진 secondary를 고쳐 쓰면 counter는 0이 된다. kernel은 mount 때 primary만 읽고, secondary의 inprogress와 counter는 보지 않는다. secondary는 primary가 깨졌을 때 xfs_repair가 backup으로 쓴다.

| field | primary(sb 0) | secondary(sb 1) |
|---|---|---|
| icount, ifree | 40576, 2020 | 0, 0 |
| fdblocks | 500447 | 507876 |
| lsn | 0x10000780f | 0 |
| inprogress | 0 | 1 |
| feature mask 넷 | 0, 0xf, 0xeb, 0 | primary와 같다 |

secondary의 값은 mkfs가 남기는 초기값과 맞는다. mkfs는 inprogress를 1, icount와 ifree를 0, fdblocks를 `dblocks - agcount * prealloc - logblocks`로 두고, 마지막에 primary의 inprogress만 0으로 쓴다. 그래서 secondary의 inprogress 1은 손상이 아니다. 공통 예제 image에서는 prealloc이 header block 하나와 root 여섯을 더한 7이라, fdblocks는 524288 - 4 × 7 - 16384 = 507876이 된다. sb 3도 counter, lsn, crc가 sb 1과 같았다. growfs가 새 AG에 쓰는 secondary도 처음에는 inprogress가 1이다.

code: xfs_format.h, xfs_sb.c, xfs_super.c, xfs_ondisk.h

#### AGF

AGF에는 AG의 free space 요약과 공간 btree(bnobt, cntbt, rmapbt, refcountbt)의 root가 들어 있다. 값은 공통 예제 image의 AG 0과 AG 1에서 읽었다.

| 묶음 | field (xfs_db 이름) | AG 0 | AG 1 |
|---|---|---|---|
| 공통 header | magicnum("XAGF"), versionnum, seqno(AG 번호), length(AG block 수) | 0x58414746, 1, 0, 131072 | 0x58414746, 1, 1, 131072 |
| btree root | bnoroot, cntroot, rmaproot, refcntroot | 4724, 7462, 8, 7464 | 1, 2, 5, 6 |
| btree level | bnolevel, cntlevel, rmaplevel, refcntlevel | 2, 2, 2, 2 | 1, 1, 1, 1 |
| AGFL의 활성 구간 | flfirst, fllast, flcount | 18, 29, 12 | 1, 6, 6 |
| free space 요약 | freeblks, longest(가장 긴 free extent), btreeblks, rmapblocks, refcntblocks | 124589, 123605, 17, 14, 4 | 131004, 131002, 0, 1, 1 |
| v5 | uuid, lsn, crc | lsn 0x1000077dc, crc correct | lsn 0x100007550, crc correct |

- root block의 짜임은 [B+tree 공통 구조](05-btree.md)에 있다.
- LSN과 CRC는 transaction이 log하지 않고, block을 disk에 쓸 때 채운다(주석의 "unlogged fields, written during buffer writeback").
- kernel은 AGF를 읽을 때 freeblks 같은 값 일부를 per-AG 구조에 둔다(`pagf_freeblks` 등).
- 새 AG의 AGF는 `xfs_agfblock_init`가 채운다. root는 위의 고정 block, level은 1이고, freeblks와 longest는 `agsize - m_ag_prealloc_blocks`다. log가 든 AG는 log 길이만큼 더 뺀다.

#### AGI

AGI에는 AG의 inode 할당 상태, inobt와 finobt의 root, unlinked list의 머리가 들어 있다. 값은 공통 예제 image의 AG 0과 AG 1에서 읽었다.

| 묶음 | field (xfs_db 이름) | AG 0 | AG 1 |
|---|---|---|---|
| 공통 header | magicnum("XAGI"), versionnum, seqno, length | 0x58414749, 1, 0, 131072 | 0x58414749, 1, 1, 131072 |
| inode 수 | count(할당된 inode), freecount | 33408, 1738 | 320, 7 |
| inobt | root, level | 4779, 2 | 3, 1 |
| finobt | free_root, free_level | 7069, 2 | 4, 1 |
| btree block 수 | ino_blocks, fino_blocks | 4, 4 | 1, 1 |
| 할당 힌트 | newino, dirino | 56896, null | 384, null |
| unlinked list | unlinked[0-63] | 값 없이 찍힘 | 값 없이 찍힘 |
| v5 | uuid, lsn, crc | lsn 0x100007550, crc correct | lsn 0x100007550, crc correct |

- inode 위치를 찾는 표는 없다. inode 번호를 풀면 위치가 나오기 때문이다. 번호의 구성은 [code를 읽기 위한 최소 모형](03-code-model.md)에 있다.
- `agi_unlinked`는 bucket 64개로 된 hash 사슬의 머리다. link 수가 0이 됐지만 아직 열려 있는 inode가 여기에 묶이고, 새 AG에서는 bucket이 모두 NULLAGINO다. 뒤처리는 [inode 수명](15-inode-lifecycle.md)에 있다.
- kernel은 freecount 같은 값을 per-AG 구조에 둔다(`pagi_freecount`).

code: xfs_format.h, xfs_alloc.c, xfs_ialloc.c, xfs_ag.c

#### AGFL

AGFL은 free space btree와 rmapbt가 자라거나 줄 때 쓸 block을 미리 떼어 둔 목록이다. 가득 찬 AG에서 block이 해제될 때도 free space btree를 갱신할 수 있게 하려는 것이다.

- v5 형식에서는 36 byte header(magic "XAFL", seqno, uuid, lsn, crc) 뒤에 AG block 번호(`__be32`) 배열이 sector 끝까지 이어진다. v4는 header 없이 배열만 있다.
- 칸 수는 `(sectsize - 36) / 4`다(`xfs_agfl_size`). 512 byte sector면 119칸, 4096 byte sector면 1015칸이다.
- AGFL은 원형 목록이다. AGF의 flfirst부터 fllast까지 flcount개 칸(활성 구간)에 block이 들어 있다. 꺼낼 때는 flfirst를 하나 올리고(`xfs_alloc_get_freelist`), 넣을 때는 fllast를 하나 올린다(`xfs_alloc_put_freelist`). 끝 칸에 닿으면 0으로 돌아가고, 활성 구간 밖의 칸은 의미가 없다.

공통 예제 image의 AG 0은 flfirst 18, fllast 29, flcount 12이고 칸은 이렇게 차 있다.

| 칸 | 0~17 | 18~24 | 25~29 | 30~118 |
|---|---|---|---|---|
| block | null | 2960~2966 | 1247, 1249, 1253, 1257, 1263 | null |

여기서 하나를 꺼내면 칸 18의 block 2960이 나가고 flfirst는 19, flcount는 11이 된다. 꺼내지 않고 하나를 넣으면 칸 30에 들어가고 fllast는 30, flcount는 13이 된다(code로만 확인).

code: xfs_alloc.c, xfs_alloc.h, xfs_rmap_btree.c

## code 경로

이 장의 동작은 code로 확인했고, mount와 손상 시험의 결과는 관찰한 것이다.

### feature bit 검사

#### "모른다"의 뜻

v5 superblock에는 32-bit feature mask가 넷 있다. kernel이 아는 bit는 종류마다 `*_ALL` mask에 모여 있고, 나머지는 모두 모르는 bit(`*_UNKNOWN`)로 친다. 아는 bit는 mount 때 `m_features`로 옮겨지고, 모르는 bit는 옮길 곳이 없다.

| mask | 뜻 | 기준 kernel이 아는 bit |
|---|---|---|
| compat | 옛 kernel이 읽고 써도 된다 | 없다. 켜진 compat bit는 모두 모르는 bit다 |
| ro_compat | 읽기는 되지만 쓰면 깨진다 | finobt, rmapbt, reflink, inobtcount |
| incompat | mount하면 안 된다 | ftype부터 zone_gaps까지 11개 |
| log_incompat | 옛 kernel이 replay할 수 없는 log 형식이나 transaction. clean unmount가 지운다 | logged xattrs 하나 |

뜻은 `XFS_SB_FEAT_COMPAT_ALL` 위 주석에 있다. 기본 mkfs image의 mask는 compat 0, ro_compat 0xf, incompat 0xeb, log_incompat 0이고, mkfs 출력에서 ro_compat의 네 feature가 모두 켜져 있다. 모르는 bit가 ro_compat에만 있다면, 예를 들어 reflink bit 하나만 모르는 kernel이라면 ro mount만 된다. 실제로 reflink를 모르는 옛 kernel은 그보다 늦게 들어온 bigtime, parent 같은 incompat bit도 모르므로 이 image를 아예 mount하지 못한다.

#### mount 경로와 검사 위치

read-only 여부는 superblock을 읽기 전에 정해진다. 그 뒤 `xfs_readsb`가 superblock을 verifier를 붙여 다시 읽고, read verifier `xfs_sb_read_verify`가 CRC, `xfs_validate_sb_common`, `xfs_validate_sb_read` 순으로 본다. feature mask 검사는 마지막 함수에 있고 v5에서만 돈다.

| mask | ro mount | rw mount | ro → rw remount | 막는 곳 |
|---|---|---|---|---|
| compat | 된다(code로만 확인) | 된다. 쓴 뒤에도 bit가 남는다 | 된다(code로만 확인) | 없다. read verifier가 경고만 찍고, write verifier와 log recovery는 보지 않는다 |
| ro_compat | 된다 | 거부 | 거부 | read verifier(rw면 -EINVAL), remount, write verifier(rw면 -EFSCORRUPTED) |
| incompat | 거부 | 거부 | mount부터 안 된다 | read verifier(늘 -EINVAL), write verifier(-EFSCORRUPTED) |
| log_incompat | log가 clean하면 성공하지만 mount 중 superblock을 쓸 때 shutdown(code로만 확인). dirty하면 거부, norecovery를 더하면 된다 | log가 clean하면 되지만 superblock을 쓸 때 shutdown. dirty하면 거부 | 알 수 없다(한계점) | log recovery(log가 dirty하면 -EINVAL), write verifier(-EFSCORRUPTED). read verifier는 보지 않는다 |

code: xfs_sb.c, xfs_super.c, xfs_mount.c, xfs_log_recover.c

#### compat, ro_compat, incompat

- compat: read verifier는 "Superblock has unknown compatible features (0x%x) enabled."와 "Using a more recent kernel is recommended."만 찍고 계속한다. write verifier도 compat을 보지 않으므로, rw로 file을 만들고 unmount한 뒤에도 bit가 남았다. xfs_db도 같은 두 줄을 찍는다.
- ro_compat: rw mount면 read verifier가 "Attempted to mount read-only compatible filesystem read-write."와 함께 -EINVAL을 내고, read-only면 통과한다. ro mount 뒤의 rw remount는 `xfs_remount_rw`가 "ro->rw transition prohibited on unknown (0x%x) ro-compat filesystem"과 -EINVAL로 막는다. write verifier는 rw 상태에서 모르는 ro_compat bit를 in-memory 손상으로 본다.
- incompat: ro 여부를 보지 않고 "Filesystem cannot be safely mounted by this kernel."을 찍고 -EINVAL로 거부한다. 이 오류는 -EFSCORRUPTED가 아니라서 verifier 오류가 아니라 buffer I/O error가 된다(`xfs_verifier_error` 대신 `xfs_buf_ioerror`). 그래서 kernel log에는 "Metadata corruption detected" 같은 verifier 보고 없이 "SB validate failed with error -22."까지 세 줄만 남았다.
- v4의 `sb_versionnum`, `sb_features2`에 모르는 bit가 있으면 `xfs_sb_good_version`이 거짓을 내어 -EWRONGFS로 거부한다. v5도 v4 필수 bit가 맞는지 본다. 관찰 kernel은 v4 자체를 거부한다(`CONFIG_XFS_SUPPORT_V4`).
- 아는 incompat이어도 needsrepair가 켜져 있으면 norecovery가 아닌 mount를 -EFSCORRUPTED로 거부한다.

#### log_incompat

- read verifier는 log_incompat을 보지 않는다. log가 dirty하면(tail과 head가 다르면) `xlog_recover`가 모르는 bit를 보고 "The log can not be fully and/or safely recovered by this kernel."과 함께 -EINVAL을 낸다. norecovery mount는 recovery를 건너뛰므로 이 검사도 건너뛴다. 관찰에서도 rw와 ro mount는 "log mount/recovery failed: error -22"로 실패했고 `ro,norecovery`만 mount됐다.
- log가 clean하면 mount는 된다. 그러나 bit를 지우는 code는 아는 bit만 지우므로 모르는 bit는 in-core superblock에 남는다. 그 superblock을 처음 disk에 쓸 때 write verifier에 걸려 file system이 shutdown된다(`SHUTDOWN_CORRUPT_INCORE`). 늦어도 unmount 때의 log covering(`xfs_log_cover`)에서 쓴다.
- norecovery가 아닌 ro mount에서도 mount 중에 `xfs_log_clean`이 superblock을 쓰고, log_incompat 검사에는 readonly 조건이 없다. 그래서 ro mount도 성공을 돌려주지만 mount 중에 shutdown된다(code로만 확인).

clean log에서 rw mount한 관찰에서는 unmount 때 superblock을 처음 썼고, "Corruption detected in superblock incompatible log features (0x80000000)!" 뒤에 shutdown됐다. 쓰기가 거부되어 disk의 bit는 그대로 남았다.

code: xfs_sb.c, xfs_log_recover.c, xfs_log.c, xfs_buf.c

#### log_incompat bit의 수명

아는 log_incompat bit는 보호할 log item을 쓰기 전에 켜지고, unmount 때만 지워진다.

- 켜기: `xfs_add_incompat_log_feature`가 primary superblock buffer에 bit를 넣고 바로 동기 write한다. 기준 tree에서는 logged xattr만 이 함수를 부른다(reflink나 rmapbt가 있어야 한다). 이 경로는 CONFIG_XFS_DEBUG로 build한 debug kernel에서 `/sys/fs/xfs/debug/larp`를 켰을 때만 탄다. 그 밖의 kernel은 이 bit를 켜지 않는다.
- parent pointer도 logged xattr를 쓰지만 bit를 켜지 않는다. parent feature 자체가 incompat bit라서 옛 kernel이 아예 mount하지 못하기 때문일 것이다(추정).
- 끄기: `xfs_clear_incompat_log_features`는 unmount가 켜는 상태(`xfs_set_done_with_log_incompat`)에서만 bit를 지운다. log quiesce와 background log covering도 지우려고 하지만 이 상태가 아니라서, freeze도 background covering도 bit를 지우지 못한다.

debug kernel에서 `larp`를 켜고 xattr를 붙이자 "EXPERIMENTAL logged extended attributes feature enabled."가 찍혔다. logged xattr 경로를 탔다는 뜻이다. features_log_incompat은 freeze 중에 0x1이었고 unmount 뒤에 0이 됐다.

code: xfs_mount.c, xfs_xattr.c, xfs_sysfs.c, xfs_log.c

### self-describing metadata 검사

v4 block에는 magic밖에 없어서, 그 block이 그 자리에 있어야 하는 것인지 알 수 없었다. v5 block에 더해진 field는 저마다 다른 잘못을 잡는다. v4와 v5의 차이는 [XFS의 설계와 역사](01-overview-history.md)에 있다. 아래 표의 -EFSCORRUPTED는 user space에 EUCLEAN("Structure needs cleaning")으로 나온다.

| field | 잡는 것 | 검사 | 손상 시험 |
|---|---|---|---|
| CRC | block 안의 bit 손상, 찢긴 write | 각 read verifier의 `xfs_buf_verify_cksum`, -EFSBADCRC | AGF 1의 CRC를 깨자 rw mount가 실패했다 |
| magic | 다른 type의 block | `xfs_verify_magic`, -EFSCORRUPTED | inobt root를 bnobt block으로 돌리자 `xfs_inobt_verify`가 잡았다 |
| UUID | 다른 file system의 block | `uuid_equal(..., &sb_meta_uuid)` | AGI 2의 uuid를 바꾸자 `xfs_agi_verify`가 잡고 mount가 실패했다 |
| blkno, seqno | 엉뚱한 자리에 쓰인 block | btree는 `bb_blkno`와 buffer 주소 비교, AG header는 `xfs_validate_ag_length` | AGI 2의 seqno는 `xfs_validate_ag_length`가 잡았다. cntbt root의 bno를 바꾼 경우는 fsync가 EUCLEAN으로 실패했다(잡은 검사는 관찰하지 않음) |
| owner | 다른 AG, 다른 inode의 block | short form은 read verifier가 `bb_owner`를 AG 번호와 비교, long form은 btree lookup이 cursor의 inode 번호와 비교 | bnobt root의 owner를 바꾸자 fsync가 EUCLEAN으로 실패했다(잡은 검사는 관찰하지 않음) |
| LSN | log보다 앞선 block, recovery 때 이미 반영된 변경 | `xfs_log_check_lsn`, `xlog_recover_get_buf_lsn` | 형식만 보았다 |

#### 검사가 도는 자리

- verifier는 block 하나만 보고 판단한다. sibling 사슬처럼 block 사이의 관계는 보지 않는다.
- 읽는 쪽이 기대하는 type을 정한다. 예를 들어 `xfs_read_agf`는 `xfs_trans_read_buf`에 `&xfs_agf_buf_ops`를 넘긴다.
- read verifier는 I/O 완료 때, write verifier는 제출 전에 돈다. write verifier가 실패하면 그 I/O를 내보내지 않고 shutdown한다.

health는 CRC 오류(-EFSBADCRC)와 손상(-EFSCORRUPTED)을 모두 sick으로 친다(`xfs_metadata_is_sick`). verifier 체계와 오류 문구, errno는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.

#### CRC

- CRC는 CRC32c(seed `~0`)로 buffer 전체에 대해 계산한다. 그래서 CRC는 AG header에서는 sector 하나를, btree block에서는 block 하나를 덮는다. 검증할 때는 buffer를 고치지 않으려고 CRC field 앞, 0 네 byte, field 뒤를 이어 계산하고(`xfs_start_cksum_safe`), 갱신할 때는 field를 0으로 놓고 한 번에 계산한다.
- CRC field는 log하지 않는다. write verifier는 field 검사, LSN 기록, CRC 계산 순으로 한다(`xfs_agf_write_verify` 등). read verifier는 CRC를 먼저 보아 틀리면 -EFSBADCRC를, 그 뒤 field가 틀리면 -EFSCORRUPTED를 낸다.
- superblock만 예외가 있다. secondary의 CRC가 틀려도 v5로 알려진 file system이 아니면 실패로 치지 않고, mount 때 primary의 -EFSBADCRC는 -EFSCORRUPTED로 바뀐다.

AG 1 AGF의 CRC를 깨고 mount하자 `xfs_agf_read_verify`가 CRC error를 보고했고 mount가 실패했다. finobt, rmapbt, reflink가 켜진 이 file system에서는 mount 중에 per-AG reservation이 이 AGF를 읽는다. `xfs_buf_read_map`은 verifier가 남긴 -EFSBADCRC(-74)를 "metadata I/O error ... error 74" 줄로 찍고 -EFSCORRUPTED로 바꿔 돌려주므로, per-AG reservation은 "Error -117 reserving per-AG metadata reserve pool."을 찍었다.

code: xfs_cksum.h, xfs_buf.h, xfs_alloc.c, xfs_sb.c, xfs_buf.c

#### magic, UUID, 위치, owner

- magic: verifier마다 v4와 v5의 magic이 하나씩 있고, file system이 v5인지에 따라 하나를 고른다(`xfs_verify_magic`). v5에서 btree block의 magic이 바뀌어, 예를 들어 inobt는 "IABT"에서 "IAB3"이 됐다. AGI 1의 inobt root를 bnobt root인 block 1로 바꾸자 mount는 됐지만, bulkstat이 inobt를 읽을 때 `xfs_inobt_verify`가 bnobt의 magic "AB3B"를 잡았고, bulkstat은 EUCLEAN으로 실패했다.
- UUID: AG header와 btree block의 uuid가 `sb_meta_uuid`와 같은지 보아, 옛 mkfs의 잔재나 다른 LUN에 잘못 간 write를 잡는다. META_UUID incompat bit가 없으면 kernel은 in-core에서 `sb_meta_uuid`를 `sb_uuid`와 같게 둔다. v5에서 사용자가 UUID를 바꾸면 xfs_admin -U(xfs_db의 `uuid`)가 이 bit를 켜고 옛 UUID를 `sb_meta_uuid`에 남기므로, 모든 block을 다시 쓸 필요가 없다. 공통 예제 image에서는 superblock의 meta_uuid가 0이고 superblock, AGF, AGFL, AGI의 uuid가 모두 같았다.
- 위치: btree block에는 자기 disk 주소(`bb_blkno`)가 적혀 있고, verifier는 이 값이 실제로 읽은 buffer의 주소와 같은지 본다. AG header에는 주소 field가 없어서 seqno가 그 AG 번호와 같은지를 대신 본다. AGF와 AGI는 `xfs_validate_ag_length`에서 length도 함께 보고, AGFL은 `agfl_seqno`를 비교한다. growfs 중의 uncached buffer는 perag가 없어 seqno 검사를 건너뛴다.
- owner: owner 칸에는 short form btree block이면 32-bit AG 번호가, long form(bmbt 등)이면 64-bit inode 번호가 들어간다. short form은 verifier가 AG 번호와 비교하고, long form은 verifier가 보지 않고 btree lookup이 block을 읽을 때 cursor의 inode 번호와 비교한다. 공통 예제 image에서 AG 0 inobt block의 owner는 0, AG 1 bnobt root는 1, inode 263(`/S_IFREG.FMT_BTREE`)의 bmbt block은 263이었다.

1,025,000 KiB file system의 AG 1 bnobt root(AG 1의 block 1)에는 bno 512512(512 byte 단위)와 owner 1이 적혀 있다. 이 block이 AG 2 bnobt root 자리(AG 2의 block 1, bno 1025016)에 잘못 쓰였다면, CRC는 맞아도 verifier에 걸린다. bno가 읽은 주소와 다르고 owner가 AG 번호 2와 다르기 때문이다(관찰하지 않음).

- v5 field는 crc feature가 있을 때만 있다. v4 AGFL은 verifier가 아무것도 보지 않는다.
- AGFL verifier(`xfs_agfl_verify`)는 각 칸이 NULL이거나 AG 안의 번호인지만 보고 활성 구간과 대조하지 않는다. 칸을 꺼낼 때 다시 본다(`xfs_verify_agbno`).

code: xfs_buf.c, xfs_btree.c, xfs_alloc.c, xfs_ialloc.c, xfs_sb.c

#### LSN

LSN은 log 안의 위치를 64-bit 값 하나로 나타낸다. 위 32 bit가 log cycle, 아래 32 bit가 log 안의 basic block(512 byte) 번호다(`CYCLE_LSN`, `BLOCK_LSN`). xfs_db는 이 값을 16진 하나로 찍고, sysfs의 `log_head_lsn`에는 "cycle:basic block" 꼴로 나온다. 예를 들어 xfs_db의 0x10000001c는 cycle 1, basic block 28이라 sysfs 형식으로 1:28이다.

- 쓸 때는 write verifier가 buffer의 log item이 마지막으로 commit된 LSN(`bli_item.li_lsn`)을 넣는다.
- 읽을 때는 block의 LSN이 지금 log head보다 앞서면 손상으로 본다(`xfs_log_check_lsn`). "Corruption warning: Metadata has LSN (%d:%d) ahead of current LSN (%d:%d)"를 찍고, NULL LSN은 통과하며 norecovery mount는 검사하지 않는다. AGF, AGI, AGFL, btree block의 read verifier가 이 검사를 부른다. btree block에서는 short form과 long form 모두 이 검사가 CRC 검사보다 앞서서, LSN이 틀려도 CRC error로 찍힌다.
- superblock의 LSN은 log가 아직 없을 때 읽으므로 read verifier가 볼 수 없다. 대신 write verifier가 보고, mount 중에는 log tail을 찾은 직후 `xlog_recover`가 본다.
- log recovery는 block에 적힌 LSN이 지금 replay하는 transaction의 LSN 이상이면 그 buffer를 건너뛴다. block의 UUID가 이 file system의 것이 아니면 낡은 block으로 보고 그냥 replay하고, v4는 늘 바로 replay한다. block에 LSN을 매기는 log의 동작은 [log](12-log.md)에 있다.

새 1 GiB file system에 directory 8개와 64 KiB file 8개를 만들고 sync한 뒤 freeze하면 LSN은 이렇게 바뀌었다.

| 시점 | sysfs head, tail | superblock | AG 0과 AG 1의 AGF, AGI, AG 0 inobt root | AG 0 AGFL |
|---|---|---|---|---|
| mkfs 직후 |  | 0 | 0(AG 0의 AGF, AGI만 읽음) | 0 |
| sync 뒤 | 1:26, 1:2 |  |  |  |
| freeze 뒤 | 1:30, 1:28 | 1:28 | 1:2 | 0 |

freeze 뒤 superblock의 LSN은 log_tail_lsn과 같았다.

code: xfs_log_format.h, xfs_log.c, xfs_btree.c, xfs_buf_item_recover.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs_format.h의 `XFS_AGF_MAGIC` 위 주석 | AG header가 "sequential 512-byte buffers" | code는 sector 크기(`m_sectbb_log`)를 쓴다. 4096 byte sector image에서 header 하나는 4096 byte였다 |
| xfs-documentation allocation_groups.asciidoc | AGFL 128칸, root가 "blocks 4 to 7" | v4 배치다. v5는 header 36 byte를 빼서 512 byte sector에 119칸이고, root block 번호도 feature에 따라 달라진다. superblock.asciidoc와 allocation_groups.asciidoc의 예제 출력도 v4 출력이다 |
| allocation_groups.asciidoc | superblock의 lazy counter가 clean unmount 때만 갱신된다 | superblock을 log할 때마다 percpu 합을 넣는다(`xfs_log_sb`) |
| Documentation/filesystems/xfs/xfs-self-describing-metadata.rst | 예제 code의 `xfs_sb_version_hascrc` | 기준 tree에는 없고, 지금 code에서는 `xfs_has_crc()`와 `xfs_buf_verify_cksum()`을 쓴다 |

## 한계점

다음은 확인하지 못했다.

- 1,025,000 KiB file system을 만들 때 mkfs가 AG 수 계산의 어느 갈래를 탔는지.
- mkfs가 AG 0의 root inode chunk를 어느 block에 두는지. 관찰 출력에 block 번호가 없다.
- AG 2에서 log가 끝나는 block 16390과 free extent가 시작하는 block 16397 사이의 6 block. AG 1처럼 AGFL이 가져간 것으로 보이지만 AG 2의 AGFL은 찍지 않았다.
- 모르는 log_incompat bit가 있고 log가 clean할 때 ro mount와 ro → rw remount의 결과. rw mount만 돌렸다.
- bnobt root의 owner와 cntbt root의 bno를 바꾼 두 시험에서 어느 검사가 잡았는지. 두 시험의 kernel log가 출력에 남지 않았다.
- AGF의 CRC를 깬 뒤 xfs_db가 같은 block의 crc를 (correct)로 찍은 이유. xfs_db는 시작할 때 `libxfs_initialize_perag_data`로 모든 AG의 AGF와 AGI를 읽는다. 이때 verifier에서 실패한 buffer는 `libxfs_buf_relse`로 cache에 남는다(UPTODATE). 다음 read는 cache hit에서 `b_error`를 0으로 지우고 verifier를 다시 돌리지 않는다. `(correct)`는 `b_error`가 `-EFSBADCRC`가 아니면 붙으므로, 틀린 CRC에도 붙는다(xfsprogs v7.1.1 `db/init.c`, `libxfs/rdwr.c`, `db/io.h`, code로만 확인). 시작할 때 읽지 않는 block은 `(bad)`로 찍힌다. zoned 관찰의 rtsb가 그 예다.
- block의 LSN이 log head보다 앞설 때의 경고.
- logged xattr bit를 실제로 모르는 옛 kernel로 옮긴 disk. 모르는 bit(0x80000000)로만 흉내 냈다.

## 실험 환경

공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 1,025,000 KiB file system(4 KiB block 256250개)은 AG 크기가 나누어떨어지지 않는 경우를 보려고 만들었다. option 없이 mkfs했고, mkfs 출력대로 512 byte sector, crc=1, finobt=1, sparse=1, rmapbt=1, reflink=1, bigtime=1, inobtcount=1, nrext64=1, exchange=1, parent=1, metadir=0으로 만들어졌다. 비회전으로 보이는 loop device에 있었지만 4 GiB보다 작아서, mkfs가 CPU 수 갈래를 탔더라도 기본 계산의 4개로 돌아온다.
- mkfs, xfs_db, xfs_admin, xfs_repair의 동작(AG 수 계산, secondary 초기값, log 자리, AGFL 채움, label과 UUID 변경, 깨진 secondary 고쳐 쓰기, `version`의 해석)은 xfsprogs v7.1.1의 mkfs/xfs_mkfs.c, libxfs/topology.c, db/sb.c, db/xfs_admin.sh, io/label.c, repair/agheader.c에서 읽었다.
- 4096 byte sector는 1 GiB에 `mkfs.xfs -s size=4096`으로 따로 만들었다.
- LSN, 모르는 feature bit, 손상 시험은 option 없이 만든 1 GiB file system에서 했다. log_incompat bit의 수명만 debug image에서 보았다.
- 손상 주입: unmount한 image에 `xfs_db -x`로 썼다. `write -d`는 write verifier를 건너뛰고 CRC를 다시 계산해 쓴다(출력의 "Allowing write of corrupted data with good CRC"). `crc -i`는 CRC를 틀린 값으로 쓴다. 모르는 feature bit는 fstests xfs/270처럼 mask의 가장 높은 bit(0x80000000)를 켜서 흉내 냈다.
- mount option은 기본값이고, 시험에 따라 `ro`, `remount,rw`, `ro,norecovery`를 더했다.

### 관찰 목록

| 관찰 | 절차 | 본문의 예 |
|---|---|---|
| AG 배치 | 1,025,000 KiB file system을 만들고 xfs_db로 sb, AGF, AGI, AGFL의 field와 `stack`, AG 1의 `convert`, AG 1과 AG 2의 bnobt root를 읽음 | AG로 나누기, AG 1의 앞, AG마다 다른 것, 위치 검사의 가정 예 |
| 4096 byte sector | 같은 읽기를 4096 byte sector image에서 | sector 크기 표, AGFL 칸 수 |
| superblock | 공통 예제 image의 `sb 0` 전체와 `version`, `sb 1`과 `sb 3`의 counter, lsn, crc, feature mask | superblock 표, primary와 secondary 표 |
| AG header | 공통 예제 image의 AG 0과 AG 1의 AGF, AGI, AGFL 전체. AGF의 flfirst와 fllast로 AGFL 칸을 다시 읽음 | AGF, AGI 표, AGFL 칸 표 |
| self-describing field | 공통 예제 image의 sb, AG header, inobt node와 leaf, AG 1 bnobt root, inode 263의 bmbt block에서 uuid, bno, owner, lsn, crc와 `crc -v` | UUID, owner |
| LSN | mkfs 직후 header LSN, directory 8개와 64 KiB file 8개를 만들고 sync한 뒤와 freeze 중의 sysfs log LSN, freeze 중 header LSN | LSN 표 |
| 모르는 feature bit | 기본 mkfs image의 mask에 0x80000000을 더했다. compat은 rw mount, file 만들기, unmount, ro_compat은 rw mount, ro mount, `remount,rw`, unmount, incompat은 ro mount와 rw mount 순으로 실행했다. 방법은 fstests xfs/270에서 가져왔다 | mount 경로 표, compat, ro_compat, incompat |
| 모르는 log_incompat bit | clean log는 갓 만든 image에 bit를 켜고 rw mount, file 만들기, unmount. dirty log는 mount한 file system에 64 KiB를 쓰고 `xfs_io -x -c "shutdown -f"`로 log를 dirty하게 둔 채 unmount한 뒤 bit를 켜고 rw, ro, `ro,norecovery` 순으로 mount | log_incompat |
| log_incompat bit의 수명 | debug image에서 `larp`를 0에서 1로 바꾸고 mount, file 만들기, `setfattr`. freeze 중과 unfreeze, unmount 뒤에 superblock을 읽음 | log_incompat bit의 수명 |
| 손상: CRC, UUID, seqno | AGF 1에 `crc -i`, AGI 2의 uuid와 seqno에 `write -d`. 각각 xfs_db로 다시 읽고 mount | CRC, field 표 |
| 손상: magic, owner, bno | AGI 1의 root를 3에서 1로, AG 0 bnobt root의 owner를 0에서 1로, AG 0 cntbt root의 bno를 16에서 8(bnobt root의 주소)로 바꾸고 mount. 뒤이어 `bulkstat -a 1`이나 4 MiB write와 fsync | magic, field 표 |

### 출력 읽는 법

- 출력의 `*-rc=`는 명령의 종료 코드다. mount의 32는 실패를 뜻한다.
- AG 배치의 첫 발췌에 나오는 stack은 `sb 1`의 것이고, `sb 0`과 AG 0 header의 stack은 둘째 발췌에 있다.
- xfs_db의 `version`은 superblock에서 계산한 mount 시점의 feature를 V5, CRC, FINOBT, RMAPBT, REFLINK, PARENT 같은 이름으로 찍는다. SHARED, DIRV2, MOREBITS는 feature가 아니라 versionnum bit를 직접 보고, v3 inode, project quota inode, realtime, zoned는 찍지 않는다. 괄호 안 두 값만 versionnum과 features2다.
- kernel은 verifier 보고의 `%pS` 자리에 verifier 함수와 offset을 찍는다(예: `xfs_agf_read_verify+0x80/0x110`). xfs_db도 libxfs의 verifier로 비슷한 문구를 찍지만, 그 자리에 함수 이름 대신 주소가 나오고 block 번호 뒤에 `/0x200`이 붙는다. 손상 시험의 block 번호는 512 byte 단위라서, 0x80001은 AG 1의 sector 1이다.
- metadata I/O error 줄의 `%pS` 자리에는 buffer 읽기를 부른 함수(예: `xfs_read_agf+0x98/0x160`)가 찍히고, error 뒤의 수는 errno의 부호를 뗀 값이다.
- shutdown을 알린 줄은 source 위치가 들어 있어 발췌에서 뺐다.
- log_incompat bit의 수명 출력에서 처음 두 줄은 `larp`의 처음 값 0과 바꾼 값 1이고, print가 낸 네 줄 뒤의 0은 `larp`를 0으로 되돌린 줄이다.

### 원본 출력

#### AG 배치

(원본 출력 "1,025,000 KiB image의 mkfs 출력과 AG 크기": `raw/04-on-disk-layout.md`)

(원본 출력 "512 byte sector image: AG 0 header와 root block 자리": `raw/04-on-disk-layout.md`)

(원본 출력 "AG 1의 AGFL과 AG 1, AG 2의 bnobt record": `raw/04-on-disk-layout.md`)

#### 4096 byte sector

(원본 출력 "4096 byte sector image: AG 0 header와 root block 자리": `raw/04-on-disk-layout.md`)

(원본 출력 "4096 byte sector image의 AG 0 AGFL": `raw/04-on-disk-layout.md`)

#### superblock

(원본 출력 "공통 예제 image의 primary superblock 전체": `raw/04-on-disk-layout.md`)

(원본 출력 "공통 예제 image의 secondary superblock (sb 1, sb 3)": `raw/04-on-disk-layout.md`)

#### AG header

(원본 출력 "공통 예제 image의 AGF 0과 AGF 1": `raw/04-on-disk-layout.md`)

(원본 출력 "공통 예제 image의 AGI 0과 AGI 1": `raw/04-on-disk-layout.md`)

(원본 출력 "공통 예제 image의 AG 0 AGFL": `raw/04-on-disk-layout.md`)

#### self-describing field

(원본 출력 "공통 예제 image의 superblock과 AG header uuid": `raw/04-on-disk-layout.md`)

(원본 출력 "공통 예제 image의 btree block 주소와 owner": `raw/04-on-disk-layout.md`)

(원본 출력 "공통 예제 image의 bmbt file inode와 bmbt block owner": `raw/04-on-disk-layout.md`)

#### LSN

(원본 출력 "mkfs 직후와 freeze 중의 header LSN과 sysfs log LSN": `raw/04-on-disk-layout.md`)

#### 모르는 feature bit

(원본 출력 "모르는 compat bit를 켠 mount": `raw/04-on-disk-layout.md`)

(원본 출력 "모르는 ro_compat bit를 켠 mount와 remount": `raw/04-on-disk-layout.md`)

(원본 출력 "모르는 incompat bit를 켠 mount": `raw/04-on-disk-layout.md`)

#### 모르는 log_incompat bit

(원본 출력 "모르는 log_incompat bit를 켠 mount (clean log)": `raw/04-on-disk-layout.md`)

(원본 출력 "모르는 log_incompat bit를 켠 mount (dirty log)": `raw/04-on-disk-layout.md`)

#### log_incompat bit의 수명

(원본 출력 "logged xattr의 log_incompat bit: freeze 중과 unmount 뒤": `raw/04-on-disk-layout.md`)

#### 손상: CRC, UUID, seqno

(원본 출력 "AGF 1의 CRC를 깬 mount": `raw/04-on-disk-layout.md`)

(원본 출력 "AGI 2의 uuid를 바꾼 mount": `raw/04-on-disk-layout.md`)

(원본 출력 "AGI 2의 seqno를 바꾼 mount": `raw/04-on-disk-layout.md`)

#### 손상: magic, owner, bno

(원본 출력 "AGI 1의 inobt root를 bnobt root로 바꾼 bulkstat": `raw/04-on-disk-layout.md`)

(원본 출력 "AG 0 bnobt root의 owner를 바꾼 write와 fsync": `raw/04-on-disk-layout.md`)

(원본 출력 "AG 0 cntbt root의 bno를 바꾼 write와 fsync": `raw/04-on-disk-layout.md`)
