# XFS의 설계와 역사

> XFS 전반 wiki, 묶음 "기초". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/01-overview-history.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/01-overview-history.md, 접힌 원본 출력은 raw/01-overview-history.md.

## 개요

- XFS는 SGI가 1993년 무렵 IRIX용으로 설계했고, 1994년 12월 IRIX 5.3에서 처음 나왔다. 1996년 논문에서 세운 목표는 terabyte 단위의 disk, 아주 큰 file, 초당 수백 MB의 I/O였다. 그 답으로 고른 64-bit 주소, AG(allocation group), extent와 B+tree, metadata log, delalloc(delayed allocation)은 지금 code에도 그대로 남아 있다. 예를 들어 AG가 4개인 file system을 mount하고 directory 8개를 처음 만들면 AG 0, 1, 2, 3, 0, 1, 2, 3에 차례로 놓인다.
- v5에서는 metadata block마다 CRC, UUID, owner(그 block이 속한 inode나 AG), LSN이 더 붙어서 block 하나만 읽어도 그 block을 검사할 수 있다(self-describing metadata). 규모가 커지자 가장 어려운 문제가 algorithm이 아니라 구조의 검증이 되었기 때문이다. v5가 나온 뒤에 생긴 on-disk 기능은 ftype 하나를 빼면 모두 v5에서만 쓸 수 있다. v4 지원은 2030년 9월에 kernel에서 지울 예정이다.
- 2010년대 이후의 큰 기능은 delayed logging(2.6.35), rmapbt(4.8)와 reflink(4.9), online fsck(scrub 4.15부터), parent pointer(6.10), rtgroup(6.13), zoned(6.15) 순서로 들어왔다. 뒤에 들어온 기능 가운데 몇은 앞의 기능이 있어야 쓸 수 있다. online repair에는 대부분 rmapbt가 있어야 한다. zoned에는 metadir, rtgroup, rmapbt가 있어야 하고, reflink와는 함께 쓸 수 없다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었고, zoned의 예만 zone 16개짜리 device를 realtime device(rt device)로 붙인 file system에서 얻었다.

## 설계 목표와 AG

이 장에서 지금 code와 맞춘 대응은 code로 확인했고, 예의 값과 "(관찰)"을 단 칸은 관찰한 것이다.

### 1996년 논문의 목표

- SGI의 이전 file system인 EFS는 file system 8GB, file 2GB가 한계였고, hardware의 I/O 대역을 application에 다 주지 못했다. 동기가 된 workload는 압축하지 않은 video(stream 하나가 30MB/s, 1시간이면 108GB)와 video on demand server였다.
- 논문은 기존 file system의 한계로 여섯 가지를 꼽았다. crash 뒤 fsck 시간이 file system 크기에 비례하고, 32-bit block pointer와 bitmap 때문에 큰 file system을 만들 수 없고, 64-bit sparse file을 만들 수 없다. 또 bitmap에서 큰 연속 공간을 찾기가 느리고, 큰 directory를 선형으로 탐색하고, inode 수를 mkfs 때 고정한다.
- 논문에 따르면 AG는 locality가 아니라 scalability와 parallelism을 위해 쓴다("AGs are used for scalability and parallelism rather than disk locality"). AG마다 구조가 따로라 free space와 inode 관리가 AG마다 병렬로 진행되고, 중앙 자원은 transaction log 하나뿐이다.
- 그렇다고 locality를 버리지는 않았다. 논문에 따르면 AG는 locality를 맡기에는 너무 커서, XFS는 대신 file과 directory 둘레에 locality를 만든다. 새 directory는 parent와 다른 AG에 두고, 그 안의 inode와 data block은 directory 가까이에 모으고, 큰 file의 extent는 inode 가까이에서 시작한다.
- code에서는 file inode가 parent의 AG에서 찾기 시작하고, data는 inode 가까이에서, file 끝에 이어 쓰면 앞 extent 바로 뒤에서 찾는다(filestream inode는 따로 고른다). 새 directory는 논문과 달리 parent의 AG를 보지 않고 AG를 돌아가며 놓인다.
- 논문은 "think big" 한 마디로 끝맺는다. 논문 때는 IRIX 6.2부터 XFS를 기본 file system으로 할 예정이었다.

### 논문의 결정이 남은 자리

| 논문의 결정 | 지금 code |
|---|---|
| 모든 global counter, block 주소, inode 번호는 64-bit. AG 안의 구조는 AG 기준 32-bit pointer | AG 안의 block 번호와 inode 번호(`xfs_agblock_t`, `xfs_agino_t`)는 32-bit이고, file system 전체의 block 번호와 inode 번호는 64-bit다. 전체 block 번호도 inode 번호처럼 위쪽 bit가 AG 번호다(주석 "agno\|agbno"). AG btree block의 sibling pointer와 record(free space, rmap)는 32-bit AG block 번호를, inode에 root를 둔 btree는 64-bit pointer를 쓴다 |
| inode 번호에 AG 번호가 들어 있다("sparse inode numbering space") | XFS에서는 inode 번호의 위쪽 bit에 AG 번호가 들어간다(`XFS_INO_TO_AGNO`). 그래서 번호가 빈틈없이 이어지지 않고, 32-bit 번호가 필요하면 `inode32`로 inode를 앞쪽 AG에만 둔다. (관찰) AG 1~3의 첫 directory가 524416, 1179776, 1572992다 |
| AG 크기에 상한을 두어 AG 안의 구조를 작게 | AG 하나는 16MB에서 1TB 사이여야 하고, 벗어나면 superblock 검사에서 mount가 거부된다(`XFS_MAX_AG_BYTES`). 논문 때는 보통 0.5~4GB였다. (관찰) 1 GiB file system에서 AG 4개, 각 65536 block(256 MiB) |
| free space는 AG마다 B+tree 한 쌍(시작 block 순, 길이 순) | bnobt와 cntbt가 같은 record `xfs_alloc_rec`를 쓴다 |
| file은 extent map이고, inode에 다 들어가지 않으면 inode에 root를 둔 B+tree로 바꾼다. extent 하나가 약 200만 block | 길이 field가 21 bit라 extent 하나가 최대 2,097,151 block(2^21 - 1)이다. 4 KiB block이면 약 8 GiB다. file offset은 54 bit다 |
| inode는 64개 묶음으로 동적으로 할당하고 AG마다 B+tree로 찾는다 | `XFS_INODES_PER_CHUNK`가 64이고 inobt record의 free mask가 64-bit다. chunk의 일부만 할당하는 sparse inode는 나중에 붙었다 |
| directory는 이름을 4 byte hash로 바꾼 B+tree | hash type `xfs_dahash_t`가 32-bit다. directory의 btree는 다른 btree와 다른 code와 block header를 쓴다. 논문에서도 directory B+tree만 조금 다르다고 했다 |
| 새 directory는 parent와 다른 AG에 | directory를 만들 때는 mount마다 하나 있는 순번(rotor)을 하나씩 올려 AG를 돌아가며 고르고, 다른 inode는 parent의 AG에서 시작한다(`xfs_dialloc_pick_ag`). (관찰) d1~d8이 AG 0, 1, 2, 3, 0, 1, 2, 3에 놓였다 |
| AG마다 구조가 따로라 병렬로 할당 | AG마다 in-core 사본인 `xfs_perag`가 있다. AGF lock과 in-core inode cache도 AG마다 따로 있다 |
| 중앙 자원은 log 하나. metadata만 write ahead log로 비동기로 기록하고, log는 따로 둔 device에 둘 수 있다 | mount마다 log가 하나 있다(`m_log`). mount에는 data device, log device, rt device가 따로 있다. log 안의 병렬화는 나중에 들어온 delayed logging의 CIL이 맡는다 |
| block 할당을 미루는 delalloc | buffered write는 보통 `xfs_bmapi_reserve_delalloc`으로 block을 예약만 한다. delalloc extent에는 진짜 start block이 없고, block 위치는 나중에 정한다 |
| 한 file에 여러 process의 I/O | 정렬된 direct I/O write는 IOLOCK을 shared로, buffered write는 exclusive로 잡는다. 논문에 따르면 buffered write의 single writer 제한은 구현 탓이라 언젠가 없앨 예정이었지만, 기준 tree에서도 exclusive다 |
| realtime device(논문의 video 동기) | Kconfig 도움말에 따르면 realtime device는 원래 media streaming에 맞는 예측할 수 있는 data rate를 주려고 설계되었다. 논문 본문에는 real-time subvolume 절이 없다 |

XFS는 free space, inode, extent map, directory를 모두 B+tree로 찾는다. 앞의 셋이 함께 쓰는 block 구조는 [B+tree 공통 구조](05-btree.md)에, free space B+tree 한 쌍과 delalloc 예약은 [free space 구조와 할당 정책](09-free-space.md)에 있다.

AG의 병렬성은 block 할당에서 잘 드러난다. 할당은 AG를 차례로 돌며 AGF(AG의 free space header, [on-disk 배치](04-on-disk-layout.md))의 lock을 trylock으로 잡아 보고, 다른 thread가 잡고 있는 AG는 건너뛴다. 예를 들어 thread A가 AG 0의 AGF를 잡고 할당하는 동안 thread B가 AG 0부터 찾는다면, B는 AG 0을 건너뛰고 AG 1로 간다(관찰하지 않음). 한 바퀴를 돌아도 쓸 AG가 없으면(잠겼거나 공간이 모자라서) 그때만 lock을 기다리며 다시 돈다. 이 순회는 `xfs_alloc_vextent_iterate_ags`가 한다.

code: xfs_types.h, xfs_format.h, xfs_fs.h, xfs_sb.c, xfs_super.c, xfs_ialloc.c, xfs_ag.h, xfs_alloc.c, xfs_mount.h, xfs_iomap.c, xfs_file.c, Kconfig

### 새 directory와 AG

새 directory는 AG를 돌아가며 놓이고, directory 안의 file은 그 directory의 AG에서 먼저 자리를 찾는다. AG가 4개인 file system의 root 아래에 directory d1~d8을 차례로 만들고 각각에 file 하나를 만들면 다음과 같이 놓인다.

| AG | directory(inode 번호) | 그 안의 file |
|---|---|---|
| 0 | root(128), d1(131), d5(133) | 132, 134 |
| 1 | d2(524416), d6(524418) | 524417, 524419 |
| 2 | d3(1179776), d7(1179778) | 1179777, 1179779 |
| 3 | d4(1572992), d8(1572994) | 1572993, 1572995 |

inode 번호의 위쪽 bit가 AG 번호라서 AG가 바뀌면 번호가 크게 뛴다. 번호를 AG와 block으로 푸는 법은 [XFS 전반](00-start.md)에 있다.

d1과 d5는 parent인 root와 같은 AG 0에 놓였다. rotor(`m_agirotor`)가 parent의 AG를 보지 않고 mount 때 0에서 시작하므로, 첫 directory와 한 바퀴 돈 다섯째 directory가 root와 같은 AG에 들어간다.

### 같은 생각을 다시 쓴 곳

- 지금 code에서는 AG의 공통 부분이 `struct xfs_group`으로 빠져 AG와 rtgroup이 함께 쓴다. 주석에 따르면 rtgroup은 per-AG 구조와 비슷하다.
- online fsck 설계 문서에 따르면 공간을 AG로 나눈 덕에 병렬성이 큰 system에서 성능이 좋아지고 손상의 피해를 가둘 수 있다. 검사와 repair도 AG 하나처럼 file system의 일부만 겨눠 할 수 있다.
- 관리 문서에 따르면 XFS는 metadata 갱신을 workqueue로 병렬화한다.

code: xfs_group.h, xfs_rtgroup.h, xfs_format.h

### SGI의 흔적

- kernel은 지금도 boot 때 이름을 "SGI XFS"로 찍는다(`XFS_VERSION_STRING`). 관찰 image의 banner는 "SGI XFS with ACLs, security attributes, realtime, scrub, repair, quota, no debug enabled"이다. 뒤에 붙는 build option 문자열은 Kconfig에 따라 달라진다.
- 관리 문서의 제목도 "The SGI XFS Filesystem"이고, 첫 문단에는 XFS가 SGI IRIX에서 나왔고 완전히 multi-thread로 동작한다고 적혀 있다.
- superblock 판 번호의 주석에는 옛 release 번호가 달려 있다(v1에 "5.3, 6.0.1, 6.1", v2에 "6.2 - attributes", v4에 "6.2+ - bitmask version"). 논문의 IRIX 5.3, 6.2와 숫자가 맞아 IRIX release로 보인다(추정).
- fs/xfs와 libxfs의 .c, .h file 245개 가운데 144개에 "Silicon Graphics" copyright가 있다.

code: xfs_super.h, xfs_super.c, xfs_format.h

## v4와 v5

이 장의 동작은 code와 문서로 확인했고, 예의 값은 관찰한 것이다.

### 판 번호

superblock `sb_versionnum`의 아래 4 bit가 판 번호이고, kernel은 v4와 v5만 받는다. option 없이 만든 file system은 `versionnum = 0xb4b5`로 아래 4 bit가 5이고, mount하면 kernel log에 "Mounting V5 Filesystem"이 찍힌다.

- `XFS_SB_VERSION_5`의 주석에 따르면 v5는 CRC를 켠 file system이다. v5인지는 `xfs_sb_is_v5`로 가린다.
- `xfs_sb_good_version`은 v5면 feature 검사로 넘기고, v4가 아니면 거부한다. v4도 v2 directory와 unwritten extent flag가 있어야 받는다.

code: xfs_format.h, xfs_sb.c, xfs_log.c

### v5에서 달라진 on-disk

v5는 superblock에 feature mask와 검사용 field를, btree block header와 inode에 검사용 field를 더하고 btree block의 magic도 바꿨다. 그 배치와 검사가 도는 방식은 [on-disk 배치](04-on-disk-layout.md)에 있다.

| 자리 | v5에서 더한 것 |
|---|---|
| superblock | feature mask 넷(`sb_features_compat`, `sb_features_ro_compat`, `sb_features_incompat`, `sb_features_log_incompat`), `sb_crc`, `sb_lsn`, `sb_meta_uuid`. 그 뒤에 metadir와 rtgroup의 field가 온다 |
| btree block header | blkno, lsn, uuid, owner, crc. 주석에 따르면 crc feature bit가 있는 file system에만 있다 |
| inode | inode version 3. core 뒤에 `di_crc`, `di_lsn`, `di_ino`, `di_uuid`가 든 확장 부분이 붙는다 |
| magic | bnobt는 v4가 `ABTB`, v5가 `AB3B`다. v5 뒤에 생긴 rmapbt와 refcountbt에는 CRC magic만 있다. finobt는 3.16에 들어왔지만 CRC 없는 magic도 정의되어 있다 |

code: xfs_format.h, xfs_sb.c

### v5로 간 이유

self-describing metadata 설계 문서에 따르면 위치가 고정된 metadata는 AG header뿐이라 나머지는 구조를 따라가야 찾을 수 있다. 사람이 손으로 하는 분석이 100TB에서는 몇 시간에서 며칠 걸리고, 1PB에서는 몇 주에서 몇 달 걸린다.

- 그래서 metadata block이 스스로를 설명하게 했다. CRC는 metadata block이 64k 이하라 32-bit인 CRC32c로 충분하다. 엉뚱한 LUN에 쓴 write를 잡으려고 자기 위치를 block 번호와 file system 식별자로 함께 적는다.
- 관리 문서와 Kconfig 도움말에 따르면 v4에는 metadata checksum, 더 강한 metadata 검증, 2038년 뒤의 timestamp가 없다. 옛 timestamp의 상한은 2038-01-19다. v5의 incompat bit인 bigtime을 켜면 2486년까지 쓸 수 있다.

### 새 기능은 v5에만

v5 뒤에 생긴 on-disk 기능은 v5 superblock의 feature mask에만 bit가 있다. 주석에도 이 mask는 새 v5 기능에만 쓴다고 적혀 있다.

- ro_compat에는 finobt, rmapbt, reflink, inobtcount 넷이, incompat에는 ftype부터 zone gap까지 11개가, log_incompat에는 logged xattr 하나가 있다. compat에는 정의된 bit가 없다.
- kernel은 v4 superblock에서 이 mask로 feature를 정하지 않는다. `xfs_sb_version_to_features`는 v4면 v4 bit만 보고, v5면 ALIGN, LOGV2, EXTFLG, LAZYSBCOUNT, PROJID32, V3INODES, CRC, PQUOTINO를 늘 켜고 나머지를 mask에서 읽는다.
- mkfs도 같다. crc=0이면 v4가 폐기 예정이라는 경고를 찍고 v5 뒤의 기능(finobt, sparse inode, rmapbt, reflink, inobtcount, bigtime, nrext64, exchange-range, parent pointer, autofsck, metadir)을 끈다. 사용자가 그중 하나나 persistent quota flag를 주면 "... not supported without CRC support"로 멈춘다.
- ftype만 예외다. v4 superblock의 features2에도 bit가 있어서, kernel은 v4에서도 이 bit로 ftype을 켜고 mkfs도 crc=0에서 ftype을 만든다.

option 없이 만든 file system의 feature mask는 다음과 같다.

| mask | 값 | 켜진 것 |
|---|---|---|
| compat | 0 | 정의된 bit가 없다 |
| ro_compat | 0xf | ro_compat의 네 기능 finobt(1), rmapbt(2), reflink(4), inobtcount(8)이 모두 켜졌다. 합은 15다 |
| incompat | 0xeb | 켜진 bit는 ftype(1), sparse inode(2), bigtime(8), nrext64(32), exchange-range(64), parent pointer(128)이고 합이 235다. meta_uuid(4)와 needsrepair(16)는 꺼져 있다 |
| log_incompat | 0 |  |

code: xfs_format.h, xfs_sb.c

### 기본값과 폐기 일정

mkfs는 2015년부터 v5를 기본으로 만들고, 기준 kernel은 v4 지원을 기본으로 끈다.

| 시점 | 일 | 근거 |
|---|---|---|
| kernel 3.10 | v5가 "(for now) experimental"로 들어왔다 | pull request mail |
| xfsprogs v3.2.3-rc1 | mkfs가 CRC(v5)와 finobt를 기본으로 켰다. 그 기능이 배포판 kernel에 들어갈 만큼 오래되었기 때문이다 | xfsprogs `566ebd5a`(Dave Chinner, 2015-05-11) |
| kernel 5.10 | v4 폐기를 알렸다 | pull request mail |
| xfsprogs v5.14.0-rc1 | mkfs가 새 v4 file system을 만들 때 경고한다 | xfsprogs `fc239bcb`(2021-11-12) |
| 2025년 9월 | `XFS_SUPPORT_V4`가 `default n`이 되었다 | Kconfig |
| 2030년 9월 | v4 지원을 kernel에서 완전히 지울 예정이다. ascii-ci도 같은 일정이다 | Kconfig, 관리 문서의 표 |

- v4 지원을 끈 kernel은 v4 mount를 거부한다. `xfs_fs_fill_super`가 "Deprecated V4 format (crc=0) not supported by kernel."을 찍고 -EINVAL을 돌려준다. 켠 kernel은 "will not be supported after September 2030"을 한 번 찍고 진행한다(code로만 확인).
- v4 code는 아직 tree에 있다. v4 지원을 끄면 v5에서 늘 켜진 feature(align, logv2, extflg, lazysbcount, projid32, v3inodes, crc, pquotino)의 검사 함수가 상수 true가 되어, compiler가 v4 분기를 지운다(`__XFS_HAS_V4_FEAT`).
- v4인지는 `xfs_info` 출력에서 `crc=`를 찾아 본다. Kconfig 도움말에 따르면 "crc=0"이 있으면 v4다.

code: Kconfig, xfs_super.c, xfs_mount.h

## 2010년대 이후의 큰 기능

이 장의 동작은 code와 문서로 확인했고, mkfs와 mount의 값은 관찰한 것이다.

| 기능 | mainline | 풀려던 문제 | 기준 kernel에서의 상태 |
|---|---|---|---|
| delayed logging | 2.6.35 실험(`-o delaylog`), 2.6.39 기본 | 같은 object를 transaction마다 다시 log하면(relog) log buffer에 낡은 사본이 쌓인다. 낡은 사본을 log에 쓰지 않는다 | 늘 켜져 있다. mount option이 없다 |
| rmapbt | 4.8 실험, 4.16 실험 해제 | 원래 설계에 없던 중복을 더한다. 할당된 extent에서 owner를 찾는 역방향 index | mkfs 기본으로 켜진다(xfsprogs v6.5.0부터) |
| reflink | 4.9 실험, 4.16 실험 해제 | 두 file이 물리 block을 나눠 가지면서도 보통의 file 의미를 지킨다. 공유 block에 쓸 때는 COW(copy on write)를 한다 | mkfs 기본으로 켜진다(xfsprogs v5.1.0부터) |
| online fsck | scrub 4.15, repair 4.18(제한), 6.8~6.10 확대, 6.16 실험 경고 제거, 6.18 Kconfig 기본 y | 깨진 metadata로 인한 갑작스러운 shutdown, 복구와 검사 동안의 서비스 중단, 운영 중에 건강 상태를 모르는 것 | Kconfig 기본 y |
| parent pointer | 6.10, 6.16 실험 경고 제거 | root부터 거치지 않고 child에서 parent를 찾는다. directory tree 전체를 다시 지을 수 있게 한다 | mkfs 기본으로 켜진다(xfsprogs v6.18.0부터) |
| rtgroup | 6.13 (rt rmap과 rt reflink는 6.14) | rt section을 따로 잠글 수 있는 조각으로 나눈다 | metadir를 따른다. mkfs 기본은 꺼짐 |
| zoned | 6.15, 실험 표시 제거는 7.2(기준 이후) | zoned block device를 rt allocator로 쓴다 | mount 때 아직 EXPERIMENTAL 경고를 찍는다 |

같은 시기에 finobt(3.16), sparse inode(4.2), inobtcount와 bigtime(5.10), nrext64와 logged xattr(5.19), exchange-range(6.10), metadir(6.13) 같은 on-disk 기능도 들어왔다. 기능에 따라서는 다음이 있어야 쓸 수 있다.

| 기능 | 있어야 하는 것 | 함께 쓸 수 없는 것 |
|---|---|---|
| online repair | 대부분 rmapbt. directory, xattr, symlink처럼 내용을 새로 지어 바꾸는 repair는 exchange-range도 |  |
| directory tree 전체 재구성 | parent pointer |  |
| software atomic write | reflink(`xfs_can_sw_atomic_write`) |  |
| rtgroup | metadir. rtgroup에는 자기 feature bit가 없다 |  |
| zoned | rt device, metadir, rtgroup, rmapbt | reflink |

### delayed logging

delayed logging에서는 transaction이 commit한 log item을 log buffer 밖의 CIL(committed item list)에 모아 두어, 같은 object의 낡은 사본을 log에 쓰지 않는다. log 쪽 동작은 [log](12-log.md)에 있다.

같은 directory block을 transaction A~D 넷이 차례로 고치고 넷이 같은 log buffer에 들어간다면, 예전 방식에서는 사본 넷이 다 쓰이고 마지막 D만 쓸모가 있다. 설계 문서에 따르면 이런 낡은 사본이 log buffer 공간의 90%를 넘을 수 있었고, 이를 줄이는 것이 delayed logging의 근본 목표였다.

- 그 전에는 log에 아직 쓰이지 않은 변경이 log buffer 크기에 묶여 최대 2MB였다. 기준 tree의 값으로는 iclog(in-core log buffer) 8개(`XLOG_MAX_ICLOGS`)에 최대 256KiB씩, 모두 2MiB다. delayed logging은 이 묶임을 풀었고, 문서에 따르면 그만큼 crash 때 잃을 수 있는 변경이 커진다.
- CIL에는 CPU마다 따로 쌓는 부분(`xlog_cil_pcp`)이 있고, flush와 commit은 rw semaphore `xc_ctx_lock`으로 가른다. CIL이 log 크기의 1/8과 32MiB 가운데 작은 쪽만큼 차면 background push를 시작한다. iclog 하나의 실제 크기는 mount option logbsize로 정하고, 주지 않으면 32KiB와 log stripe unit 가운데 큰 쪽이다. 32MiB는 iclog 8개의 최대 크기 합 2MiB를 16배 한 값이라 logbsize와 상관없다(code로만 확인).
- 설계 문서에 따르면 CIL의 commit 경로는 CPU가 2048개인 system에서처럼 commit이 한꺼번에 많이 들어와도 견디게 설계해야 한다. 논문도 log를 가장 경합이 심한 자원으로 꼽았다.
- 설계 문서의 요구 사항에는 on-disk 형식을 바꾸지 않는 것도 있었다.

code: xfs_log_priv.h, xfs_log.c, xfs_log_format.h

### rmapbt와 reflink

rmapbt는 할당된 extent에서 owner를 거꾸로 찾는 btree이고([rmap과 refcount 구조](10-rmap-refcount.md)), reflink는 두 file이 같은 물리 block을 함께 쓰게 하는 기능이다([reflink, COW, dedupe, exchange-range](26-reflink-cow.md)).

- online fsck 설계 문서에 따르면 원래 설계("circa 1993")는 중복을 더하기를 꺼렸다. rmapbt는 그 뒤에 더한 보조 공간 index이고, online fsck, free space defrag, media 오류 보고, shrink를 가능하게 하는 열쇠다. directory tree, block map, AG 같은 주 metadata는 바꾸지 않아 규모 특성을 지켰다.
- rmapbt는 AG마다 하나씩 있고, record에는 AG block 번호, 길이, owner, owner 안의 offset이 있다. refcountbt도 AG마다 있고 record에는 시작 block, 길이, refcount가 있다. 두 superblock bit는 모두 ro_compat이라, 이 bit를 모르는 kernel도 read-only로는 mount할 수 있다.
- reflink file은 공유 block에 쓸 때 새 block을 잡아 쓰고, 성공하면 mapping을 바꿔 단다. 이때 delalloc을 그대로 쓰고, mapping은 따로 둔 COW fork에 둔다.
- zoned의 zone GC는 rmap으로 victim zone의 살아 있는 extent를 찾는다.
- mkfs에서는 reflink가 xfsprogs `ec1b42e6`(v5.1.0-rc1)부터, rmapbt가 `bcd5b1b7`(v6.5.0)부터 기본으로 켜진다. online fsck가 기능을 다 갖춰 reverse mapping을 켤 이유가 생겼기 때문에 rmapbt를 켰다. rmapbt는 kernel에 들어온 때(4.8, 2016-08 pull request mail)부터 mkfs 기본값이 되기(`bcd5b1b7`의 author date 2023-09-25)까지 약 7년 걸렸다.

code: xfs_format.h, xfs_reflink.c, xfs_zone_gc.c, xfs_mount.h

### online fsck

online fsck는 scrub과 repair 둘을 함께 가리킨다. scrub은 mount된 file system을 검사하고, repair는 깨진 metadata를 다시 짓는다. 두 도구와 xfs_repair의 동작은 [xfs_repair와 online fsck](32-fsck.md)에 있다.

- online fsck 설계 문서의 "Problem Statement"에는 일곱 가지가 있다. metadata가 조용히 깨져 있다가 갑자기 shutdown하는 것, shutdown 뒤 복구하는 동안이나 미리 검사하려고 offline으로 내린 동안 서비스가 멈추는 것, 운영 중에 건강 상태를 볼 수 없어 정비 시간을 잡지 못하는 것이 그 예다. 답은 mount된 file system에서 도는 세 번째 fsck 도구다. 앞의 두 도구 `xfs_check`와 `xfs_repair`는 unmount된 file system에서만 돈다.
- repair 도구는 논문 때 이미 필요했다. 논문에 따르면 log는 무작위 block 손상을 고치지 못하는데도 첫 release에는 필요 없으리라는 순진한 생각에서 repair 도구를 넣지 않았고, 고객 덕분에 그 생각이 틀렸음을 알게 되었다.
- Kconfig 도움말에 따르면 online repair에는 reverse mapping과 parent pointer 같은 보조 metadata가 있어야 한다. code에서는 AGF repair가 rmapbt 없이는 -EOPNOTSUPP를 돌려준다. directory repair에는 rmapbt와 exchange-range가 있어야 한다. parent pointer가 있으면 tree를 scan해 directory를 다시 짓고, 없으면 남은 entry를 건진다.
- 기준 tree에서는 `XFS_ONLINE_SCRUB`와 `XFS_ONLINE_REPAIR`가 모두 `default y`다. scrub에는 tmpfs와 shmem이 있어야 한다. ioctl로는 `XFS_IOC_SCRUB_METADATA`와, 여러 개를 한 번에 하는 `XFS_IOC_SCRUBV_METADATA`가 있다.
- mkfs에서는 xfsprogs `8aed3354`(v7.0.0)가, rmapbt와 parent pointer를 둘 다 켠 file system이면 사용자가 autofsck를 따로 주지 않았을 때 online repair 설정(autofsck)을 켠다. 보조 metadata의 비용을 이미 낸다면 self-repair까지 받는 편이 낫기 때문이다.

code: Kconfig, agheader_repair.c, dir_repair.c, xfs_fs.h

### parent pointer

child inode는 parent directory의 inode 번호와 entry 이름을 parent pointer라는 xattr로 들고 있다([xattr와 parent pointer](08-xattr-parent.md)). parent pointer가 있으면 root부터 directory tree를 거치지 않고도 child에서 parent를 찾을 수 있다.

- online fsck 설계 문서에 따르면 parent pointer가 없으면 directory tree를 다시 짓는 일이 막힌다. rmap이 없을 때 space metadata를 다시 짓는 일이 막혔던 것과 같다.
- SGI가 2000년대 후반에 먼저 제안했지만 Linux에 들어오지 않았다. 참조 무결성을 지킬 기반이 없었고, entry 이름을 기록하지 않아 tree를 다시 잇는 데 쓸 수 없었다.
- xattr의 이름은 directory entry의 이름이고, 값은 parent inode 번호와 generation이다. xattr namespace flag는 `XFS_ATTR_PARENT`, superblock bit는 incompat이고, user는 `XFS_IOC_GETPARENTS`로 읽는다.
- mkfs에서는 xfsprogs `54aad16b`(v6.18.0)가 parent pointer와 exchange-range를 기본으로 켰다. LTS가 다가오고 있었기 때문이다.

code: xfs_da_format.h, xfs_format.h, xfs_fs.h

### rtgroup과 metadir

rt section은 원래 bitmap 하나와 summary 하나로 관리했다. rtgroup은 AG의 생각을 rt section에 옮겨, rt section을 따로 잠글 수 있는 조각으로 나눈다. 조각마다 metadata inode가 생기고, metadir는 이 inode를 숨긴 directory tree에 둔다. rt section의 배치와 metadir tree는 [realtime, rtgroup, metadir](28-realtime.md)에 있다.

- in-core 구조는 `xfs_rtgroup`이고, group마다 bitmap, summary, rmap, refcount inode의 자리가 있다. superblock field로는 `sb_rgcount`와 `sb_rgextents`가 있다. zoned가 아니고 rt section이 있으면 첫 block에 rt superblock(magic 'Frog')이 있다.
- `xfs_has_rtgroups`는 `xfs_has_metadir`를 돌려준다. 주석에 따르면 metadir file system이면 모두 rtgroup을 허용한다. metadir tree의 root는 `sb_metadirino`다.
- option 없는 mkfs는 metadir를 끈다(`metadir=0`, `rgcount=0`).

code: xfs_rtgroup.h, xfs_format.h, xfs_mount.h, xfs_metadir.c

### zoned

Kconfig 도움말에 따르면 zoned block device를 쓸 때는 rt subvolume을 zoned device에 두고 metadata는 일반 block device(main device)에 둔다. 앞쪽에 conventional zone이 있는 host-managed SMR HDD라면 disk 하나로 되고, conventional zone이 main device, 나머지 zone이 rt subvolume이 된다. zone 안은 순서대로만 쓸 수 있어서, 남은 data를 다른 zone으로 옮기고 zone을 reset하는 zone GC가 필요하다. zone 상태와 zone GC는 [zoned XFS](29-zoned.md)에 있다.

- zoned는 incompat bit다. mount하려면 위 의존 표의 기능 말고도 rt extent 크기가 1이고, rt 크기가 rtgroup 크기의 배수이고, rtgroup이 5개 이상이어야 한다. 하나라도 어긋나면 mount가 거부된다.
- mount 때 `xfs_warn_experimental`이 "EXPERIMENTAL zoned RT device feature enabled."로 시작하는 경고를 찍는다. 이 경고는 `xfs_mountfs`를 부르기 전이라 zone 수 message보다 먼저 나온다(관찰에서도 "Mounting V5 Filesystem"보다 먼저 찍혔다).
- 기준 tree에서는 online shrink, logged xattr, zoned에 아직 실험 경고가 남아 있다. logged xattr는 DEBUG build의 knob로만 켠다.
- 운영 interface로는 `/sys/fs/xfs/<dev>/zoned/` 아래에 `max_open_zones`, `nr_open_zones`, `zonegc_low_space`가 있다.

zoned device를 rt device로만 주면 mkfs가 `metadir=1`, `zoned=1`, `rmapbt=1`, `reflink=0`을 스스로 고른다. mkfs는 zoned면 reflink를 끄고, 사용자가 reflink를 명시하면 멈춘다(code로만 확인). 64 MiB zone 16개로 된 1 GiB(1024 MiB) device에서는 zone 하나가 rtgroup 하나가 되었다.

| 값 | device | mkfs와 mount |
|---|---|---|
| zone 하나, rtgroup 하나의 크기 | 64 MiB / 4 KiB = 16384 block | `rgsize=16384 extents`. rt의 `blocks=262144`와 `rtextents=262144`가 같고 `extsz=4096`이라 rt extent 하나가 block 하나다 |
| 개수 | 1024 MiB / 64 MiB = zone 16개 | `rgcount=16`, mount의 "16 zones of 16384 blocks" |

code에서도 `xfs_report_zones`가 rtgroup마다 write pointer를 하나 물어 zone 하나를 초기화하고, zone capacity를 rtgroup 크기로 둔다.

code: Kconfig, xfs_super.c, xfs_zone_alloc.c, xfs_zone_gc.c, xfs_sb.c, xfs_zones.h, xfs_message.h

## 기준 이후 변화

- zoned의 실험 표시는 7.2에서 지웠다. pull request mail 한 건에서 이를 보았다.

## 문서와 code가 다른 곳

| 문서 | code |
|---|---|
| xfs-documentation의 "Theoretical Limits" 표: "Max AG Size"가 block 크기에 따라 2TiB, 8TiB, 128TiB | AG를 1TB로 제한하고, superblock 검사에서 이를 넘으면 거부한다(`XFS_MAX_AG_BYTES`) |
| 같은 표: "Inodes 2^63/2^64". 논문: inode 번호는 64-bit | type은 64-bit지만 상한은 2^56 - 1이다(`XFS_MAXINUMBER`). shortform directory가 inode 번호를 이 mask로 자르고, statfs의 `f_files`도 이 값으로 자른다 |
| 논문: 새 directory는 parent와 다른 AG에 둔다 | rotor는 parent의 AG를 보지 않는다(`xfs_dialloc_pick_ag`). 관찰에서도 root 아래의 d1과 d5가 root와 같은 AG 0에 놓였다 |
| delayed logging 설계 문서: 요구 사항 "Enable and disable with a mount option". CIL 구조체의 주석: delayed logging mount option을 켰을 때만 쓴다 | option이 없고 CIL을 늘 만든다(`xlog_cil_init`) |
| 관리 문서의 "Removed Mount Options" 표: `delaylog/nodelaylog`를 v4.0에 지웠다 | 기준 tree의 fs/xfs에는 `delaylog` 문자열이 없다. 3.3의 pull request mail에는 "xfs: remove the deprecated nodelaylog option"이 있다 |
| xfs_super.c에서 `XFS_FEAT_CRC`를 보는 자리의 주석: "xfs_has_crc always returns false for !CONFIG_XFS_SUPPORT_V4" | v4 지원을 끄면 늘 true를 돌려준다(`__XFS_HAS_V4_FEAT`). 주석의 결론(그래서 `m_features`를 직접 본다)은 맞고 방향만 반대다 |
| Kconfig 도움말(`XFS_ONLINE_REPAIR`): repair에는 reverse mapping과 parent pointer가 있어야 한다 | parent pointer가 없다고 -EOPNOTSUPP를 돌려주는 repair는 없다. parent pointer가 있어야 하는 것은 directory tree 구조 검사(dirtree) 하나이고, parent pointer가 없으면 scrub ioctl이 그 검사와 repair를 -ENOENT로 받지 않는다 |
| 관리 문서와 Kconfig: "on-disk compatible with the IRIX version of XFS" | v1~v3 superblock을 받지 않고(`xfs_sb_good_version`), 관찰 image처럼 v4 지원을 끈 build는 v4도 받지 않는다 |

## 한계점

다음은 확인하지 못했다.

- kernel 쪽 기능 도입 commit. 읽은 kernel tree에서 history를 볼 수 없어서 version은 모두 pull request mail에서 옮겼다.
- XFS가 Linux mainline에 들어온 kernel version과 시점.
- buffered write가 지금도 IOLOCK을 exclusive로 잡는 이유. 논문에 따르면 없앨 계획이었다.
- zoned 관찰에서 mkfs가 `sectsz=4096`을 고른 이유. option 없는 1 GiB mkfs는 `sectsz=512`였다.
- autofsck(`8aed3354`)가 on-disk 어디에 기록되는지.
- 2030년 v4 제거 때 지워질 code의 범위.
- v4 file system의 동작. 관찰 image가 v4 지원을 끈 build라 mount할 수 없다. mount 거부는 이 image에서 볼 수 있지만 관찰하지 않았고 code로만 읽었다.
- 3.3에 `nodelaylog`를, 4.0에 `delaylog` 자체를 지웠는지.
- IRIX가 만든 file system이 v4 조건(v2 directory, extent flag)을 채우는지.
- 기준 이후의 mailing list와 xfs-linux branch head. zoned의 실험 표시 말고는 보지 않았다.
- LWN의 XFS 기사. 열지 않았고 그 내용은 이 page에 쓰지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- Kconfig는 관찰에 쓴 kernel image를 build한 `.config`를 host에서 grep으로 읽었다.
- zoned 관찰에는 null_blk을 configfs로 만든 zoned device(memory_backed, 크기 1 GiB(1024 MiB), block 4096 byte, zone 64 MiB, conventional zone 0개)를 rt device로 주었다. data device는 1 GiB loop device다. mkfs에는 `-r rtdev=`만 주었고 mount option으로는 `rtdev=` 하나를 주었다.
- 문헌: 1996년 USENIX 논문 "Scalability in the XFS File System", kernel의 Documentation/filesystems/xfs/(self-describing metadata, delayed logging, online fsck 설계 문서)와 Documentation/admin-guide/xfs.rst(관리 문서), xfs-documentation, xfsprogs v7.1.1의 mkfs/xfs_mkfs.c와 git history. 기능의 mainline version은 XFS pull request mail에서 옮겼다.

### 관찰 목록

| 관찰 | 절차 | 본문의 예 |
|---|---|---|
| superblock과 mkfs 기본값 | option 없이 mkfs한 뒤 mount 전에 xfs_db로 `version`과 superblock을 읽고, mount해서 kernel log와 `xfs_info`를 봄 | 판 번호, feature mask 표, 논문의 결정 표의 AG 크기, SGI의 흔적의 boot banner |
| 새 directory와 AG | directory d1~d8을 차례로 만들고 각각에 file f를 만든 뒤 `stat -c %i`로 inode 번호를 보고, unmount한 뒤 `xfs_db -r -c "convert inode <번호> agno"`로 AG 번호를 봄. convert의 쓰임은 fstests의 xfs/176을 따름 | 새 directory와 AG 표 |
| Kconfig | host에서 관찰 image의 `.config`를 grep | 기본값과 폐기 일정 |
| zoned | null_blk zoned device를 rt device로 준 mkfs와 mount, kernel log | zoned 표 |

### 출력 읽는 법

- xfs_db `version` 출력의 대괄호 안 두 값은 versionnum과 features2이고, 그 뒤에 V5부터 PARENT까지 feature 이름이 붙는다. superblock의 crc field 옆에는 "(correct)"가 찍혔다.
- mount한 file system의 `xfs_info`는 mkfs 출력과 같은 줄을 찍었고 `crc=1`이 있다.

### 원본 출력

#### superblock과 mkfs 기본값

(원본 출력 "option 없는 mkfs의 출력": `raw/01-overview-history.md`)

(원본 출력 "superblock의 판 번호와 feature mask, mount message": `raw/01-overview-history.md`)

(원본 출력 "관찰 image의 uname과 boot banner": `raw/01-overview-history.md`)

#### 새 directory와 AG

(원본 출력 "superblock geometry, inode 번호, AG 번호": `raw/01-overview-history.md`)

#### Kconfig

(원본 출력 "관찰 image의 XFS Kconfig": `raw/01-overview-history.md`)

#### zoned

(원본 출력 "zoned device, mkfs 출력, mount의 kernel log": `raw/01-overview-history.md`)
