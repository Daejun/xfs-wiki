# transaction

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/11-transaction.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/11-transaction.md, 접힌 원본 출력은 raw/11-transaction.md.

## 개요

- XFS에서 metadata는 평소 transaction 안에서 바뀐다. transaction은 시작할 때 log 공간과 쓸 수 있는 free block 수를 미리 잡는다. 끝나면 바뀐 객체를 log item으로 묶어 CIL에 넘긴 뒤 남은 예약을 돌려준다.
- log 공간은 평균이 아니라 최악의 경우만큼 잡는다. 수정을 반쯤 한 채 log에 쓸 자리가 없어 멈추면 되돌릴 방법이 없기 때문이다. 이 최악값은 mount 때 transaction 종류마다 한 번 계산해 둔다.
- 한 transaction에 다 담지 못하는 일은 조각마다 commit하고 예약은 다음 조각으로 이어 간다. extent 1280개인 file의 truncate는 log 예약 한 번으로 roll을 3215번 이어 갔다. 조각 사이에는 쥔 객체를 다시 log해서 log tail을 붙잡지 않게 한다.

transaction handle과 buffer가 붙는 모양은 [code를 읽기 위한 최소 모형](03-code-model.md)에, commit 뒤 변경이 log에 쓰이는 길은 [log](12-log.md)에 있다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. 기본 mkfs가 rmapbt, reflink, parent pointer를 켜므로 write, truncate, 그리고 create나 rename 같은 namespace 변경의 logcount가 header의 기본값보다 크다.

## log 예약

이 장의 동작은 code와 문서로 확인했고, 예의 값은 관찰한 것이다.

### 최악의 경우를 미리 잡는 이유

kernel의 delayed logging 설계 문서에 따르면 reservation은 수정을 시작하기 전에, 그 수정을 journal에 쓸 물리 공간이 있음을 보장한다. B+tree를 고치는 transaction이면 leaf에서 root까지 전부 split하는 경우까지 넣어야 한다. 이렇게 잡아 두면 수정 중간에 log 공간이 없어 멈추는 일이 없다.

### rolling transaction과 permanent reservation

큰 일을 조각마다 commit하는 방식을 rolling transaction이라 하고, 다음 조각으로 넘겨 가는 예약을 permanent reservation이라 한다. 한 번에 끝나는 transaction은 one-shot transaction이라 한다. transaction 종류의 `tr_logflags`에 `XFS_TRANS_PERM_LOG_RES`가 서 있으면 permanent transaction이다. deferred operation은 permanent transaction에서만 쓸 수 있다.

- 조각내는 이유: 한 번에 담으면 reservation이 작업 크기에 비례해 커진다. extent 1,000개를 지우는 truncate를 한 transaction에 담는다면 free space btree를 1,000번 고쳐야 하고, 그 최악값은 log 크기를 넘을 수 있다. 그래서 truncate는 한 번에 extent 2개만 unmap하고 그때마다 deferred 해제를 처리하며 roll한다. 한 조각의 예약(logres)도 이만큼이어서, `tr_itruncate`는 extent 2개 unmap과 extent 4개를 푸는 EFI(extent 해제 intent) 처리만큼만 잡는다.
- 그 대신 chain 전체는 원자적이지 않다. crash하면 마지막으로 log에 닿은 조각까지만 복구되고, 끝까지 해야 하는 일은 intent로 남겨 두었다가 복구 때 이어 간다([deferred operation, intent, log recovery](13-intent-recovery.md)). chain 동안 inode lock을 계속 쥐므로 다른 thread는 중간 상태를 보지 못한다.

### 종류별 예약 표

예약할 크기는 mount 때 log를 열기 전에 `xfs_trans_resv_calc`가 transaction 종류마다 한 번 계산해 `xfs_trans_resv` 표에 둔다. 항목에는 byte 수(`tr_logres`), 몇 조각까지 선불할지(`tr_logcount`), flag(`tr_logflags`)가 들어 있다. 표에는 종류 27개가 있다. tracepoint와 `xfs_db logres`는 표의 순서를 type 번호로 찍는다. 관찰한 file system의 대표 항목은 다음과 같았다.

| type | 종류 | logres(byte) | logcount | permanent |
|---|---|---|---|---|
| 0 | `tr_write` | 142840 | 5 | 예 |
| 1 | `tr_itruncate` | 226176 | 5 | 예 |
| 2 | `tr_rename` | 428160(가장 큼) | 5 | 예 |
| 8 | `tr_mkdir` | 281892 | 6 | 예 |
| 10 | `tr_ichange` | 2168 | 0 | 아니오 |
| 24 | `tr_sb` | 640 | 1 | 아니오 |

27개 가운데 18개가 permanent이고, 나머지 9개(`tr_ichange`, `tr_fsyncts`, `tr_sb` 등)는 one-shot이다. xfsprogs 7.1.1의 `xfs_db logres`는 같은 계산 code로 값을 낸다. 그 27개 값은 kernel이 mount 때 낸 값과 모두 같았다.

### logcount

logcount는 roll 몇 번 분량의 예약을 미리 사 둘지 정한다. 문서의 예를 보면 inode 할당은 보통 transaction 둘로 끝나므로 logcount를 2로 잡는다. 문서에 따르면 permanent transaction은 roll할 때마다 한 조각을 통째로 쓴다. code에서는 roll마다 선불 조각 수(`t_cnt`)가 0이 될 때까지 하나씩 줄고, 그 조각에서 쓰지 않은 byte는 두 grant head에서 빠진다(truncate 한 번의 roll 표). 그래서 처음에 "한 조각 크기 x logcount"를 잡아 두면 흔한 경우 log 예약을 한 번만 한다. 관찰한 file system에서는 header의 기본값에 기능별 몫이 더해졌다.

| 종류 | 기본값 | 더하는 몫 | 결과 |
|---|---|---|---|
| `tr_write`, `tr_itruncate` | 2 | rmapbt와 reflink의 intent 몫 3(bmap, refcount, rmap 하나씩) | 5 |
| create, remove, link, rename | 2 | parent pointer xattr 하나를 넣거나 빼는 몫 3 | 5 |
| symlink, mkdir | 3 | parent pointer xattr 몫 3 | 6 |

intent 몫은 `tr_qm_dqalloc`에도 더한다. parent pointer 몫은 create, mkdir, symlink, link에서는 xattr 설정(`tr_attrsetm`)의 logcount이고, remove에서는 xattr 삭제(`tr_attrrm`)의 logcount다. rename에서는 둘 중 큰 값이다.

### 한 항목의 크기

한 항목은 "buffer 몇 개 x (크기 + header)"로 센다(`xfs_calc_buf_res`). header 몫은 op header와 buf log format을 128 byte 배수로 올려 잡는다. free space btree 몫은 extent 하나를 할당하거나 해제할 때마다 tree 둘에서 각각 (2 x 최대 깊이 - 1) block이 바뀔 수 있다고 보고 센다. rmapbt가 있으면 tree 하나를 더한다.

동시에 일어나지 않는 경우들은 더하지 않고 가장 큰 값 하나만 잡는다. 할당과 그 뒤의 정리는 roll로 나뉜 서로 다른 transaction에서 일어나기 때문이다. 예로 `tr_write`(142840 byte)는 다음 넷 가운데 가장 큰 값에 dquot 6개 몫을 더한다.

- inode 1개, bmbt 높이만큼의 block, sector 3개(AGF 둘과 superblock 하나), extent 둘의 free space btree split
- realtime file일 때
- 할당 뒤 bmbt block을 풀 때
- refcountbt 갱신(지금은 별도 transaction에서 하므로 따로 센다)

`tr_itruncate`는 inode와 bmbt, extent 4개를 푸는 EFI 처리, refcount 둘을 센다.

### ticket과 grant head

`xfs_trans_alloc`은 `xfs_log_reserve`로 log ticket을 만들어 예약을 grant head에 올린다. XFS에는 grant head가 둘 있다. reserve head는 예약한 총량을, write head는 log에 실제로 쓸 수 있게 내준 공간을 센다. ticket에는 다음 값이 들어 있다.

- `t_unit_res`: 한 조각의 예약 byte.
- `t_curr_res`: 지금 조각이 쓸 수 있는 byte.
- `t_cnt`: 남은 선불 조각 수. 처음 값은 `t_ocnt`에 남는다.
- `t_tid`: 처음에는 random이고 roll마다 1씩 오른다.

한 조각 크기는 `tr_logres`에 log 형식의 덧붙임을 더해 정한다. 덧붙임에는 transaction header와 start op header가 들어간다. data가 걸치는 iclog마다(최소 하나) split op header와 log record header가 더해진다. 여기에 commit record의 log record header와 roundoff 두 번이 붙는다. 첫 예약에서 permanent ticket은 "한 조각 x 조각 수"를, one-shot ticket은 한 조각을 reserve head와 write head에 함께 더한다. truncate 한 번의 첫 예약은 이렇게 커졌다.

| 단계 | byte | 계산 |
|---|---|---|
| `tr_itruncate`의 logres | 226176 | 예약 표의 type 1 |
| 한 조각(`t_unit_res`) | 231944 | 226176 + 덧붙임 5768 |
| 첫 예약 | 1159720 | 231944 x 5(logcount) |
| reserve head와 write head | 212992 → 1372712 | 1159720만큼 오름 |

남은 공간이 모자라면 ticket은 AIL(log에 commit됐지만 아직 제자리에 쓰지 않은 item 목록)을 밀어 tail을 움직이게 한 뒤 FIFO 대기열에서 잔다. 앞에 기다리는 ticket이 있으면 그쪽을 먼저 깨운다. 다 깨우지 못했거나 깨운 뒤 남은 공간이 모자라면 새 ticket은 그 뒤에 선다. 남은 공간을 세는 방법은 [log](12-log.md)에 있다.

code: xfs_trans_resv.c, xfs_trans_resv.h, xfs_trans.c, xfs_bmap.c, xfs_log.c, xfs_log_priv.h

## roll 한 번

이 장의 동작은 code와 문서로 확인했고, 예의 값은 관찰한 것이다.

### xfs_trans_roll

`xfs_trans_roll`은 같은 ticket을 쓰는 새 handle을 만들고, 옛 handle을 commit한 뒤, 새 handle의 log 예약을 채운다.

| 단계 | 하는 일 |
|---|---|
| 새 handle(`xfs_trans_dup`) | 같은 ticket을 참조로 공유한다(permanent여야 한다). 남은 block 예약과 realtime extent, deferred op 목록을 새 handle로 옮기고, 옛 handle에는 쓴 만큼만 남긴다. 잠긴 item은 옮기지 않는다. |
| 옛 handle commit | CIL이 쓴 byte를 `t_curr_res`에서 뺀다. checkpoint마다 첫 commit은 checkpoint 몫의 byte도 함께 낸다. |
| (commit 안) `xfs_log_ticket_regrant` | `t_cnt`가 0보다 크면 하나 줄이고, 쓰지 않고 남은 `t_curr_res`를 두 head에서 뺀 뒤 `t_curr_res`를 한 조각으로 되돌린다. 이때 `t_cnt`가 0이면 reserve head에 한 조각을 기다리지 않고 더한다. |
| `xfs_log_regrant` | `t_tid`를 1 올리고 `t_curr_res`를 한 조각으로 채운다. `t_cnt`가 0보다 크면 여기서 끝나고, 0이면 write head에서 한 조각을 받을 때까지 잘 수 있다. |

- 남은 예약이 음수가 되면 "Transaction log reservation overrun"을 내고 shutdown하므로, 예약 계산이 작으면 곧바로 드러난다.
- 쓴 byte는 grant head에서 빼지 않고 checkpoint의 크기로 세었다가, checkpoint가 log에 쓰인 뒤에 돌려준다.
- dirty가 아니어서 CIL로 가지 않는 roll도 같은 regrant를 부른다.
- 옮기지 않은 item은 호출자가 다시 join한다. inode 하나만 쥔 호출자는 `xfs_trans_roll_inode`를 쓴다. deferred operation 처리는 쥔 buffer와 lock flag가 0인 inode를 roll 전에 다시 dirty로 log하고(`xfs_defer_save_resources`), roll 뒤에 다시 붙인다.

### truncate 한 번의 roll

logcount 5인 truncate chain은 처음 다섯 조각을 선불로 쓰고, 다섯 번째 roll부터는 roll마다 한 조각을 새로 받는다. extent 1280개인 file을 `xfs_io`로 truncate했을 때 두 head는 다음과 같이 움직였다.

| 시점 | 지나기 전 `t_cnt` | 이 조각이 쓴 byte | reserve head | write head |
|---|---|---|---|---|
| 예약 전 |  |  | 212992 | 212992 |
| 예약 직후 | 5 |  | 1372712 | 1372712 |
| roll 1 | 5 | 9052 | 1149820 | 1149820 |
| roll 2 | 4 | 8276 | 926152 | 926152 |
| roll 3 | 3 | 15092 | 709300 | 709300 |
| roll 4 | 2 | 4132 | 481488 | 481488 |
| roll 5 | 1 | 4148 | 253692 → 485636 | 253692 → 485636 |
| roll 6 | 0 | 4400 | 258092 → 490036 | 258092 → 490036 |

- roll마다 먼저 ticket이 쓰지 않은 byte를 두 head에서 빼므로 head가 내려간다. 쓴 byte는 checkpoint가 log에 쓰일 때까지 head에 남는다. 그래서 roll 5에서 뺀 직후의 253692는 예약 전 212992에 다섯 조각이 쓴 byte의 합 40700을 더한 값이다.
- roll 5에서 `t_cnt`가 0이 되면 reserve head가 먼저 한 조각(231944)만큼 오르고 write head가 뒤따른다. 그 뒤로는 roll마다 조각이 쓴 byte만큼만 head가 앞으로 간다(roll 6에서 490036 - 485636 = 4400).

chain 하나는 log 예약 1번, roll N번, commit N+1번으로 이루어진다. 이 truncate에서는 roll이 3215번, commit이 3216번이었다. file을 `rm`으로 지울 때는 kworker가 `xfs_inactive_truncate`에서 같은 모양의 chain(`ocnt=5`)을 돌렸다. 두 경우 모두 log 공간을 기다리며 잔 일과 cancel은 없었다. 통계는 다음과 같이 늘었다.

| 시점 | async commit 수 | 예약 시도 수 | 잠든 횟수 | 늘어난 양 | 그 구간의 commit |
|---|---|---|---|---|---|
| truncate 전 | 7716 | 7716 | 0 |  |  |
| truncate 뒤 | 10932 | 10932 | 0 | 3216 | `xfs_io` 3216 |
| `rm` 뒤 | 14153 | 14153 | 0 | 3221 | kworker 3220, `rm` 1 |

roll 안의 commit도 async로 센다. 예약 시도 수에는 `xfs_log_reserve` 1번과 roll마다 부르는 `xfs_log_regrant`가 들어간다. 그래서 모든 commit이 dirty이고 cancel이 없으면 이 수는 async 수와 같은 양만큼 는다. 관찰에서도 두 수는 같은 양만큼 늘었다.

### grant head가 둘인 이유

선불이 바닥난 chain이 roll 도중 log 공간을 기다리며 잔다고 하자. 그동안 chain이 lock을 쥔 item의 옛 변경이 log tail에 있으면, 그 item을 제자리에 쓰지 못해 tail이 움직이지 않고 공간이 영영 나지 않는다. XFS는 문서에 나오는 이 교착을 head 둘과 다시 log하는 규칙으로 피한다.

- reserve head는 물리 공간이 아니라 예약 총량만 센다. 그래서 commit 도중에는 잠들지 않고 overcommit할 수 있다. roll 표에서 reserve head가 먼저 오른 것도 이 때문이다.
- 물리 공간을 세는 write head 몫은 commit이 끝난 뒤 따로 받는다. 이때는 잠들어도 된다.
- 문서에 따르면 permanent reservation을 쓰는 code는 roll할 때마다, lock을 쥔 채 넘기는 item을 반드시 다시 log해야 한다. 다시 log한 item은 log head 쪽으로 옮겨 가므로 tail을 붙잡지 않는다. roll 전의 `xfs_defer_save_resources`가 이 규칙을 지킨다. 오래 남은 intent도 done을 쓰고 새 intent로 다시 log해서 tail을 놓아 준다(`xfs_defer_relog`).

**도식: 다시 log하면 tail이 풀리는 까닭**

```text
log ring: 왼쪽이 오래된 쪽이고, 오른쪽으로 쓴다

다시 log하지 않을 때
  tail                                      write head
  |                                         |
  [ A의 옛 변경 ][ 다른 변경 ...           ][ 남은 공간 ]
  ^
  chain이 A의 lock을 쥔 채 잠들어 A를 제자리에 쓰지 못한다
  -> A의 옛 변경이 tail에 남아 tail이 움직이지 않는다
  -> 남은 공간이 늘지 않아 기다리는 chain이 깨지 않는다

roll마다 A를 다시 log할 때
  tail                                      write head
  |                                         |
  [ 다른 변경 ...           ][ A의 새 사본 ][ 남은 공간 ]
  A의 변경이 head 쪽으로 옮겨 가서 tail이 A의 옛 자리를 지날 수 있다
```

code: xfs_trans.c, xfs_log.c, xfs_log_cil.c, xfs_defer.c, xfs_trans_inode.c

## log item

이 장의 동작은 code로 확인했다.

### 공통 머리와 함수 표

XFS에서는 transaction이 log에 남길 변경 하나를 log item이라는 객체로 나타낸다. buffer, inode, dquot, intent에는 저마다 `xfs_log_item`이 하나씩 들어 있다. 그 안의 `li_trans`로 item을 지금 붙은 transaction의 `t_items` 목록에 건다. `li_type`에는 on-disk log 형식의 종류 상수(`XFS_LI_INODE`, `XFS_LI_BUF`, intent와 done item 쌍 등)가 들어간다. 예외로 `XFS_LI_IUNLINK`는 log에 쓰이지 않는다. CIL과 AIL 쪽 field는 [log](12-log.md)에 있다.

commit과 cancel의 공통 code는 item 종류로 갈라지지 않고, 함수 표(`li_ops`)의 함수와 flag를 쓴다.

| 함수 | 언제, 무엇을 |
|---|---|
| `iop_precommit` | commit 직전, 아직 lock을 쥔 채 부른다. |
| `iop_size`, `iop_format` | CIL에 넣을 때 크기를 재고 log vector로 복사한다. |
| `iop_pin`, `iop_unpin` | log에 쓰일 때까지 객체를 memory에 붙잡는다. |
| `iop_committing` | CIL에 복사한 뒤 item을 transaction에서 떼며 부른다. buffer와 inode는 여기서 lock이 풀린다. 단, hold했거나 stale인 buffer와, join할 때 lock flag를 0으로 준 inode는 풀리지 않는다. |
| `iop_release` | commit할 것이 없거나 cancel일 때 부른다. |

flag `XFS_ITEM_RELEASE_WHEN_COMMITTED`가 붙은 item은 checkpoint가 journal에 commit되면 AIL에 넣지 않고 바로 놓는다. icreate item이 그렇다. icreate item에는 pin과 push도 없어 log에만 남는다. iunlink item에는 format이 없다. iunlink item은 precommit에서 inode buffer에 next 포인터를 기록하고 목록에서 빠진다. log에는 buffer item이 남는다.

### 붙이기, 표시하기, commit

item은 transaction에 붙은(join) 뒤 따로 dirty로 표시되고(log), commit 때 precommit을 거쳐 CIL로 간다.

| 단계 | 하는 일 | item |
|---|---|---|
| join | item을 `t_items`에 넣는다(`xfs_trans_add_item`). inode와 buffer는 item이 없으면 이때 만든다. inode는 `ILOCK_EXCL`을 쥔 것만 붙고 commit 때 풀 lock flag를 적는다. buffer는 참조를 하나 올린다. | dirty가 아니어야 한다 |
| log | inode는 바뀐 field를 bit로 모으고(`xfs_trans_log_inode`), buffer는 바뀐 byte 범위를 bitmap에 적는다(`xfs_trans_log_buf`). | item과 transaction이 dirty가 된다 |
| precommit | dirty item의 `iop_precommit`을 lock 순서로 정렬해 부른다. 이미 dirty라 되돌릴 수 없으므로 실패하면 shutdown한다. inode가 cluster buffer와 아직 이어져 있지 않으면 여기서 inode item을 그 buffer의 목록에 건다. |  |
| commit 뒤 | CIL이 dirty item만 골라 형식을 만들고, 모든 item을 떼며 `iop_committing`을 부른다. | 떨어지며 `XFS_LI_DIRTY`가 지워진다 |

- buffer는 바뀐 byte 범위(물리 log)를, inode는 바뀐 field(논리 log)를 남긴다. ordered buffer는 log에 내용을 쓰지 않는다. 그래도 dirty로 표시되어, log한 buffer처럼 AIL에서 writeback 순서를 지킨다. buffer 쪽 log item은 [buffer cache와 verifier](14-buffer-verifier.md)에 있다.
- superblock counter는 item이 아니라 transaction의 delta field에 쌓고(`xfs_trans_mod_sb`), commit 때 필요하면 superblock buffer를 join해 log한다. lazysbcount면 free block과 inode 수는 log하지 않고 in-core counter만 고친다.
- 같은 inode나 buffer가 다음 transaction에 다시 붙으면 거기서 다시 log된다. 이때 이미 CIL이나 log에 있는 이전 변경과 합쳐 새 사본으로 쓴다. 이것을 re-logging이라 한다. rolling transaction은 re-logging에 기댄다.
- 지금 kernel의 CIL은 checkpoint 하나에 transaction header 하나를 쓴다. header의 tid에는 CIL 자신의 ticket tid가 들어간다. 그래서 개별 transaction의 경계는 on-disk log에 남지 않는다(추정). [log](12-log.md)의 관찰에서도 file 100개를 만진 변경이 `xfs_logprint`에서 TRANS 하나로 나왔다.

code: xfs_trans.h, xfs_trans.c, xfs_trans_inode.c, xfs_trans_buf.c, xfs_inode_item.c, xfs_buf_item.c, xfs_icreate_item.c, xfs_iunlink_item.c, xfs_log_cil.c

## commit과 cancel

이 장의 동작은 code로 확인했다.

### commit

`xfs_trans_commit`은 변경을 CIL에 넘기고, sync transaction이 아니면 log 쓰기를 기다리지 않고 돌아온다.

1. permanent면 남은 deferred op를 먼저 끝낸다(`xfs_defer_finish_noroll`). 이 안에서 여러 번 roll하고, 실패하면 cancel한다.
2. superblock delta와 dquot delta를 item으로 만들고 precommit을 부른다.
3. dirty가 아니면 CIL로 가지 않고 예약만 반납한다. log가 shutdown이면 EIO를 돌려준다.
4. dirty면 `xlog_cil_commit`이 I/O 없이 item을 형식으로 만들어 CIL에 넣는다. 그다음 ticket을 regrant 또는 ungrant하고, block 예약을 반납한다. 이어서 `iop_committing`을 부른 뒤 checkpoint sequence를 돌려준다.
5. `XFS_TRANS_SYNC`가 서 있으면 그 sequence까지 log force를 기다린다.

문서에 따르면 `xfs_trans_commit()`이 돌아온 시점에 변경이 stable storage에 commit됐다는 보장은 없다. log force는 [log](12-log.md)에 있다. commit이 실패해도 item은 이미 풀렸으므로 호출자는 transaction을 다시 만지지 않는다. 통계에서는 commit을 sync, async, empty(dirty가 아님)로 나눠 센다. 관찰한 commit은 모두 async였다.

### cancel

XFS에는 undo가 없다. `xfs_trans_cancel`은 바뀐 것을 되돌리지 않고 다음을 한다.

1. deferred op가 붙어 있으면 dirty로 보고 버린다.
2. dirty면 shutdown한다(`SHUTDOWN_CORRUPT_INCORE`). 주석에 따르면 dirty transaction은 이전 상태로 되돌릴 방법이 없기 때문이다. 이미 shutdown 중이면 report를 다시 하지 않는다.
3. block과 quota 예약을 반납한다.
4. log 예약을 전부 돌려준다(`xfs_log_ticket_ungrant`). `t_cnt`를 (0보다 크면) 하나 줄인 뒤, 남은 `t_curr_res`에 아직 쓰지 않은 선불 조각 `t_unit_res x t_cnt`를 더해 두 head에서 뺀다.
5. item을 떼고 놓는다. dirty였으면 `XFS_LI_ABORTED`를 세운다.

그래서 cancel은 아직 아무것도 log하지 않았을 때만 안전하다. quota 예약이 EDQUOT나 ENOSPC로 실패해 retry할 때(`xfs_trans_alloc_inode`)는 수정 전에 cancel한다. 예약 단계에서 실패해도 할당 함수 안에서 cancel한다. 빈 transaction([code를 읽기 위한 최소 모형](03-code-model.md))은 예약이 없어 cancel해도 반납할 것이 없지만, dirty로 만들면 안 된다.

code: xfs_trans.c, xfs_defer.c, xfs_log_cil.c, xfs_log.c

## block 예약

이 장의 동작은 code로 확인했다.

block도 log처럼 최악의 경우를 미리 잡는다. data device에서 쓸 block 수는 `xfs_trans_alloc`의 `blocks` 인자로 예약한다. 예약은 block, log, realtime extent 순으로 잡고, 뒤가 실패하면 앞을 되돌린다. 몇 block을 잡을지는 호출자가 정한다. direct write 할당은 data 길이에 bmbt split 몫을 더하고, create는 inode chunk와 inobt, finobt, directory entry 몫을 더한다. truncate는 보통 block을 풀기만 하므로 0을 잡는다.

- 예약할 때는 in-core free counter를 줄여 `t_blk_res`에 적고, on-disk 할당은 아직 하지 않는다. counter가 쓸 수 없는 양 밑으로 내려가면, `XFS_TRANS_RESERVE` transaction은 reserve pool에서 꺼내고, 다른 transaction은 ENOSPC로 실패한다. ENOSPC면 blockgc로 speculative preallocation 등을 비운 뒤 한 번 더 시도한다. counter의 구성과 reserve pool은 [free space 구조와 할당 정책](09-free-space.md)에, blockgc는 [fallocate와 mmap](21-fallocate-mmap.md)에 있다.
- 할당기는 실제로 잡은 block을 `xfs_trans_mod_sb`로 transaction에 알린다. `xfs_trans_mod_sb`는 쓴 양이 예약을 넘으면 shutdown한다.
- `XFS_TRANS_RES_FDBLKS`가 서 있으면 roll 사이에 푼 block은 전역 counter가 아니라 이 transaction의 예약으로 돌아간다. 주석에 따르면 block을 되풀이해 풀고 할당하는 roll chain 동안 예약을 유지하려는 것이다.

commit과 cancel은 남은 예약을 똑같이 반납한다. 반납할 block은 `t_blk_res + t_fdblocks_delta`다. 할당은 delta가 음수라 쓴 만큼 빠지고, 해제는 양수라 더해진다. 돌려줄 때 reserve pool이 비어 있으면 그쪽을 먼저 채운다. 10 block을 예약하고 3 block을 할당하면 다음과 같다(code로만 확인한 예).

| handle | `t_blk_res` | `t_fdblocks_delta` | 반납 |
|---|---|---|---|
| roll 없이 commit | 10 | -3 | 7 |
| roll한 옛 handle | 3 | -3 | 0. 남은 7은 `xfs_trans_dup`이 새 handle로 옮긴다 |

lazysbcount(v5 기본)면 이 counter 변경은 on-disk superblock에 log되지 않는다.

code: xfs_trans.c, xfs_mount.c, xfs_ag_resv.c, xfs_trans_space.h, xfs_trans_space.c

## log 크기의 하한

이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

mount는 가장 큰 transaction 둘이 log에 함께 들어가는지 검사하고, v5에서 log가 하한보다 작으면 mount를 거부한다. 하한을 구할 때는 `logres x logcount`가 가장 큰 항목을 골라 한 조각 크기로 바꿔 logcount를 곱하고, 512 KiB(log stripe unit이 없을 때)를 더한 뒤 3(`XFS_MIN_LOG_FACTOR`)을 곱한다. 관찰한 file system에서는 `tr_rename`(428160 byte, logcount 5)이 골라졌다. 덧붙임을 빼고 어림하면 (428160 x 5 + 524288) x 3 = 7995264 byte(약 7.6 MiB)이고(code로 낸 어림, 하한 값은 관찰하지 않음), 이 file system의 log(16384 block, 64 MiB)는 그보다 훨씬 크다.

parent pointer가 있는 file system에서는 하한 계산용 표도 runtime과 같은 `xfs_trans_resv_calc`로 만든다. 없으면 옛 reflink 시절의 값을 쓴다. 옛 값에서는 runtime이 별도 transaction으로 하는 refcount 갱신이 logres에 다시 들어가고, reflink가 있으면 `tr_write`와 `tr_itruncate`의 logcount가 8이다. 주석에 따르면 하한을 줄이면 새 mkfs가 만든 작은 log를 옛 kernel이 mount하지 못하므로, 2023년에 없던 feature인 parent pointer가 있을 때만 runtime 표를 쓴다. 관찰한 file system에는 parent pointer가 있어서 하한 계산용 표가 runtime 표와 같았다.

code: xfs_log_rlimit.c, xfs_log.c, xfs_trans_resv.c, xfs_trans_resv.h

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code |
|---|---|---|
| Documentation/filesystems/xfs/xfs-delayed-logging-design.rst | 선불이 바닥난 permanent transaction은 write head 공간을 "commit 뒤의 `xfs_log_reserve()` 호출"로 받는다 | 지금 code는 `xfs_log_regrant`를 부른다. |
| xfs-documentation journaling_log.asciidoc | transaction header의 type 표(`XFS_TRANS_SETATTR_NOT_SIZE` 등) | delayed logging 전의 형식이다. 지금 CIL은 checkpoint 하나에 header 하나를 쓰고 type은 늘 `XFS_TRANS_CHECKPOINT`(40)다. |

## 한계점

다음은 확인하지 못했다.

- mkfs.xfs의 log 최소 크기 규칙과 mount가 검사하는 하한의 관계.
- roll마다 오르는 `t_tid`가 on-disk log에 남지 않는다는 것. code로만 읽었고 log를 직접 읽어 확인하지 않았다.
- lazysbcount일 때 on-disk superblock의 counter가 언제 맞춰지는지.
- log 공간이 모자라 ticket이 FIFO 대기열에서 잠드는 경로. code만 읽었고, 관찰한 두 workload에서는 잠든 event가 없었다.
- 여러 task가 동시에 같은 log 공간을 두고 경쟁할 때의 모습. 한 번에 한 workload씩만 보았다.
- extent 1280개인 file의 truncate에서 roll이 3215번 난 내역. unmap 한 번에 deferred 작업이 몇 개 붙는지는 나눠 보지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 기본 image를 option 없는 `mkfs.xfs`로 새로 만들고 option 없이 mount했다. 이 image에는 rmapbt, reflink, parent pointer, lazy-count가 켜져 있다.
- 두 관찰 모두 log가 넉넉한 file system에서 한 번에 한 workload만 돌렸다. roll 관찰에서 ring buffer 넘침은 CPU마다 0이었다.

### 관찰 목록

명령의 `$i`는 1 GiB image file, `$L`은 그 loop device, `$m`은 mount 지점이다.

| 관찰 | 절차와 명령 | 켠 event | 출처 | 본문의 예 |
|---|---|---|---|---|
| 예약 크기 | ```bash<br>xp71 mkfs.xfs -f $i<br>xp71 xfs_db -f -r -c logres $i          # mount 전<br>mount $L $m                             # event를 켠 채<br># kernel event의 type 줄과 xfs_db logres의 type 줄을 diff<br>``` | `xfs_trans_resv_calc`, `xfs_trans_resv_calc_minlogsize`, `xfs_log_get_max_trans_res` | man xfs_db(8)의 logres(xfsprogs와 kernel의 reservation 계산 차이를 찾는 용도), fstests common/tracing | 종류별 예약 표, logcount 표, log 크기의 하한 |
| roll | ```bash<br>for f in b c; do<br>xp71 xfs_io -f -c "falloc 0 10m" $m/$f<br>/opt/fstests-d98a5b5/src/punch-alternating $m/$f<br>done<br># 여기서 event를 켠다<br>xp71 xfs_io -c "truncate 0" $m/b; sync<br>rm -f $m/c; sync; sleep 2<br># stats는 truncate 전, truncate 뒤, rm 뒤에 읽음<br>grep -E "^(trans\|push_ail) " /sys/fs/xfs/<dev>/stats/stats<br>``` | `xfs_trans_alloc`, `xfs_trans_roll`, `xfs_trans_commit`, `xfs_trans_cancel`, `xfs_defer_trans_roll`, `xfs_log_reserve`, `xfs_log_reserve_exit`, `xfs_log_regrant`, `xfs_log_regrant_exit`, `xfs_log_ticket_regrant`, `xfs_log_ticket_regrant_sub`, `xfs_log_ticket_regrant_exit`, `xfs_log_ticket_ungrant`, `xfs_log_grant_sleep` | fstests tests/xfs/423(falloc 뒤 punch-alternating), tests/xfs/057(file 제거 workload. DEBUG 전용 errortag는 뺐다), common/tracing | truncate 한 번의 roll |

### 출력 읽는 법

- `xfs_db logres`와 `xfs_trans_resv_calc` event의 줄은 type, logres(byte), logcount, flags 순이고, flags의 `0x4`는 permanent를 뜻한다. `xfs_db logres`는 마지막에 `minlogsize` 줄을 더 찍는다.
- log 예약 쪽 event의 `rsv`, `wr`는 tail이 잡은 공간에 각 head의 grant를 더한 위치라서 sysfs의 grant 값과 다르다. 본문의 산수는 차이만 쓴다.
- `xfs_log_ticket_regrant`는 `t_cnt`를 줄이기 전에 찍힌다. 그 조각이 쓴 byte는 231944에서 이 event의 `curr`를 뺀 값이다. `xfs_log_regrant`는 선불이 남은 동안 찍히지 않는다.
- `xfs_trans_commit` event는 roll 안의 commit과 chain 끝 commit에서 모두 나서, chain 하나에서 roll 수보다 한 번 더 난다.
- stats의 `trans` 줄은 sync, async, empty 순이다. `push_ail` 줄의 앞 두 값은 예약 시도 수(`xs_try_logspace`)와 잠든 횟수(`xs_sleep_logspace`)다.

### 원본 출력: 종류별 예약 표

(원본 출력 "xfs_db logres와 kernel의 reservation 계산값 비교": `raw/11-transaction.md`)

(원본 출력 "하한 계산용 표의 앞부분과 고른 항목": `raw/11-transaction.md`)

### 원본 출력: truncate 한 번의 roll

(원본 출력 "xfs_io truncate의 처음 roll들: xfs_log_reserve에서 steady state까지": `raw/11-transaction.md`)

(원본 출력 "xfs_io truncate의 t_cnt 분포와 ticket 크기": `raw/11-transaction.md`)

(원본 출력 "task별 event 수": `raw/11-transaction.md`)

(원본 출력 "truncate와 rm 전후의 stats": `raw/11-transaction.md`)
