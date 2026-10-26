# code를 읽기 위한 최소 모형

> XFS 전반 wiki, 묶음 "기초". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/03-code-model.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/03-code-model.md, 접힌 원본 출력은 raw/03-code-model.md.

## 개요

- XFS code에서는 mount된 file system 하나를 `xfs_mount`로 나타내고, AG(allocation group)마다 하나씩 있는 `xfs_perag`는 그 안의 group 표에서 번호로 찾는다. in-core inode인 `xfs_inode`에는 AG를 가리키는 pointer가 없어서, AG는 inode number에서 계산해 찾는다.
- XFS에서는 inode number에 AG 번호, AG 안 block, block 안 위치가 bit로 들어 있어서 inode number가 곧 disk 위치가 된다. fsblock도 AG 번호를 위쪽 bit에 둔다. AG마다 2의 거듭제곱 크기의 번호 칸을 잡으므로, AG 크기가 2의 거듭제곱이 아니면 fsblock과 daddr(512 byte 단위 disk 주소)는 shift 하나로 바꿀 수 없다.
- 함수 인자의 `tp`는 진행 중인 transaction이고, 그것으로 읽은 buffer 가운데 고친 것은 commit이나 cancel까지 잠긴 채 남는다. inode lock 셋은 IOLOCK, MMAPLOCK, ILOCK 순으로 잡고, ILOCK만 transaction을 만든 뒤에 잡는다.

예의 값은 AG 크기가 2의 거듭제곱이 아닌 1,025,000 KiB file system, option 없이 만든 1 GiB file system, 공통 예제 image에서 얻었다.

## 구조

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### mount, group, inode의 관계

`xfs_mount`에서 AG로는 group 표를 번호로 찾아 내려가고, AG마다 둔 inode cache에서 `xfs_inode`를 찾는다. 거꾸로 올라갈 때는 구조마다 든 mount pointer를 쓴다.

**도식: mount, AG, inode 사이의 pointer**

```text
super_block --s_fs_info--> xfs_mount (mp) --m_super--> super_block
                               |
                               | m_groups[XG_TYPE_AG].xa (key = agno)
                               v
                           xfs_perag (pag)
                             pag_group = xfs_group
                               xg_mount -> mp, xg_gno
                               xg_ref (passive), xg_active_ref (active)
                             pag_ici_root (key = agino)
                               |
                               v
VFS inode <--VFS_I / XFS_I--> xfs_inode (ip)
                             i_mount -> mp
                             i_ino (AG는 여기서 계산한다)
```

| 구조 | 하나가 나타내는 것 | 주요 field |
|---|---|---|
| `struct xfs_mount` | mount된 file system 하나 | in-core superblock 사본 `m_sb`, VFS superblock `m_super`, log `m_log`, data device `m_ddev_targp`, root directory inode `m_rootip`, group 표 `m_groups` |
| `struct xfs_groups` | group type 하나(AG, 또는 realtime device를 나눈 rtgroup)의 그릇 | 번호로 찾는 xarray `xa`, geometry `blocks`, `blklog` |
| `struct xfs_group` | AG와 rtgroup의 공통 머리 | mount pointer `xg_mount`, 번호 `xg_gno`, reference count `xg_ref`(passive)와 `xg_active_ref`(active) |
| `struct xfs_perag` | AG 하나의 in-core 상태 | 첫 field `pag_group`, AGF와 AGI에서 읽어 둔 `pagf_freeblks` 등, inode cache `pag_ici_root`(radix tree, key는 agino) |
| `struct xfs_inode` | in-core inode 하나 | mount pointer `i_mount`, 절대 inode number `i_ino`, ILOCK인 `i_lock`, 안에 품은 VFS inode `i_vnode` |

- mount는 superblock 값으로 block과 512 byte 단위(BB) 사이의 shift(`m_blkbb_log`) 같은 값과 AG geometry를 계산하고, superblock의 `sb_rootino`로 `xfs_iget`을 불러 `m_rootip`을 채운다.
- group은 mount 때 AG마다 xarray에 등록하고(growfs와 그 log recovery에서는 새 AG만), 찾을 때는 RCU 안에서 `xa_load`로 찾는다.

| 가진 것 | 얻는 것 | code |
|---|---|---|
| `struct super_block *sb` | mount | `XFS_M(sb)` |
| VFS `struct inode *` | xfs inode | `XFS_I(inode)` |
| xfs inode | mount | `ip->i_mount` |
| mount, AG 번호 | perag (passive) | `xfs_perag_get` → `xfs_group_get` |
| mount, AG 번호 | perag (active) | `xfs_perag_grab` → `xfs_group_grab` |
| mount, 모든 AG | perag 하나씩 | `while ((pag = xfs_perag_next(mp, pag)))` |
| inode number | perag | `xfs_perag_get(mp, XFS_INO_TO_AGNO(mp, ino))` |
| fsblock 번호 | group | `xfs_group_get_by_fsb` |
| group | perag | `to_perag(xg)` (perag의 첫 field라 `container_of`) |
| perag | mount, AG 번호 | `pag_mount(pag)`, `pag_agno(pag)` |
| transaction | mount | `tp->t_mountp` |
| buffer | mount, transaction | `bp->b_mount`, `bp->b_transp` |

함수 인자 이름도 이 표대로 굳어 있다. `mp`는 mount, `pag`는 perag, `xg`는 group, `ip`는 xfs inode, `tp`는 transaction, `bp`는 buffer를 가리킨다.

code: xfs_mount.h, xfs_group.h, xfs_ag.h, xfs_inode.h, xfs_super.c, xfs_group.c

### passive와 active reference

group을 붙잡는 reference는 두 종류가 있고, count를 올리는 방법이 다르다. passive는 늘 올리고, active는 count가 0이면 올리지 않는다(`atomic_inc_not_zero`). mount가 group마다 active reference를 하나 들고 있다가 group을 지울 때 놓는다. shrink나 offline 중인 group은 active로 찾으면 NULL이 나온다.

|  | passive | active |
|---|---|---|
| 함수 | `xfs_group_get`, `xfs_group_hold`, `xfs_group_put` | `xfs_group_grab`, `xfs_group_rele` |
| 드는 쪽 | group을 오래 붙잡아 두는 쪽(cache된 buffer 등). group을 지울 때 함께 정리한다 | tree를 걷거나 상태를 볼 때 짧게 쓰는 쪽 |
| 쓰는 곳 | `xfs_iget`, buffer 찾기(`xfs_buftarg_get_pag`가 daddr에서 AG를 계산한다), mount가 모든 AG의 AGF와 AGI를 읽는 loop, AG geometry ioctl | AG를 차례로 도는 helper(`xfs_group_next_range`, `xfs_perag_grab_next_tag`), inode 할당(`xfs_dialloc`) |

trace에서도 file 하나를 만들 때 inode 할당이 AG 0을 active로, 이어서 `xfs_iget`이 passive로 잡았다 놓았다. `fsmap`은 AG 0을 rele한 바로 뒤에 AG 1을 grab하는 식으로 AG 3까지 갔고, buffer를 하나 찾을 때마다 `xfs_group_get`과 `xfs_group_put`이 한 쌍씩 찍혔다.

code: xfs_group.c, xfs_ag.h, xfs_icache.c, xfs_buf.c, xfs_ialloc.c

### 주소 type

XFS code에서는 주소를 네 가지 type(agblock, fsblock, daddr, inode number)으로 나누어 쓰고, 그 밖의 번호에도 폭과 뜻이 정해진 type이 있다.

| type | 폭 | 뜻 |
|---|---|---|
| `xfs_agnumber_t` | u32 | AG 번호 |
| `xfs_agblock_t` | u32 | AG 안의 block 번호(agbno) |
| `xfs_fsblock_t` | u64 | file system block 번호(fsbno). agno와 agbno를 bit로 붙인 것 |
| `xfs_rfsblock_t` | u64 | 붙이지 않은 raw block 번호(개수, 크기) |
| `xfs_daddr_t` | s64 | device의 512 byte 단위 주소(basic block, BB) |
| `xfs_ino_t` | u64 | 절대 inode number |
| `xfs_agino_t` | u32 | AG 안 inode number(agino) |
| `xfs_fileoff_t` | u64 | file 안의 block offset |
| `xfs_rtblock_t` | u64 | realtime device(rt device)의 block |

NULL 값으로는 모든 bit가 1인 값을 쓴다(`NULLFSBLOCK` 등). byte와 BB 사이는 `BTOBB`, `BBTOB`로 바꾼다.

네 type 사이를 바꾸는 계산에는 superblock 값 넷이 들어간다. 예에 쓴 두 image에서는 값이 이렇다.

| superblock 값 | 뜻 | 공통 예제 image | 1,025,000 KiB image |
|---|---|---|---|
| `sb_blocklog` | block 크기의 log2 | 12 | 12 |
| `sb_agblocks` | AG의 block 수 | 131072 | 64063 |
| `sb_agblklog` | agblocks를 올림한 log2 | 17 | 16 (65536으로 올림) |
| `sb_inopblog` | block 안 inode 수의 log2 | 3 | 3 |
| AG마다 생기는 빈 번호 |  | 없음(agblocks가 2^17) | 1473개(65536 - 64063) |

**도식: fsblock과 inode number의 bit 구성**

```text
1,025,000 KiB image (agblklog 16, inopblog 3)

fsblock 65552
  +------------+---------------------+
  | agno 1     | agbno 16            |
  +------------+---------------------+
    bit 16 위     bit 15~0

inode number 524417
  +------------+---------------------+-----------+
  | agno 1     | agbno 16            | 위치 1    |
  +------------+---------------------+-----------+
    bit 19 위     bit 18~3              bit 2~0
               |<----- agino 129 (19 bit) ------>|
```

### fsblock과 daddr

AG 크기가 2의 거듭제곱이 아니면 fsblock과 daddr는 shift 하나로 바꿀 수 없다. fsblock은 AG마다 2^agblklog개의 번호 칸을 잡으므로 AG가 그보다 작으면 AG 끝에 빈 번호가 남고, daddr는 AG를 빈틈없이 이어 센다. 1,025,000 KiB image의 AG 경계에서는 두 주소가 이렇게 갈린다.

| 자리 | fsblock | daddr |
|---|---|---|
| AG 0의 마지막 block(agbno 64062) | 64062 | 512496 |
| 빈 번호 | 64063~65535(1473개) | 없음 |
| AG 1의 block 0 | 65536 | 512504 |
| AG 3의 block 0 | 196608(3 << 16) | 1537512(3 × 64063 × 8) |

AG 경계에서 fsblock은 1474 뛰지만 daddr는 block 하나(8 BB)만 나아간다. 빈 번호가 없는 공통 예제 image에서는 fsblock × 8이 그대로 daddr가 되어, AG 2의 block 1은 fsblock 262145, daddr 2097160(262145 × 8)이다.

- fsblock은 AG 번호를 `sb_agblklog` bit만큼 올리고 그 아래에 agbno를 붙여 만들고(`XFS_AGB_TO_FSB`), 거꾸로 풀 때는 shift와 mask를 쓴다. daddr는 `agno * sb_agblocks + agbno`를 BB로 바꿔 얻는다(`XFS_AGB_TO_DADDR`). perag가 있으면 `xfs_agbno_to_fsb(pag, agbno)`, `xfs_agbno_to_daddr(pag, agbno)`를 쓴다.
- 그래서 `XFS_FSB_TO_DADDR`는 AG 크기와 관계없이 fsblock을 agno와 agbno로 풀었다가 다시 묶고, 거꾸로 daddr의 AG는 `sb_agblocks`로 나누어 얻는다.
- AG header 같은 고정 자리는 AG 시작 daddr에 sector 단위 offset을 더해 얻는다. 배치는 [on-disk 배치](04-on-disk-layout.md)에 있다.
- `xfs_fsb_to_db`는 realtime inode면 `xfs_rtb_to_daddr`를, 아니면 `XFS_FSB_TO_DADDR`를 쓰고, iomap에는 이 daddr를 byte로 바꿔 넘긴다.
- metadata buffer cache는 daddr를 key로 찾으므로, metadata를 읽는 code는 마지막에 늘 daddr를 만든다.

code: xfs_format.h, xfs_mount.h, xfs_bmap_util.c, xfs_buf.c

### inode number

XFS에서는 inode number가 곧 disk 위치가 된다. 번호를 풀기만 하면 inode가 든 block이 나오므로, AGI에는 inode 위치를 적은 표가 따로 없다. inode number의 아래 `sb_inopblog + sb_agblklog` bit에는 agino가, 그 위에는 AG 번호가 들어간다. AG 크기가 2의 거듭제곱이 아니면 inode number에도 fsblock과 같은 빈 번호가 생긴다.

1,025,000 KiB image의 `d2/f`(inode 524417)를 풀면 다음과 같다. 관찰한 값도 같았다.

| 단계 | 계산 | 결과 | code |
|---|---|---|---|
| agino의 폭 | 3 + 16 | 19 bit | `XFS_INO_AGINO_BITS` |
| AG 번호 | 524417 >> 19 | AG 1 | `XFS_INO_TO_AGNO` |
| agino | 524417 - (1 << 19) | 129 | `XFS_INO_TO_AGINO` |
| AG 안 block | 129 >> 3 | block 16 | `XFS_INO_TO_AGBNO` |
| block 안 위치 | 129 & 7 | 1번(0부터 셈) | `XFS_INO_TO_OFFSET` |
| fsblock | (1 << 16) + 16 | 65552 | `XFS_INO_TO_FSB` |
| inode가 든 block의 daddr | (64063 + 16) × 8. 65552 × 8이 아니다 | 512632. 이 block이 inode cluster의 첫 block이라 inode buffer도 여기서 시작한다 | `XFS_AGB_TO_DADDR` |
| inode의 daddr | 512632 + 1 × 512 / 512 (inode 하나가 512 byte = 1 BB) | 512633 | - |
| disk byte | 512633 × 512 | 262468096 | `BBTOB` |

같은 image의 `d1/f`~`d4/f`는 AG 0~3에 하나씩 놓였고, 번호는 AG n이면 n << 19에 agino를 더한 값이었다.

- inode number에서 disk 위치는 `xfs_imap`이 구한다. inode cluster(inode를 한 번에 읽는 buffer)가 block 하나면 그 block이 buffer가 되고, 아니면 inode chunk의 시작을 정렬 mask나 inobt 조회로 찾아 cluster 시작 block을 계산한다. 결과 `struct xfs_imap`의 `im_blkno`는 daddr, `im_len`은 BB 수, `im_boffset`은 buffer 안 byte offset이다.
- inode cluster 크기는 기본 8192 byte이고, v5에서 inode 정렬이 허락하면 `inodesize / 256`을 곱한다. 512 byte inode와 4 KiB block이면 8192 × 2 = 16384 byte, block 4개(32 BB)가 된다. `stack`으로 본 inode buffer도 어느 inode든 32 BB였고, kernel도 file을 만들 때 inode cluster를 32 BB로 읽었다.
- 절대 번호의 최대는 `XFS_MAXINUMBER`(56 bit)다. `inode32` mount option을 주었고 마지막 inode number가 32 bit를 넘으면 inode를 앞 AG에만 둔다. inode chunk와 할당 정책은 [inode](06-inode.md)에 있다.

code: xfs_format.h, xfs_ialloc.c, xfs_inode_buf.h, xfs_super.c

### on-disk pointer의 type

AG 안 B+tree(bnobt, cntbt, inobt, rmapbt 등)는 pointer로 32 bit agbno를, inode에 root가 있는 bmbt는 64 bit fsblock을 쓴다. extent record의 시작 block도 fsblock이다(disk의 record에서는 52 bit, in-core `struct xfs_bmbt_irec`의 `br_startblock`에서는 64 bit). 읽을 때는 `xfs_btree_ptr_to_daddr`가 B+tree 종류에 맞는 변환으로 daddr를 만든다.

공통 예제 image의 bmbt format file(inode 263)에서 bmbt pointer 7461은 fsblock이고, daddr로는 59688(7461 × 8)이다.

같은 extent를 `xfs_db`의 `bmap`은 fsblock으로, `xfs_io`의 `bmap -v`는 daddr로 낸다. 1,025,000 KiB image의 `d2/f`는 `xfs_db`에서 "startblock 65560 (1/24)"로, `xfs_io`에서 BLOCK-RANGE 512696..512823으로 나온다. 512696은 65560 × 8 = 524480이 아니라 (64063 + 24) × 8이다.

xfs-documentation에 따르면 directory entry와 superblock에는 절대 inode number를, AG 안 구조(inobt record, AGI unlinked bucket)에는 agino를 쓴다. code로는 in-core inode의 unlinked list pointer `i_next_unlinked`가 `xfs_agino_t`인 것만 확인했다.

code: xfs_btree.c, xfs_types.h, xfs_inode.h

### inode number에서 in-core inode까지

위 구조는 `xfs_iget`을 따라가면 한 번에 볼 수 있다.

1. 번호를 검사한다. AG 번호가 범위 안인지, 번호를 다시 조립해 같은지 본다.
2. `XFS_INO_TO_AGNO`로 AG를 구해 `xfs_perag_get`(passive)으로 잡고, `XFS_INO_TO_AGINO`로 agino를 얻는다.
3. 그 AG의 cache `pag_ici_root`에서 agino로 찾는다.
4. 없으면 `xfs_iget_cache_miss`가 새 inode를 만들고 `xfs_imap`으로 disk 위치를 구한다. 이어서 inode cluster buffer를 읽고, on-disk inode를 옮긴 뒤 cache에 넣는다.
5. passive reference를 놓는다.

주석에 따르면 `xfs_iget`은 lookup 중에 ILOCK만 잡게 되어 있고, IOLOCK을 넘기면 ASSERT에 걸린다.

code: xfs_icache.c

## code 경로

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### tp: 진행 중인 transaction

함수 인자로 `tp`를 넘기면, 같은 transaction 안에서 이미 잠근 buffer를 다시 찾을 수 있고 고친 buffer를 commit까지 붙잡아 둘 수 있다. `struct xfs_trans`에는 다음이 있다.

- log reservation `t_log_res`와 block 예약 `t_blk_res`
- mount를 가리키는 `t_mountp`
- 이 transaction에 붙은 log item 목록 `t_items`. buffer와 inode가 여기 붙는다.
- deferred operation 목록 `t_dfops`
- 이 transaction이 잠근 AGF 중 가장 높은 번호 `t_highest_agno`

transaction은 log 공간과 block을 예약하는 `xfs_trans_alloc`이나, 예약 없이 읽기만 하는 `xfs_trans_alloc_empty`로 만든다. `xfs_trans_alloc`을 감싼 wrapper로는 inode 하나를 잠가 붙이는 `xfs_trans_alloc_inode`와 `xfs_trans_alloc_ichange`, inode 둘을 잠그는 `xfs_trans_alloc_dir`, 새 inode의 quota만 예약하고 inode lock은 잡지 않는 `xfs_trans_alloc_icreate`가 있다. 끝낼 때는 `xfs_trans_commit`이나 `xfs_trans_cancel`을 부른다.

kernel 문서(xfs-delayed-logging-design.rst)에 따르면 한 번짜리 transaction은 alloc, lock, join, 수정, commit 순서로 가고, commit하면 붙은 자원이 풀린다. ENOSPC 때 다시 예약하기와, 긴 작업을 이어지는 transaction으로 나누면서 앞 것을 commit하는 roll은 [transaction](11-transaction.md)에 있다.

빈 transaction에서는 붙은 block을 다시 잡을 수 있어서, cross-link된 B+tree를 읽다가 같은 block을 두 번 잡아도 deadlock하지 않고 verifier가 잡아낸다. 빈 transaction은 dirty로 만들지 않고 cancel해야 한다.

code: xfs_trans.h, xfs_trans.c

### xfs_trans_read_buf

`xfs_trans_read_buf`는 같은 transaction에 이미 붙은 buffer가 있으면 그것을 다시 쓰고, 없으면 읽어서 잠근 채 transaction에 붙인다. 고치지 않은 buffer는 `xfs_trans_brelse`로 먼저 놓을 수 있고, 고친 buffer는 commit 때 놓인다.

새 1 GiB file system의 빈 file에 `falloc 0 1m`을 하면, 할당 transaction(`xfs_trans_alloc_inode`)이 처음 roll할 때까지 AG 0의 buffer 넷을 이렇게 다룬다.

| daddr | 이 transaction에서 | 놓인 때 |
|---|---|---|
| 1(AGF) | 고침 | roll의 commit 뒤 |
| 3(AGFL) | 고치지 않음 | 읽은 바로 뒤의 `xfs_trans_brelse` |
| 16(AG 0의 block 2, cntbt root 자리로 추정) | 고침 | `xfs_trans_brelse`를 불러도 남았다가 commit 뒤 |
| 8(AG 0의 block 1, bnobt root 자리로 추정) | 고침 | `xfs_trans_brelse`를 불러도 남았다가 commit 뒤 |

본체 `xfs_trans_read_buf_map`은 다음 순서로 일한다.

1. `tp`가 있으면 `t_items`에서 같은 device, 같은 daddr, 같은 길이의 buffer를 찾는다.
2. 있으면 다시 읽지도, lock을 다시 잡지도 않고 recursion count(`bli_recur`)만 올린다. 이런 buffer에는 보통 verifier가 이미 붙어 있어서 다시 검사하지 않는다(관찰하지 않음).
3. 없으면 `xfs_buf_read_map`으로 buffer cache에서 찾거나 disk에서 읽고, buffer를 잠근 채 받는다. 새로 읽은 buffer는 I/O 완료 때 verifier가 검사한다.
4. 읽기 오류인데 transaction이 이미 dirty면 shutdown한다(고친 것을 되돌릴 수 없기 때문으로 추정).
5. `tp`가 있으면 buf log item을 만들어 `t_items`에 넣고 `bp->b_transp`에 `tp`를 적는다. `tp`가 NULL이면 붙이지 않고, 잠긴 buffer를 받은 호출자가 `xfs_buf_relse`로 놓는다.

- `xfs_trans_brelse`는 tp가 NULL이면 `xfs_buf_relse`와 같고, recursion count가 남았으면 줄이기만 한다. 이 transaction에서 dirty하거나 stale(무효로 표시됨)인 buffer는 놓지 않고, 깨끗한 buffer만 transaction에서 떼고 놓는다.
- 고쳤다는 표시는 `xfs_trans_log_buf`에 byte 범위를 주어 한다(AGF는 `xfs_alloc_log_agf`가 field 단위로 계산한다). commit하면 dirty transaction은 `xlog_cil_commit`에서 item마다 `iop_committing`을 부른다. buffer item은 여기서 `b_transp`를 지우고 buffer lock을 푼다(hold(commit 뒤에도 호출자가 잡아 두겠다는 표시)나 stale이면 풀지 않는다). dirty가 아닌 transaction의 commit과 cancel도 같은 release를 부른다.
- inode도 같다. `xfs_trans_ijoin`에 lock flag를 주면 commit 때 그 lock을 풀어 주고, 0을 주면 호출자가 직접 푼다.

verifier가 붙는 때, 오류 값, buffer cache는 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.

code: xfs_trans_buf.c, xfs_buf.c, xfs_buf_item.c, xfs_inode_item.c, xfs_trans_inode.c

### tp의 세 경우

`tp`는 NULL, 빈 transaction, 보통 transaction 가운데 하나이고, 경우마다 읽은 buffer가 놓이는 때가 다르다.

| tp | 쓰는 곳 | 읽은 buffer | 놓는 때 |
|---|---|---|---|
| NULL | 깨끗하지 않게 내려간 뒤 mount가 summary counter를 다시 셀 때(lazysbcount)의 AGF 읽기, AG geometry ioctl의 AGI와 AGF 읽기, mount의 root inode 읽기 | transaction에 붙지 않는다 | 호출자의 `xfs_buf_relse` |
| 빈 transaction | bulkstat, fsmap, online fsck | 붙는다. 같은 block을 다시 찾으면 recursion | 고치지 않으므로 `xfs_trans_brelse` 때. 그때까지 놓지 않은 것은 `xfs_trans_cancel` 때 |
| 보통 transaction | 할당, 해제, 이름 바꾸기 등 수정 | 붙는다. 이 transaction에서 고치면 commit까지 잠긴다 | 고치지 않았으면 `xfs_trans_brelse` 때, 고쳤거나 그때까지 남은 것은 commit 또는 cancel 때 |

trace에서도 세 경우가 갈렸다.

- AG geometry ioctl(`xfs_io`의 `aginfo -a 1`)은 AG 1의 AGI(daddr 524290)와 AGF(524289)를 `xfs_trans_read_buf` 줄 없이 읽었다.
- file을 만드는 transaction은 AGI(daddr 2)와 inode btree block 둘(daddr 24, 32)을 고쳤고, 셋 모두 commit 뒤에 놓였다.
- `fsmap -v`는 AG마다 AGF와 block 하나를 빈 transaction에 붙여 읽고 `xfs_trans_brelse`로 놓은 뒤, 끝에 `xfs_trans_cancel`을 불렀다.

code: xfs_ag.c, xfs_fsmap.c, xfs_itable.c, xfs_mount.c

### metadata를 읽는 code의 모양

metadata를 읽는 code는 대개 모양이 같다. daddr를 만들어 `tp`, verifier와 함께 넘겨 읽고, 오류면 health를 표시한다. 읽은 내용은 on-disk struct로 보고 big-endian으로 읽는다. AGF를 읽는 `xfs_read_agf`가 그 전형이다.

**xfs_read_agf의 읽기 부분 (주석은 설명용)**

```c
error = xfs_trans_read_buf(mp, tp, mp->m_ddev_targp,
        XFS_AG_DADDR(mp, pag_agno(pag), XFS_AGF_DADDR(mp)),   /* 주소: daddr */
        XFS_FSS_TO_BB(mp, 1), flags, agfbpp, &xfs_agf_buf_ops); /* 길이: BB, verifier */
if (xfs_metadata_is_sick(error))
        xfs_ag_mark_sick(pag, XFS_SICK_AG_AGF);                 /* health 표시 */
if (error)
        return error;
xfs_buf_set_ref(*agfbpp, XFS_AGF_REF);                          /* LRU 우선순위 */
```

- verifier는 `struct xfs_buf_ops`(`verify_read`, `verify_write`)로 나타내고, AGF에는 `xfs_agf_buf_ops`를 준다. inode cluster를 읽는 `xfs_imap_to_bp`와 B+tree block을 읽는 `xfs_btree_read_buf_block`도 같은 모양이다.
- 읽은 내용은 `bp->b_addr`에 있다. 읽기만 했으면 `xfs_trans_brelse(tp, bp)`를 부르고, 고쳤으면 `xfs_trans_log_buf` 계열을 부르고 놓지 않는다.
- AGF와 AGI를 읽는 helper는 읽기만 하는 것(`xfs_read_agf`, `xfs_read_agi`)과, 읽은 뒤 perag 값을 처음 채우는 것(`xfs_alloc_read_agf`, `xfs_ialloc_read_agi`)으로 나뉜다. 그래서 trace에는 두 event가 짝으로 찍힌다.

code: xfs_alloc.c, xfs_ialloc.c, xfs_inode_buf.c, xfs_btree.c

### inode lock 셋

XFS inode 하나에는 lock이 셋 있고, 지키는 것이 서로 다르다.

| XFS 이름 | 실체 | 지키는 것 |
|---|---|---|
| IOLOCK | VFS inode의 `i_rwsem` | file I/O 사이, 그리고 I/O와 크기 변경 사이 |
| MMAPLOCK | VFS `address_space`의 `invalidate_lock` | page fault와, page cache를 무효화하는 extent 조작(truncate, hole punch 등) 사이 |
| ILOCK | xfs inode의 `i_lock`(rw_semaphore) | inode metadata, 특히 extent map(fork) |

- 셋 다 `xfs_ilock`에 `XFS_IOLOCK_EXCL`, `XFS_ILOCK_SHARED` 같은 flag를 주어 잡는다. 같은 lock의 SHARED와 EXCL은 함께 줄 수 없다. flag를 여럿 주면 IOLOCK, MMAPLOCK, ILOCK 순서로 잡고, `xfs_iunlock`도 같은 순서로 푼다.
- 기다리지 않고 잡을 때는 `xfs_ilock_nowait`을, EXCL을 SHARED로 내릴 때는 `xfs_ilock_demote`를 쓴다.
- IOLOCK과 MMAPLOCK은 VFS의 lock이라 VFS가 XFS를 부르기 전에 직접 잡기도 한다. truncate에서는 VFS가 `i_rwsem`을 잡고 `notify_change`를 부른다. DAX가 아닌 file의 read fault에서는 XFS가 MMAPLOCK을 잡지 않고 `filemap_fault`를 바로 부른다. 이렇게 잡은 lock은 `xfs_ilock` tracepoint에 나오지 않는다.
- `xfs_ilock` 위 주석에 따르면 read(2)와 write(2)는 copy 중에 page fault를 내거나 `get_user_pages()`로 mmap_lock을 잡으므로, invalidate_lock을 syscall I/O 내내 들 수 없다. 거꾸로 fault는 이미 mmap_lock을 들고 있어 i_rwsem을 잡을 수 없다. 그래서 IOLOCK 하나로는 모자라고, 둘 다 막아야 하는 truncate와 hole punch만 두 lock을 다 잡는다. 크기를 바꾸는 code는 두 lock을 모두 EXCL로 잡았는지 assert한다.
- extent를 읽으려면 ILOCK이, inode를 transaction에 붙여 log하려면 ILOCK_EXCL이 있어야 한다. btree format file의 extent를 disk에서 처음 읽을 때도 EXCL이 필요해서, 읽기용 helper `xfs_ilock_data_map_shared`가 SHARED와 EXCL 가운데 고른다.
- ILOCK이 inode의 모든 field를 지키지는 않는다. `i_flags`는 spinlock `i_flags_lock`이 지키고, `i_next_unlinked`는 AGI buffer lock 아래에서만 바꾼다.

I/O 경로가 IOLOCK과 ILOCK을 쓰는 방식은 [iomap](17-iomap.md)에 있다.

code: xfs_inode.c, xfs_inode.h, xfs_file.c, xfs_iops.c, xfs_bmap.c, xfs_trans_inode.c

### 잡는 순서

inode lock은 IOLOCK, MMAPLOCK, ILOCK 순으로 잡는다. transaction과 함께 쓸 때는 IOLOCK과 MMAPLOCK을 transaction을 만들기 전에 잡고, ILOCK은 만든 뒤에 잡는다. code 주석에 따르면 기본 순서는 `i_rwsem -> invalidate_lock -> page_lock -> i_ilock`이고, kernel 문서(xfs-online-fsck-design.rst)에도 IOLOCK과 MMAPLOCK은 transaction보다 먼저 잡아야 한다고 되어 있다.

기본 1 GiB image에서 64 KiB file을 `truncate 32k`로 줄이면 다음 순서가 된다. 순서 1은 code로 확인했고, 2~7은 trace에 나온다.

| 순서 | 일 | caller |
|---|---|---|
| 1 | VFS가 `i_rwsem`(IOLOCK)을 잡는다. trace에는 없다 | VFS |
| 2 | MMAPLOCK_EXCL | `xfs_vn_setattr` |
| 3 | `xfs_trans_alloc` | `xfs_vn_setattr_size` |
| 4 | ILOCK_EXCL | `xfs_vn_setattr_size` |
| 5 | commit 네 번. 그중 세 번은 roll | `xfs_trans_roll`, `xfs_trans_commit` |
| 6 | ILOCK_EXCL을 푼다 | `xfs_vn_setattr_size` |
| 7 | MMAPLOCK_EXCL을 푼다 | `xfs_vn_setattr` |

- code로는 2와 3 사이에 page cache를 정리한다(`truncate_setsize`).
- fallocate는 IOLOCK과 MMAPLOCK을 한 호출로 잡고(trace에 `IOLOCK_EXCL|MMAPLOCK_EXCL` 한 줄), `xfs_trans_alloc_inode`가 transaction을 만든 뒤 ILOCK을 잡는다.
- page fault 쪽에서는 mmap_lock, `sb_start_pagefault`, invalidate_lock(MMAPLOCK), page lock, i_lock(ILOCK) 순으로 잡는다(주석).
- transaction 안에서 잡은 ILOCK은 두 방식으로 풀렸다. timestamp 갱신(`xfs_vn_update_time`)의 transaction에서는 commit 중에 `xlog_cil_commit`이 풀었고, truncate, fallocate, rename에서는 commit 뒤에 호출자가 직접 풀었다.

code: xfs_inode.c, xfs_iops.c, xfs_file.c, xfs_bmap_util.c, fs/open.c

### 여러 inode를 잡을 때

같은 종류의 lock을 inode 여럿에 잡을 때는 lock마다 정해진 순서를 따르고, ILOCK은 inode number 순으로 잡는다. 순서는 kernel 문서(xfs-online-fsck-design.rst)에 있고, 오른쪽 열의 정렬 함수는 code로 확인했다.

| lock | 순서 | 정렬하는 code |
|---|---|---|
| IOLOCK | directory를 갱신할 때는 parent → child(VFS가 잡는다, code로 확인하지 않음), 그 밖에는 `struct inode`의 주소 순 | `xfs_iolock_two_inodes_and_break_layout` |
| MMAPLOCK | `struct address_space`의 주소 순 | `filemap_invalidate_lock_two` |
| ILOCK | inode number 순 | `xfs_lock_two_inodes`, rename은 `xfs_sort_inodes` 뒤 `xfs_lock_inodes` |

이미 있는 `b` 위로 `mv a b`를 하면, `xfs_rename`이 transaction을 만든 뒤 `xfs_lock_inodes`가 inode 셋의 ILOCK_EXCL을 번호 순으로 잡는다. IOLOCK 줄은 없었다.

| 순서 | inode | 잡은 함수 | flag에 붙은 값 | lockdep subclass |
|---|---|---|---|---|
| 1 | directory(128) | `xfs_ilock` | 없음 | 0 |
| 2 | `a`(132) | `xfs_ilock_nowait` | `0x2000000` | 2 |
| 3 | `b`(133) | `xfs_ilock_nowait` | `0x3000000` | 3 |

- `xfs_lock_inodes`는 inode 2~5개를 EXCL로 잡고, 한 번에 한 종류의 lock만 잡는다. subclass를 두는 bit 자리는 lock마다 다르다(`XFS_IOLOCK_SHIFT` = 16, `XFS_MMAPLOCK_SHIFT` = 20, `XFS_ILOCK_SHIFT` = 24). 위 표의 subclass는 flag 값을 24 bit 내린 값이다.
- lockdep subclass로는 정렬한 배열의 자리 번호를 쓰고, 같은 inode가 거듭 들어 있으면 건너뛴다. rename은 두 directory와 두 inode를 한 배열에 넣어 정렬하므로, 같은 directory 안의 rename에서는 directory가 두 번 들어간다. 이 예에서는 directory(128)의 번호가 가장 작아 자리 0과 1을 차지하고, `a`와 `b`가 2와 3이 된다.
- 이미 잡은 inode가 AIL(active item list)에 있으면 다음 inode는 trylock으로 잡고, 실패하면 모두 풀고 다시 한다. 주석에 따르면 AIL에 있는 inode를 잡은 채 긴 transaction(truncate 등)이 쥔 inode를 기다리면, 그 transaction이 log tail을 밀려고 우리 inode를 기다릴 수 있다. rename 표의 nowait 두 줄은 directory inode 128이 AIL에 있었기 때문으로 추정한다.
- directory 조작 transaction은 parent와 child의 ILOCK을 transaction을 만든 뒤 함께 잡는다(`xfs_trans_alloc_dir`).

code: xfs_inode.c, xfs_inode.h, xfs_trans.c, mm/filemap.c

### inode lock 다음의 buffer lock

AG header buffer를 잡을 때는 AGI를 AGF보다 먼저 잡고, AGF 여럿은 번호가 오르는 순서로만 잡는다.

- metadata buffer는 semaphore(`b_sema`)로 잠그고, transaction으로 읽은 buffer는 commit까지 잠긴 채 남을 수 있다.
- inode 할당은 AGI를 잡은 뒤 새 extent를 할당하며 AGF를 잡는다. AG geometry ioctl도 AGI(daddr 524290)를 AGF(524289)보다 먼저 읽었다.
- 한 transaction이 잡은 AGF 중 가장 높은 번호를 `t_highest_agno`에 두고, 다음 할당은 그 번호부터 찾는다. 주석에 따르면 AGF는 transaction을 commit할 때까지 놓을 수 없다. transaction 쪽은 [transaction](11-transaction.md)에 있다.

code: xfs_buf.c, xfs_inode.c, xfs_alloc.c

### 경로별 정리

경로마다 잡는 lock은 아래 표와 같다. trace도 표와 같았고, buffered read에서 iomap_begin의 ILOCK 줄이 없었던 것만 달랐다. 표의 timestamp 갱신 transaction은 `xfs_vn_update_time`이 만든다.

| 경로 | IOLOCK | MMAPLOCK | transaction | ILOCK |
|---|---|---|---|---|
| buffered read | SHARED (XFS) | - | timestamp 갱신 transaction 말고는 없음 | iomap_begin에서 SHARED |
| direct read | SHARED (XFS) | - | timestamp 갱신 transaction 말고는 없음 | iomap_begin |
| buffered write | EXCL (XFS) | - | timestamp 갱신 transaction 말고는 없음(delalloc: block 할당을 writeback 때로 미루고, 할당 transaction은 그때 생긴다) | iomap_begin에서 EXCL |
| 정렬된 direct write | SHARED, 필요하면 EXCL로 올렸다가 demote | - | timestamp 갱신 transaction, 할당하면 할당 transaction | iomap_begin |
| mmap write fault | - | SHARED (XFS) | timestamp 갱신 transaction 말고는 없음 | iomap_begin |
| truncate | EXCL (VFS가 잡아 trace에 없다) | EXCL (XFS) | 있음 | transaction 뒤 EXCL |
| fallocate | EXCL (XFS) | EXCL (XFS, 같은 호출) | 있음 | transaction 뒤 EXCL |
| rename | VFS가 잡는다 | - | 있음 | transaction 뒤 EXCL, inode number 순, subclass 증가 |

- 정렬되지 않은 direct write는 EOF에 닿거나 넘으면 처음부터 EXCL로 잡는다. 나머지는 SHARED로 overwrite만 시도하다가, 할당이나 block 일부 zeroing이 필요해 -EAGAIN이 나면 EXCL로 다시 한다.
- write fault는 reflink remap 중이면 MMAPLOCK을 EXCL로 바꿔 기다린다. DAX read fault도 MMAPLOCK을 SHARED로 잡는다.
- close 때 EOF 너머 preallocation을 정리하는 쪽은 `xfs_ilock_nowait`으로 IOLOCK_EXCL을 시도한다.

code: xfs_file.c, xfs_iops.c, xfs_iomap.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| xfs-documentation common_types.asciidoc | "All the following XFS types can be found in xfs_types.h" | kernel에서 `xfs_ino_t`, `xfs_daddr_t`, `xfs_off_t`는 xfs_platform.h에 있다 |
| 같은 문서 | `xfs_extnum_t` "Signed 32 bit", `xfs_aextnum_t` "Signed 16 bit" | 옛 정의다. 지금은 xfs_types.h에서 각각 u64, u32다 |
| allocation_groups.asciidoc | superblock field `sb_inoplog` | field 이름은 `sb_inopblog`다 |
| xfs_inode.c의 `xfs_ilock` 위 주석 | "the xfs inode contains 2 multi-reader locks: invalidate_lock and the i_lock" | `invalidate_lock`은 xfs inode가 아니라 VFS `address_space`에 있다 |

## 한계점

다음은 확인하지 못했다.

- 새 directory가 AG 사이에 퍼지는 방식. 관찰에서는 `d1`~`d4`와 그 안 file이 AG 0~3에 하나씩 갔지만 AG 선택 code는 읽지 않았다. AG 선택은 [inode](06-inode.md)에 있다.
- 1,025,000 KiB image의 AG 2에서만 inode와 data가 block 16이 아니라 16400 근처에 놓인 까닭. 기본 1 GiB image에서는 AG 2 앞부분(AG-OFFSET 56..131127, 512 byte 단위)을 log가 차지하지만, 이 image의 log 자리는 보지 않았다.
- VFS rename이 `i_rwsem`을 잡는 순서(`lock_rename`)와 `filemap_fault`가 `invalidate_lock`을 잡는 때. VFS와 mm code는 읽지 않았다.
- IOLOCK과 MMAPLOCK을 transaction 전에 잡아야 하는 code상의 이유(log 공간 예약이 무엇을 기다리는지). `__xfs_trans_alloc`은 freeze 보호(`sb_start_intwrite`)를 잡고, fault는 MMAPLOCK보다 먼저 `sb_start_pagefault`를 잡는다. 이 둘이 단서다.
- xfsprogs에서 `xfs_ino_t`, `xfs_daddr_t`, `xfs_off_t`를 정의한 자리.
- `xfs_db`의 `stack`이 inode buffer 길이를 kernel의 `xfs_imap`과 같은 방법으로 정하는지. 값(32 BB)은 계산한 inode cluster 크기와 같다.
- buffered read 구간에 iomap_begin의 ILOCK 줄이 없는 까닭(읽기 직전에 같은 file을 쓰고 fsync했다).
- rename 때 directory inode 128이 AIL에 있었는지.
- 같은 transaction에서 같은 buffer를 다시 찾는 recursion 경로. trace에 `xfs_trans_read_buf_recur` 줄이 없었다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- AG 크기가 나누어떨어지지 않는 image는 1,025,000 KiB file에 option 없이 mkfs했다. mkfs 출력대로 AG 4개, AG마다 64063 block, 4 KiB block, 512 byte inode로 만들어졌다. mount한 뒤 directory `d1`~`d4`를 만들고 각각에 64 KiB file `f`를 쓰고 fsync했다.
- tracepoint 관찰은 기본 1 GiB image에서 했다. 이 image에서 AG 1은 daddr 524288에서 시작하고, fsmap 출력에서는 AG 0의 daddr 24~39가 inode btree, 128~191이 inodes로 나온다.

### 관찰 목록

| 관찰 | 절차와 명령 | 켠 event | 본문의 예시 |
|---|---|---|---|
| 공통 예제 image의 주소 | `xfs_db`의 `sb`, `convert`, `agf`, `addr`, `path`, `print`, `stack` | 없음 | superblock 값 표, AG 2의 block 1, bmbt pointer 7461 |
| 1,025,000 KiB image의 주소 | file마다 `stat`으로 inode number를 얻고 `xfs_io`의 `bmap -v`를 본 뒤, unmount하고 `xfs_db`의 `convert`, `inode`, `stack`, `bmap` | 없음 | AG 경계 표, inode 524417 단계표, 같은 extent의 두 단위 |
| group과 buffer 읽기 | `xfs_io -r -c "aginfo -a 1"`, `touch`로 file 하나 만들기, `xfs_io -c "falloc 0 1m"`, `xfs_io -r -c "fsmap -v"` | group reference, AGF와 AGI 읽기, buffer 읽기, transaction, buf log item(`xfs_trans_read_buf_recur` 포함) | passive와 active, falloc 표, tp의 세 경우 |
| inode lock | 64 KiB file `f`를 쓰고 fsync한 뒤 4 KiB buffered read, 4 KiB direct read(`xfs_io -d`), offset 16 KiB에 4 KiB direct write, offset 4 KiB에 4 KiB buffered write, mmap한 뒤 offset 8 KiB에 4 KiB `mwrite`, `truncate 32k`, `falloc 64k 64k`, 이미 있는 `b` 위로 `mv a b` | `xfs_ilock`, `xfs_ilock_nowait`, `xfs_ilock_demote`, `xfs_iunlock`, transaction | truncate 순서표, rename 표, 경로별 정리 |

trace 관찰에서는 구간마다 `trace_marker`에 표시를 쓰고 명령을 실행했다.

### 출력 읽는 법

- group event는 "agno 0x.. passive refs N active refs N caller ..." 꼴로 찍힌다. `xfs_io`는 XFS file을 열 때와 `aginfo`, `fsmap -v`를 준비할 때 file system geometry ioctl(`XFS_IOC_FSGEOMETRY`)을 보낸다. 그 안의 `xfs_fsop_geom_health`가 rtgroup을 하나씩 grab하고 rele하므로, 구간 머리에 rtgno 0x0 줄이 한두 쌍 나온다. 이 줄은 AG geometry ioctl과 관계가 없다.
- group event의 refs는 count를 바꾸기 전 값이다. 그래서 buffer 찾기의 get 줄이 3이면 짝인 put 줄은 4다.
- buffer 읽기는 `xfs_buf_read` 줄로 찍힌다. 그 buffer를 transaction에 붙이면 `xfs_trans_read_buf` 줄이 뒤따르고, 같은 transaction에 이미 붙은 buffer를 다시 찾으면 `xfs_buf_read` 없이 `xfs_trans_read_buf_recur` 줄만 찍힌다. tp가 NULL이어도 `xfs_buf_read`의 caller는 `xfs_trans_read_buf_map`이다.
- `xfs_trans_alloc` 줄은 예약이 성공한 뒤에만 찍히고, 빈 transaction을 만드는 `xfs_trans_alloc_empty`에서는 찍히지 않는다. 그래서 `fsmap -v` 구간에는 이 줄이 없다.
- inode lock event는 "ino 0x.. flags %s caller %pS" 꼴로 찍힌다. lockdep subclass bit(16~31)에는 이름이 없어서, 남은 bit가 `ILOCK_EXCL|0x2000000`처럼 hex로 붙는다.
- direct read, truncate, rename을 뺀 구간마다 `xfs_vn_update_time`의 timestamp transaction이 끼어, `xfs_trans_alloc` 뒤에 ILOCK_EXCL을 잡는다. `xfs_io`로 file을 연 구간에는 끝의 close에서 `xfs_file_release`가 IOLOCK_EXCL을 `xfs_ilock_nowait`으로 잡는 줄이 나온다.

### 원본 출력

#### 공통 예제 image의 주소

(원본 출력 "공통 예제 image의 superblock geometry": `raw/03-code-model.md`)

(원본 출력 "공통 예제 image: AG 2 block의 fsblock과 daddr": `raw/03-code-model.md`)

(원본 출력 "공통 예제 image: root inode를 풀어 본 값과 stack": `raw/03-code-model.md`)

(원본 출력 "공통 예제 image: bmbt pointer의 fsblock과 daddr": `raw/03-code-model.md`)

#### 1,025,000 KiB image의 주소

(원본 출력 "1,025,000 KiB image의 mkfs와 superblock geometry": `raw/03-code-model.md`)

(원본 출력 "1,025,000 KiB image: AG 경계의 fsblock과 daddr": `raw/03-code-model.md`)

(원본 출력 "1,025,000 KiB image: directory와 file의 inode number, convert와 stack": `raw/03-code-model.md`)

(원본 출력 "1,025,000 KiB image: 같은 extent의 xfs_io bmap -v": `raw/03-code-model.md`)

#### group과 buffer 읽기

(원본 출력 "aginfo와 file 생성 구간의 group reference": `raw/03-code-model.md`)

(원본 출력 "fsmap이 AG를 차례로 도는 active reference": `raw/03-code-model.md`)

(원본 출력 "tp가 NULL인 읽기: aginfo -a 1": `raw/03-code-model.md`)

(원본 출력 "보통 transaction: file 생성": `raw/03-code-model.md`)

(원본 출력 "보통 transaction: falloc 0 1m": `raw/03-code-model.md`)

(원본 출력 "빈 transaction: fsmap -v": `raw/03-code-model.md`)

(원본 출력 "1 GiB image의 fsmap 출력": `raw/03-code-model.md`)

#### inode lock

(원본 출력 "read, write, mmap write fault 구간의 inode lock": `raw/03-code-model.md`)

(원본 출력 "truncate, fallocate, rename 구간의 inode lock": `raw/03-code-model.md`)
