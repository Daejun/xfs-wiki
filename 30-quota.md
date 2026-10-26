# quota

> XFS 전반 wiki, 묶음 "기능". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/30-quota.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/30-quota.md, 접힌 원본 출력은 raw/30-quota.md.

## 개요

XFS quota는 user, group, project마다 쓰는 block 수와 inode 수를 세고, limit을 넘는 할당을 막는다. id 하나의 limit, 사용량, grace timer를 담은 record를 dquot이라 하고, dquot는 메모리와 disk의 quota inode 양쪽에 있다.

- limit은 block을 할당할 때가 아니라 예약(reservation)할 때 검사한다. buffered write는 write(2) 안에서 transaction 없이 예약하므로, hard limit이 1 MiB이고 이미 68 KiB를 쓴 user가 2 MiB를 64 KiB씩 쓰면 896 KiB까지 쓴 뒤 다음 write(2)가 `EDQUOT`로 실패한다.
- transaction은 dquot에 변화량(delta)만 쌓았다가 commit에서 in-core dquot에 반영하고, dquot 값 전체를 log에 싣는다. 그래서 recovery는 delta를 다시 더하지 않고 log의 값으로 덮어쓴다.
- superblock에 켜진 type의 quotacheck 완료 bit(CHKD)이 없으면, mount가 log recovery 뒤에 모든 inode를 돌며 사용량을 다시 센다. non-metadir file system은 quota option 없이 한 번 mount하면 이 bit이 지워진다.

예의 값은 uid 1000이 1 GiB file system에 file을 쓴 경우에서 얻었고, file system은 metadir(metadata file을 사용자에게 보이지 않는 directory tree에 담는 기능, [realtime, rtgroup, metadir](28-realtime.md))을 켠 것과 끈 것(non-metadir) 두 가지다.

## 켜고 끄는 상태

quota를 켜고 끈 상태는 superblock의 `sb_qflags`에 남고, mount 중에는 in-core `m_qflags`를 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

quota type마다 accounting(ACCT), enforcement(ENFD), quotacheck가 끝났다는 표시(CHKD)의 bit이 하나씩 있다.

| type | ACCT | ENFD | CHKD |
|---|---|---|---|
| user | 0x1 | 0x2 | 0x4 |
| group | 0x40 | 0x80 | 0x100 |
| project | 0x8 | 0x200 | 0x400 |

- `m_qflags`는 mount option으로 정해진다. `uquota`는 ACCT와 ENFD를, `uqnoenforce`는 ACCT만 켜고, group과 project도 같다.
- quota option이 없으면 non-metadir에서는 "resetting quota flags"를 찍고 `sb_qflags`를 CHKD까지 0으로 지운다. metadir에서는 `xfs_qm_resume_quotaon`이 disk의 ACCT와 ENFD를 되살리고 CHKD도 남으므로 quotacheck가 돌지 않는다. `-o noquota`를 주면 non-metadir처럼 reset된다.
- xfsprogs v7.1.1의 `mkfs.xfs -m uquota`는 ACCT와 ENFD만 미리 넣는다. metadir을 따로 정하지 않았으면 metadir을 켜고, `-m metadir=0`과 같이 주면 거부한다.

세 file system을 차례로 mount했을 때 umount 뒤의 `sb_qflags`는 다음과 같았다. 표에서 quota라고 쓴 mount는 `-o uquota,pquota`를 주었고, `0x60f`는 user의 `0x7`에 project의 `0x8`, `0x200`, `0x400`을 더한 값이다.

| file system | mount | mount 때 | qflags |
|---|---|---|---|
| non-metadir | quota | quotacheck | `0x60f` |
|  | quota | quotacheck 없음 | `0x60f` |
|  | option 없음 | reset | 0 |
|  | quota | quotacheck | `0x60f` |
| metadir | quota | quotacheck | `0x60f` |
|  | option 없음 | quotacheck 없음, mount option에 `usrquota,prjquota` | `0x60f` |
|  | `-o noquota` | reset | 0 |
|  | option 없음 | quotacheck 없음 | 0 |
|  | quota | quotacheck | `0x60f` |
| `-m metadir=1,uquota` | (mkfs 직후) |  | `0x3` |
|  | option 없음 | quotacheck, mount option에 `usrquota` | `0x7` |

### 실행 중에 끄기

mount 중에 quota를 끄면 enforcement만 꺼지고 accounting은 계속된다. accounting까지 끄라는 요청이 오면 `xfs_qm_scall_quotaoff`는 "disabling of quota accounting not supported."만 찍고 ENFD만 지워 `sb_qflags`에 쓴다. 예를 들어 `-o uquota,pqnoenforce`(`qflags` `0x40f`, project는 pqnoenforce라 ACCT와 CHKD만)에서 `disable -u`와 `off -u`를 차례로 하자 user의 enforcement만 꺼지고 accounting은 켜진 채로 남았다. message는 `off -u`에서만 찍혔다. 그 뒤 `qflags`는 user ENFD(`0x2`)만 빠진 `0x40d`였고, uid 1000이 쓴 256 KiB도 report에 잡혔다.

code: xfs_log_format.h, xfs_super.c, xfs_mount.c, xfs_qm_bhv.c, xfs_qm_syscalls.c

## disk의 dquot

dquot는 quota type마다 quota inode 하나 안에 id 순서로 놓이므로, id로 위치를 계산할 수 있다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### quota inode

- non-metadir에서는 superblock의 `sb_uquotino`, `sb_gquotino`, `sb_pquotino`에 quota inode 번호가 들어 있다. 예에서는 131, null, 132였다. v4에서는 gquotino 하나를 group과 project가 번갈아 쓰므로 둘을 같이 켤 수 없다.
- metadir에서는 세 field가 disk에서 0(in-core는 NULLFSINO)이고, quota inode는 metadir tree의 `/quota/user`, `/quota/group`, `/quota/project`에 있다. 처음 quota를 켜고 mount할 때 켜진 type의 것이 없으면 그때 만들므로, 예에서는 `user`와 `project`만 있었다([realtime, rtgroup, metadir](28-realtime.md)).
- quota inode와 metadir inode 자신은 quota에 세지 않는다.

### record와 위치

record 하나(`struct xfs_dqblk`)에는 magic(`0x4451`), type(user 1, project 2, group 4, bigtime `0x80`), id, limit, 사용량(`d_bcount`, `d_icount` 등), timer, warn 횟수가 있다. 끝에 CRC, LSN, UUID까지 붙으면 136 byte가 된다. 4 KiB block 하나에는 record가 30개(4080 byte) 들어가고 16 byte가 남는다. 그래서 id N의 record는 file block `N / 30`의 `(N % 30) * 136` byte 자리에 있고, id 30부터는 `N * 136`과 어긋난다.

| id | file block | block 안 offset | file 안 byte | id × 136 |
|---|---|---|---|---|
| 29 | 0 | 3944 | 3944 | 3944 |
| 30 | 1 | 0 | 4096 | 4080 |
| 42 (project) | 1 | 1632 | 5728 | 5712 |
| 1000 | 33 | 1360 | 136528 | 136000 |

실제 위치도 계산과 같았다. user quota inode는 file block 0과 33에, project quota inode는 0과 1에 block이 있었고, log의 dquot item도 offset 1360을 가리켰다.

quota file은 sparse file이라, 처음 쓰는 id의 자리는 hole일 수 있다. 그 자리가 hole이면 create, chown, chproj, quotacheck, limit 설정일 때만 별도 transaction으로 block 하나를 할당해 record 30개를 초기화한다. 다른 호출은 `-ENOENT`를 받는다. 이 할당은 file size를 고치지 않으므로 `core.size`는 0으로 남는다(이유는 code로만 확인, 관찰한 값도 0이었다).

in-core dquot에는 사용량 `count`와, 여기에 예약을 더한 `reserved`가 따로 있다. disk에는 `count`만 쓰고 읽을 때는 `reserved = count`로 시작하므로, delalloc 예약은 disk에 남지 않는다.

### id

user와 group은 inode의 uid와 gid를, project는 `i_projid`를 id로 쓴다. 새 inode는 parent에 PROJINHERIT가 있으면 parent의 projid를, 없으면 0을 받으므로 project quota는 directory tree 단위가 된다. inode core에서 이 id와 flag가 놓이는 자리는 [inode](06-inode.md)에 있다. projid가 다른 inode를 PROJINHERIT directory로 link하거나 rename하면 `-EXDEV`로 실패한다. 예를 들어 directory `d`에 project 42와 PROJINHERIT를 준 뒤 만든 `p1`은 projid 42를 받았고, 그 전에 만든 `f1`은 0으로 남았다.

code: xfs_dquot_buf.c, xfs_sb.c, xfs_qm.c, xfs_format.h, xfs_dquot.c, xfs_dquot.h, xfs_inode_util.c, xfs_inode.c

## limit 검사

limit과 비교할 때는 `count`가 아니라, delalloc 예약까지 합친 `reserved`에 이번 요청을 더한 값을 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 검사하는 곳

대표 진입점은 세 곳이 있다.

| 경로 | 진입 함수 | transaction |
|---|---|---|
| buffered write (delalloc) | `xfs_bmapi_reserve_delalloc` | 없다. 실패를 write(2)가 받는다. 그 전에 preallocation 없이 한 번, preallocation을 거둔 뒤 한 번 더 시도한다 |
| direct write, fallocate, 일반 inode 변경 | `xfs_trans_alloc_inode` | 있다. 실패하면 preallocation을 거둔 뒤 다시 시도한다 |
| create (inode 1개와 directory block) | `xfs_trans_alloc_icreate` | 있다 |

directory 변경, rename, reflink, exchange-range, chown, chproj, scrub 등도 같은 방식으로 예약한다. buffered write의 흐름은 [buffered write](18-buffered-write.md)에 있다.

### 판정 규칙

예약은 `xfs_trans_reserve_quota_bydquots`가 user, group, project 순으로 하고, 하나라도 실패하면 앞의 예약을 되돌리므로 전부 되거나 전부 안 된다. dquot 하나의 판정은 `xfs_trans_dqresv`가 한다.

- block을 먼저, inode를 다음에 검사한다. 실패하면 project는 `-ENOSPC`, user와 group은 `-EDQUOT`를 돌려준다.
- `XFS_QMOPT_FORCE_RES`를 준 예약이거나(예: force로 한 chown, chproj), dquot의 id가 0이거나, enforcement가 꺼져 있으면 검사를 건너뛴다. 이때도 `reserved`는 올리므로 accounting은 그대로다.
- id 0 면제는 process가 root인지가 아니라 dquot의 id로 정한다. 그래서 root가 project 42 tree에 쓰면 project 42 dquot가 검사된다.
- dquot에 limit이 없으면 default limit을 쓰고, XFS에서는 id 0 dquot의 limit이 곧 default limit이 된다. 그래서 `limit -u bhard=N 0`은 root가 아니라 자기 limit이 없는 모든 user에게 걸린다.
- hard를 넘으면 바로 실패하고, hard와 같으면 통과한다. soft를 넘으면 timer가 정해져 있고 이미 지났을 때만 실패하고, 아니면 경고만 한다.
- timer는 commit 때 count가 soft나 hard를 넘으면 지금 시각에 grace를 더한 값으로 정하고, 내려오면 0으로 되돌린다. id 0 dquot의 timer field에는 만료 시각이 아니라 default grace 길이가 들어 있다.

block 수에는 attr fork block도 들어간다. 할당할 때 data fork와 attr fork를 가리지 않고 `i_nblocks`를 올리면서 같은 양을 dquot에 넘기고, quotacheck도 `i_nblocks`를 세기 때문이다. 예를 들어 uid 1000의 4 KiB file에 60000 byte xattr value를 붙이자 attr fork extent 1개(64 KiB)가 생겨 사용량이 4 KiB에서 68 KiB가 되었다.

### uid 1000을 따라가기

hard limit은 write(2)에서 걸리고, 그 뒤의 create는 inode limit에 닿기 전에 실패했다. xattr 예의 uid 1000(68 KiB 사용)에 soft 512 KiB, hard 1 MiB, inode hard 4를 준 경우다. disk의 dquot에는 limit이 fs block 단위(`blk_softlimit` 128, `blk_hardlimit` 256)로 들어갔다.

| 단계 | block 사용량 | inode | 결과 |
|---|---|---|---|
| 시작 | 17 block(68 KiB) | 2(file `x`와 directory `u`) |  |
| 2 MiB를 64 KiB씩 write | 17 + 224 = 241 block(964 KiB) | 3 | file이 896 KiB가 된 뒤 EDQUOT. soft를 넘어 grace timer가 설정되었다(`[7 days]`) |
| (실패한 다음 64 KiB) | 241 + 16 = 257 > 256 |  | hard에 걸린 것으로 보인다(추정) |
| `touch` 4번 | 241 + 90 > 256 | 3 + 1 = 4, hard를 넘지 않음 | 4번 모두 EDQUOT |

`touch`가 실패한 것은 create가 inode와 함께 block 90개(transaction 관찰의 create 값)를 예약하고 block을 먼저 검사하기 때문으로 보인다(추정).

### default limit과 project limit

- `limit -u bhard=256k 0`을 주자 limit이 없는 uid 1001은 file을 만들다가 `Disk quota exceeded`로 실패했다. create의 block 예약이 default 64 block을 넘은 것으로 보인다(추정). root는 1 MiB를 다 써서 `#0`의 사용량이 hard 256 KiB를 넘는 1024 KiB가 되었다. id 0 dquot는 검사를 건너뛰기 때문이다. uid 1000의 limit 칸은 자기 값(512, 1024) 그대로였다.
- project limit은 root도 막는다. project 42에 hard 512 KiB를 주고 root가 1 MiB를 쓰자 `ENOSPC`로 실패했고, 사용량은 512 KiB였다.

code: xfs_trans_dquot.c, xfs_iomap.c, xfs_trans.c, xfs_qm.c, xfs_qm_syscalls.c, xfs_dquot.c, xfs_bmap.c, xfs_inode.c

## dquot와 transaction

transaction이 쌓은 delta는 commit에서 in-core dquot에 들어가고, quota file의 record는 나중에 AIL이 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

**도식: dquot 값이 예약에서 quota file까지 가는 길**

```text
create (uid 1000)
   |
   |  (1) 예약: reserved + 요청량을 limit과 비교
   v
in-core dquot                 reserved += 요청량
   |
   |  (2) transaction 안: count는 그대로 두고 delta만 쌓음
   v
transaction의 dqtrx 칸        RES_REGBLKS +90, RES_INOS +1, ICOUNT +1
   |
   |  (3) commit
   v
in-core dquot                 count += 쓴 양
   |                          reserved -= 쓰지 않은 예약
   |                          dquot 값 전체를 log item에 실음
   v
log (DQUOT item, region 2개)
   |
   |  (4) AIL push -> dqflush
   v
quota file의 record           file block 33, offset 1360
```

### transaction을 열기 전

inode는 `i_udquot`, `i_gdquot`, `i_pdquot`로 자기 dquot를 가리키고, dquot는 transaction을 열기 전에 `xfs_qm_dqattach`로 구해 붙여 둔다. create는 새 inode가 받을 id로 dquot를 먼저 구한다. user dquot는 fsuid로, group dquot는 fsgid(parent가 setgid면 parent의 gid)로, project dquot는 parent가 PROJINHERIT일 때 parent의 projid로 구한다. 순서는 [create, link, rename](24-create-link-rename.md)에 있다.

### delta

`xfs_trans_mod_dquot`는 transaction 안에 dquot별 칸(dqtrx)을 두고 field별 delta만 더한다. superblock counter도 이렇게 delta로 쌓는다. field는 예약(`RES_BLKS`, `RES_INOS` 등), 사용(`BCOUNT`, `ICOUNT` 등), delalloc에서 실제 할당으로 바뀌는 양(`DELBCOUNT` 등)으로 나뉜다. `RES_BLKS`는 trace에 `RES_REGBLKS`로 찍힌다. 사용량 delta는 대표적으로 다음 곳에서 넣는다.

| 일 | delta | 함수 |
|---|---|---|
| block 할당 | delalloc에서 왔으면 `DELBCOUNT`, 아니면 `BCOUNT`를 더함 | `xfs_bmap_alloc_account` |
| bmbt block 할당, 해제 | `BCOUNT` +1, -1 |  |
| extent 해제 | 해제한 만큼 `BCOUNT`를 뺌 | `xfs_bmap_del_extent_real` |
| inode 해제 | `ICOUNT` -1([unlink 뒤의 흐름](25-unlink-inactivation.md)의 inodegc worker에서) | `xfs_inactive_ifree` |
| chown | 옛 dquot에서 빼고 새 dquot에 더함 | `xfs_qm_vop_chown` |

fork 형식 변환, reflink remap, exchange-range, scrub repair도 delta를 넣는다. [transaction](11-transaction.md)의 rolling transaction에서는 남은 예약을 다음 transaction으로 넘긴다.

### commit

commit은 precommit 전에 `xfs_trans_apply_dquot_deltas`를 부른다.

1. 바뀐 dquot를 id 순서로 lock하고 transaction에 join한다.
2. `count`에 BCOUNT와 DELBCOUNT를 더하고, timer를 맞춘다(id 0은 빼고). 그다음 dirty로 표시하고 log item을 붙인다.
3. 예약한 양에서 실제로 쓴 BCOUNT를 뺀 나머지를 `reserved`에서 내린다. DELBCOUNT는 delalloc 때 이미 `reserved`에 들어갔으므로 빼지 않는다.

cancel하거나 dirty가 아닌 채로 commit하면 예약만 되돌린다.

### log item, writeback, recovery

- log item은 dquot마다 하나씩 dquot 안에 있고, type은 `XFS_LI_DQUOT`(`0x123d`)이다. item에는 id와 위치를 담은 `struct xfs_dq_logformat`과 `struct xfs_disk_dquot` 전체가 region 두 개로 들어간다.
- recovery는 log의 `xfs_disk_dquot`로 buffer의 dquot를 통째로 덮어쓰고, buffer의 `dd_lsn`이 replay 중인 LSN 이상이면 건너뛴다.
- disk에는 [log](12-log.md)의 AIL이 쓴다. item을 push하면 `xfs_qm_dqflush`가 in-core 값을 buffer의 record에 복사하고, crc가 켜져 있으면 `dd_lsn`과 checksum을 쓴다. 주석에 따르면 dqflush가 reclaim context에서도 불리므로, precommit이 dquot buffer를 미리 붙여 두어 그때 buffer를 할당하지 않게 한다.
- `XFS_LI_QUOTAOFF` item은 정의와 크기 검사 말고는 recovery code에만 있다(code로만 확인). 지금 kernel은 quotaoff를 log하지 않으므로 옛 log를 replay하려고 남긴 것으로 보인다(추정).

### 단계별 dquot tracepoint

buffered write는 write(2) 때 in-core dquot의 `reserved`만 올리고, `count`는 delalloc block이 실제로 할당될 때(예에서는 fsync가 writeback을 일으켰을 때) `DELBCOUNT` delta로 올린다. direct write는 할당 transaction마다 `BCOUNT` delta로 `count`를 올린다. uid 1000이 처음 file을 만들고 쓴 경우다.

| 단계 | trace에 나온 것 |
|---|---|
| 첫 create | `xfs_dqread`, `xfs_dqalloc`, 별도 transaction의 commit 두 줄(dquot block 할당), `xfs_dqget_miss`. 이어서 create transaction에 `RES_REGBLKS` +90, `RES_INOS` +1, `ICOUNT` +1 |
| buffered write 256 KiB(fsync 전) | delta 없음. transaction 없는 예약은 delta를 쌓지 않는다 |
| fsync | 할당 transaction에 `DELBCOUNT` +64(256 KiB의 block 수) |
| 새 file에 direct write 64 KiB | create 몫 뒤에 `BCOUNT` +1이 든 transaction이 따로따로 나왔다(남긴 80줄 안에 10번). `BCOUNT` +16은 한 번도 없었다. xfs_io가 4 KiB씩 16번 쓴 결과로 보인다(추정) |
| freeze와 thaw | `xfsaild`가 `xfs_dqflush`를 부른다([mount, unmount, freeze](16-mount-freeze.md)) |

빈 file 하나를 만들고 128 KiB file 하나를 더 쓴 뒤 shutdown하자 log에는 dquot item 하나가 region 2개, offset 1360으로 남았다. 다시 mount하면 log recovery가 돌고, quotacheck 없이 shutdown 직전과 같은 값(256 + 64 + 128 = 448 KiB, file 4개)이 나왔다.

code: xfs_qm.c, xfs_trans_dquot.c, xfs_trans.c, xfs_dquot.c, xfs_dquot_item.c, xfs_dquot_item_recover.c, xfs_bmap.c, xfs_bmap_btree.c, xfs_inode.c

## quotacheck

quotacheck는 transaction 없이 모든 inode를 세어 dquot를 다시 쓴다. 이 절의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### 도는 조건

켜진 type 가운데 하나라도 disk의 `sb_qflags`에 CHKD가 없으면 돈다. qflags 표의 첫 quota mount, reset 뒤의 quota mount, `-m uquota`로 만든 file system의 첫 mount가 그렇다. 꺼진 type의 CHKD는 mount마다 지우므로, 꺼져 있던 type을 새로 켤 때도 돈다. xfs_repair는 자기 quotacheck가 불일치를 찾았을 때만 CHKD를 지운다(xfsprogs v7.1.1 code로만 확인).

- `xfs_qm_newmount`는 disk에 quota가 이미 켜져 있고 quotacheck가 필요 없으면 바로 quota를 켠다. 아니면 `m_qflags`를 잠시 0으로 두어 그동안 inode inactivation이 quota code를 건드리지 않게 하고, log recovery를 마무리하는 `xfs_log_mount_finish`가 끝난 뒤 되돌린다. 그래서 quotacheck는 log recovery 뒤에 돈다. recovery는 [deferred operation, intent, log recovery](13-intent-recovery.md)에, mount 순서는 [mount, unmount, freeze](16-mount-freeze.md)에 있다.
- quota를 켜는 `xfs_qm_mount_quotas`는 rt device가 있는데 rtgroup이 없거나 zoned면 "Cannot turn on quotas for realtime filesystem"을 찍고 quota를 끈다([realtime, rtgroup, metadir](28-realtime.md), [zoned XFS](29-zoned.md)). 끝에서는 `m_qflags`를 `sb_qflags`에 넣고, 바뀌었으면 superblock을 쓴다.
- 시작할 때 "Quotacheck needed: Please wait.", 끝나면 "Quotacheck: Done."을 찍는다. 실패하면 "Quotacheck: Unsuccessful (Error %d): Disabling quotas."를 찍고 `sb_qflags`를 reset한 뒤 quota 없이 mount를 계속한다(관찰하지 않음).

### 하는 일

1. **0으로 초기화한다.** quota inode의 할당된 block을 모두 읽어 깨진 record는 고치고, 사용량을 0으로 만든다. id 0이 아니면 timer와 warn도 0으로 만들지만, id 0에는 default grace와 warn 한계가 있으므로 그대로 둔다. limit은 건드리지 않고, 이 변경은 log하지 않는다.
2. **모든 inode를 센다.** `xfs_iwalk_threaded`로 inode마다 `xfs_qm_dqusage_adjust`를 부른다. 이 함수는 quota inode와 metadir inode를 빼고, 켜진 type의 dquot에 inode 1개와 `i_nblocks`를 더한 뒤 timer를 맞춘다(rt block은 따로 센다). 중간에 crash하면 처음부터 다시 하면 되므로 transaction 없이 dirty로만 표시한다.
3. **쓰고 표시한다.** dirty dquot를 delwri queue에 모아 한꺼번에 쓰고, 성공하면 `m_qflags`에 CHKD를 세운다.

quotacheck 중에 새로 할당하는 dquot block은 그 type의 CHKD가 아직 없으면 log하지 않고 ordered buffer로만 처리해, log된 초기화가 recovery 때 검사 결과를 덮어쓰지 않게 한다.

### 다시 센 결과

다시 센 값은 실행 중에 쌓인 값과 같았다. uid 1000이 directory `d`에 64 KiB file 5개를 만들고, `d`에 project 42를 준 뒤 128 KiB file 하나를 더 만들었다. 이 file system을 다시 세면 다음과 같다.

| dquot | inode | block |
|---|---|---|
| user 1000 | 7 (directory 1, file 6) | 112 block(64 KiB × 5 + 128 KiB = 448 KiB) |
| project 42 | 2 (directory, file 1) | 32 block(128 KiB) |
| user 0 | non-metadir 3, metadir 1 | 0 |
| project 0 | non-metadir 8, metadir 6 | 80 block(320 KiB) |

두 file system은 id 0의 inode 수만 2개 다르다. non-metadir에서는 rt device가 없어도 mkfs가 만든 rt bitmap과 rt summary inode가 uid 0, project 0으로 세어지고, metadir에서는 그런 inode가 있더라도 metadir inode라 빠지는 것으로 보인다(추정).

code: xfs_quota.h, xfs_qm_bhv.c, xfs_mount.c, xfs_qm.c, xfs_dquot.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| xfs_quota(8) | `off`는 "Permanently switches quota off" | enforcement만 꺼지고 accounting은 계속된다 |
| xfs-documentation의 quota inode 절 | "ID offset multiplied by the size of xfs_dqblk_t (136 bytes)"로 record를 찾는다 | block마다 16 byte 빈 칸이 있어 id 30 이상에서는 맞지 않는다 |
| 같은 절 | "Extended attributes do not contribute towards the ID's quota." | attr fork block도 사용량에 들어간다 |
| xfs-documentation의 "Update Quota File" 절 | `qlf_type`이 "inode create operation, 0x123e" | dquot item은 `0x123d`이고, `0x123e`는 `XFS_LI_QUOTAOFF`다 |

## 한계점

- direct write에서 `RES_REGBLKS` +12만 든 transaction이 무엇인지 확인하지 못했다. `RES_REGBLKS` +7과 `BCOUNT` +1이 든 transaction과 번갈아 나왔다(원본 출력 block에는 앞부분만 옮겼다).
- id 0의 inode 수 차이가 rt bitmap과 rt summary inode 때문인지 inode 번호로 확인하지 못했고, metadir file system에 그 두 inode가 있는지도 보지 않았다.
- `touch`와 uid 1001의 create가 block limit에 걸렸다는 것을 trace로 확인하지 못했다.
- shutdown 직전에 바뀐 dquot가 buffer에는 없고 log에만 있었는지 확인하지 못했다.
- xfs_repair 뒤의 CHKD 처리, quotacheck의 실패 경로, recovery에서 dquot item을 replay하는 경로는 관찰하지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 mkfs를 새로 했다. mkfs option은 `-m metadir=0`(non-metadir), `-m metadir=1`, `-m metadir=1,uquota` 세 가지를 썼다. 관찰은 한 번씩 실행했다.
- file은 대부분 uid 1000이 썼다. file을 쓸 때는 distro의 `xfs_io`를, 나머지는 xfsprogs v7.1.1의 도구를 썼다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| mount와 quotacheck | 세 file system을 qflags 표의 순서로 mount하고 umount마다 superblock을, 첫 quota mount 뒤에는 quota inode와 dquot record도 읽었다. file은 첫 quota mount 뒤에 만들었으므로, 첫 quotacheck는 빈 file system을 셌다 | 없음 | qflags 표, record와 위치, id, 다시 센 결과 |
| quota 끄기 | non-metadir, `-o uquota,pqnoenforce`. `state`를 보고, 다시 mount해 `disable -u`, `off -u` 뒤 256 KiB를 썼다 | 없음 | 실행 중에 끄기 |
| limit 검사 | non-metadir, `-o uquota,pquota`. uid 1000의 4 KiB file에 `setfattr`로 60000 byte value를 붙이고, `limit -u bsoft=512k bhard=1m isoft=0 ihard=4 1000` 뒤 `pwrite -b 64k 0 2m`와 `touch` 4번, `limit -u bhard=256k 0` 뒤 uid 1001과 root가 `pwrite -b 64k 0 1m`, `limit -p bhard=512k 42` 뒤 root가 project 42 directory에 같은 write를 했다. umount 뒤 xfs_db로 dquot를 읽었다 | 없음 | attr fork, uid 1000을 따라가기, default limit과 project limit |
| dquot와 transaction | non-metadir, `-o uquota`. 단계마다 trace를 켜고 껐다: `touch`, `pwrite -q 0 256k`, `fsync`, 새 file에 `-d`로 `pwrite -q 0 64k`(`-b` 없이), `freeze`와 `thaw`. direct write의 trace는 134줄 가운데 앞 80줄만 남겼다. 이어 `touch`와 `-d`의 `pwrite -q 0 128k`, `shutdown -f`, umount 뒤 `xfs_logprint -t`, 다시 mount. trace는 tracefs instance가 아니라 top-level buffer(`/sys/kernel/debug/tracing`의 `set_event`, `trace`)에서 켰다 | `xfs_trans_mod_dquot`, `xfs_trans_apply_dquot_deltas_after`, `xfs_dqalloc`, `xfs_dqread`, `xfs_dqget_miss`, `xfs_dqflush`, `xfs_trans_commit` | 단계별 dquot tracepoint |

### 출력 읽는 법

- report의 block 칸은 KiB 단위이고, 그 값은 `count`가 아니라 예약까지 더한 `reserved`다(code로만 확인).
- xfs_db의 `diskdq.bcount`, `blk_hardlimit`, `blk_softlimit`은 4 KiB block 단위다(`bcount = 112`는 448 KiB, `blk_hardlimit = 256`은 1 MiB). `ino_`로 시작하는 field는 inode 수다.
- trace의 `id 0x3e8`은 uid 1000이고, dquot 줄의 `res_bc`는 `reserved`다. `bcnt`, `rtbcnt`, `ihardlimit`, `isoftlimit` field는 명령이 지웠다.
- `xfs_trans_apply_dquot_deltas_after` 줄은 쓰지 않은 예약을 내리기 전에 찍히므로, 그 줄의 `res_bc`에는 그 transaction이 예약한 block이 아직 남아 있다(예: 첫 create 뒤의 `res_bc 0x5a`는 create가 예약한 90 block이다).
- dmesg 줄에는 앞선 관찰이 남긴 XFS 줄이 섞여 있어, 그 단계의 줄만 옮겼다.
- xfs_io의 실패 message가 `pwrite:`가 아니라 file 경로로 시작하면 file을 여는 단계(`-f`의 O_CREAT)에서 실패한 것이다. uid 1001의 실패가 이 꼴이다.

### 원본 출력: mount와 quotacheck

(원본 출력 "non-metadir fs의 mount와 sb_qflags 변화": `raw/30-quota.md`)

(원본 출력 "metadir fs의 mount와 sb_qflags 변화": `raw/30-quota.md`)

(원본 출력 "mkfs -m metadir=1,uquota 직후의 첫 mount": `raw/30-quota.md`)

(원본 출력 "quota inode의 위치와 extent": `raw/30-quota.md`)

(원본 출력 "dquot record 두 개 (xfs_db)": `raw/30-quota.md`)

(원본 출력 "projid와 report (non-metadir fs)": `raw/30-quota.md`)

(원본 출력 "quotacheck가 센 사용량 (non-metadir fs)": `raw/30-quota.md`)

(원본 출력 "quotacheck가 센 사용량 (metadir fs)": `raw/30-quota.md`)

### 원본 출력: quota 끄기

(원본 출력 "state, disable -u, off -u": `raw/30-quota.md`)

### 원본 출력: limit 검사

(원본 출력 "xattr block이 사용량에 들어간다": `raw/30-quota.md`)

(원본 출력 "block hard limit과 grace": `raw/30-quota.md`)

(원본 출력 "id 0 limit이 default로 걸린다": `raw/30-quota.md`)

(원본 출력 "project limit에서는 root도 ENOSPC": `raw/30-quota.md`)

(원본 출력 "limit이 disk의 dquot에 들어간 모양": `raw/30-quota.md`)

### 원본 출력: dquot와 transaction

(원본 출력 "create부터 freeze까지 tracepoint": `raw/30-quota.md`)

(원본 출력 "shutdown 뒤의 DQUOT item과 recovery": `raw/30-quota.md`)
