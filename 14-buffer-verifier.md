# buffer cache와 verifier

> XFS 전반 wiki, 묶음 "정합성". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/14-buffer-verifier.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/14-buffer-verifier.md, 접힌 원본 출력은 raw/14-buffer-verifier.md.

## 개요

XFS에서 metadata를 바꾸는 code는 disk block을 직접 만지지 않고 memory의 buffer를 거친다.

- buffer는 cache에서 lock된 채로 나오고, transaction에 들어가면 buffer당 하나뿐인 buf log item이 붙는다. commit이 끝나면 buffer는 unlock되지만, disk에는 checkpoint가 log에 쓰인 뒤 xfsaild가 AIL을 push할 때 쓰인다.
- v5에서 AGF와 AGI의 verifier는 읽을 때 CRC를 먼저 보고, 맞을 때만 구조를 본다. 쓸 때는 구조를 검사한 뒤 LSN과 CRC를 쓰기 직전에 채운다.
- inode는 buffer의 byte 범위가 아니라 in-core inode에서 만든 내용을 log에 적고, disk에는 inode cluster buffer 단위로 함께 쓴다.

예의 값은 option 없이 만든 1 GiB file system에서 얻었다. 관찰마다 같은 조건의 새 file system을 만들었다.

## buffer cache와 transaction

metadata buffer는 daddr로 cache에서 찾고, transaction은 그 buffer에 buf log item을 붙여 바꾼 범위를 적는다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### cache 단위

buffer cache는 buftarg(buffer를 읽고 쓰는 대상으로, 보통 device 하나다)마다 hash 하나와 LRU 하나로 이루어지고, hash의 key는 buffer의 시작 daddr다. 같은 daddr라도 길이가 다르면 같은 buffer로 보지 않으므로 cache 단위는 daddr와 길이의 쌍이 된다. ASSERT에 따르면 같은 daddr에 길이가 다른 buffer가 함께 있을 수 있는 것은 stale buffer(무효로 표시해 disk에 쓰지 않고 곧 버릴 buffer)뿐이다.

새 file system에서 file을 만들 때 다룬 buffer 몇 개의 위치와 크기는 다음과 같다. AG header의 배치는 [on-disk 배치](04-on-disk-layout.md)에 있다.

| buffer | ops | 위치(AG 0) | 크기 |
|---|---|---|---|
| AGI | `xfs_agi_buf_ops` | block 0 안의 sector 2(0부터 셈) | 512 byte |
| inobt block | `xfs_inobt_buf_ops` | block 3 | 4 KiB |
| finobt block | `xfs_finobt_buf_ops` | block 4 | 4 KiB |
| inode cluster | `xfs_inode_buf_ops` | block 16부터 4 block | 16 KiB(512 byte inode 32개) |

buffer를 찾는 `xfs_buf_get_map`은 hash에서 buffer를 찾은 뒤 lock까지 잡아서 돌려준다. 그래서 cache에서 꺼낸 buffer는 늘 lock된 상태다. transaction은 바꾼 buffer의 lock을 commit까지 쥐고 있고, 그동안 같은 buffer를 원하는 다른 transaction은 기다린다. `XBF_TRYLOCK`으로 부른 쪽은 기다리지 않고 `-EAGAIN`을 받는다.

file 200개를 만드는 동안 buffer 조회 1228번 가운데 1212번(약 98.7%)이 hit였고, disk에서는 8번 읽었다. transaction 200개가 AGI를 한 번씩 붙였지만 AGI를 disk에서 읽은 적은 없었다. inobt와 finobt block은 203번씩 붙이는 동안 disk에서 한 번씩만 읽었다.

code: xfs_buf.c, xfs_buf.h

### LRU와 b_lru_ref

참조가 모두 풀린 buffer는 바로 해제되지 않고 LRU에 남아 cache가 된다. memory가 모자라면 shrinker가 LRU를 훑으며 쓰이지 않는 buffer의 `b_lru_ref`를 1씩 줄이고, 이미 0인 buffer만 회수한다. 그래서 `b_lru_ref`는 buffer가 shrinker를 몇 번 견디는지를 나타낸다. 마지막 참조가 풀릴 때 `b_lru_ref`가 0인 buffer와 uncached buffer는 LRU에 남지 않고 바로 해제된다.

새 buffer는 1로 시작하고, AGF를 읽는 `xfs_read_agf`처럼 구조를 읽는 함수가 `xfs_buf_set_ref`로 아래 값을 넣는다.

| 값 | 구조 |
|---|---|
| 4 | AGF, AGI |
| 3 | AGFL, inobt |
| 2 | free space btree, bmap btree, rmap btree, directory btree, inode |
| 1 | xattr btree, dquot, refcount btree |

예를 들어 쓰이지 않는 AGF와 AGI(4)는 refcount btree block(1)보다 shrinker를 세 번 더 견딘다.

code: xfs_buf.c, xfs_shared.h, xfs_alloc.c, xfs_btree.c

### transaction에 붙이기: buf log item

transaction이 buffer를 붙이면 그 buffer에 buf log item(bli)이 붙는다. bli는 buffer마다 하나뿐이라 없을 때만 만들고, transaction마다 새로 생기는 것은 bli 참조와 transaction 목록 연결이다. buffer를 log했으면 bli는 transaction이 끝나도 남고, log하지 않은 깨끗한 bli는 commit 때 해제된다. transaction 자체는 [transaction](11-transaction.md)에 있다.

- [code를 읽기 위한 최소 모형](03-code-model.md)에 나온 `xfs_trans_read_buf`는 먼저 이 transaction이 같은 target, daddr, 길이의 buffer를 이미 붙였는지 본다. 붙였다면 lock도 이미 쥐고 있으므로 재귀 횟수(`bli_recur`)만 올려 돌려준다. 아니면 cache에서 읽고 bli를 붙인다. 읽지 않고 할당만 하는 `xfs_trans_get_buf`도 같은 순서다.
- transaction이 이미 dirty인데 읽기가 실패하면 file system을 shutdown한다(`SHUTDOWN_META_IO_ERROR`).
- 바꾼 범위는 `xfs_trans_log_buf`로 적는다. bli와 transaction을 dirty로 표시하고, 바뀐 byte 범위를 bli의 dirty bitmap에 128 byte 단위로 표시한다. AGF라면 바뀐 field의 첫 byte와 끝 byte를 구해 그 범위만 log한다.

code: xfs_trans_buf.c, xfs_buf_item.c, xfs_alloc.c

### commit에서 disk까지

commit은 buffer의 바뀐 범위를 CIL에 넣고 unlock할 뿐, disk에는 쓰지 않는다. buffer는 checkpoint가 log에 쓰인 뒤 AIL에 들어가고, xfsaild가 AIL을 push할 때 제자리에 쓰인다. CIL, checkpoint, AIL은 [log](12-log.md)에 있다.

file 하나를 만드는 transaction이 AGI를 바꾸면, AGI buffer는 disk에 쓰이기까지 이런 상태를 거친다.

| 단계 | 함수 | buffer lock | `b_pin_count` | bli | AIL |
|---|---|---|---|---|---|
| 1. transaction이 읽음 | `xfs_trans_read_buf` | 잡음 | 0 | 붙음 | 밖 |
| 2. 바꾼 범위를 적음 | `xfs_trans_log_buf` | 잡음 | 0 | dirty | 밖 |
| 3. commit: CIL에 들어감 | `xfs_buf_item_format`, `xfs_buf_item_pin` | 잡음 | 1 | pin이 참조를 쥠 | 밖 |
| 4. commit 끝 | `xfs_buf_item_release` | 풂 | 1 | 남음 | 밖 |
| 5. checkpoint가 log에 쓰임 | `xfs_buf_item_unpin` | 풂 | 0 | 남음 | 들어감 |
| 6. xfsaild가 push | `xfs_buf_item_push` | 잡았다가 delwri 목록에 넣고 풂 | 0 | 남음 | 안 |
| 7. 목록을 냄 | `xfs_buf_delwri_submit_nowait` | 다시 잡음(못 잡으면 다음 차례) | 0 | 남음 | 안 |
| 8. write 완료 | `xfs_buf_item_done` | 풂 | 0 | 해제 | 빠짐 |

trace에서도 AGI의 `xfs_trans_log_buf`(bli가 `DIRTY|LOGGED`)와 `xfs_buf_item_pin`이 찍혔고, AGI의 write는 freeze 때 한 번 났다.

- CIL에 들어갈 때는 dirty bit가 이어진 구간마다 그 시점 buffer memory를 128 byte chunk로 복사한다. 그래서 field 하나만 바뀌어도 그 field가 든 128 byte가 통째로 log에 들어간다.
- pin은 같은 CIL context에 처음 들어갈 때 한 번만 한다. file 200개를 만드는 동안 `xfs_trans_log_buf`는 2070번, buffer pin은 15번이었다. 같은 buffer를 여러 transaction이 거듭 log했기 때문일 것이다(buffer별로 세지 않아 추정).
- pin된 buffer는 쓰지 않는다. write를 내기 전에 pin이 풀리길 기다리고, 목록을 낼 때는 lock을 못 잡았거나 pin된 buffer를 건너뛰어 남긴다.

file 200개를 만들고 sync한 뒤 freeze하자 buffer write는 freeze 때만 났다.

| 구간 | `xfs_trans_log_buf` | buffer pin | buffer item push | buffer write |
|---|---|---|---|---|
| file 200개 touch | 2070 | 15 | 0 | 0 |
| sync | 0 | 0 | 0 | 0 |
| freeze와 unfreeze | 2 | 2 | 17 | 19 |

sync는 log만 force하므로 buffer는 제자리에 쓰이지 않았다. freeze는 log force를 한 뒤 AIL을 전부 push하므로 그때 19개가 한꺼번에 쓰였다. 19개 가운데 8개는 inode cluster buffer다. buffer item push 열에는 buffer item의 push만 들어 있고, inode item의 push는 세지 않았다.

code: xfs_buf_item.c, xfs_buf.c, xfs_log_cil.c, xfs_trans_ail.c, xfs_log.c, xfs_super.c

## 읽을 때와 쓸 때의 verifier

verifier는 buffer 내용이 그 종류의 구조에 맞는지 검사하는 함수 묶음이고, 읽기가 끝난 직후와 write를 내기 직전에 실행된다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### ops

buffer 종류마다 `struct xfs_buf_ops`가 하나 있고, 그 안에 이름, v4와 v5 magic, `verify_read`, `verify_write`, `verify_struct`가 들어 있다. `verify_struct`는 I/O 없이 memory 내용만 검사해 실패 주소를 돌려준다. 어떤 구조로 읽을지는 caller가 read 때 ops를 넘겨 정하고, verifier는 그 구조가 맞는지만 본다. magic은 `xfs_has_crc`로 v4와 v5 가운데 하나를 골라 비교한다.

ops는 disk에서 처음 읽을 때 buffer에 둔다. 이미 읽은 buffer(`XBF_DONE`)는 I/O 없이 돌려주고, ops가 없을 때만 다시 검사한다. 주석에 따르면 ops 없는 buffer는 V4 log recovery와 online repair에서 생긴다.

앞의 file 200개를 만들 때 disk에서 읽은 buffer 8개와 freeze 때 쓴 buffer 19개에는 모두 ops가 붙어 있었다. 쓴 buffer는 superblock, AGF, AGI, free space btree, inobt와 finobt, rmapbt, directory block, inode cluster였다.

### 읽을 때: AGF

AGF와 AGI의 read verifier는 v5에서 CRC를 먼저 보고, CRC가 맞을 때만 구조를 검사한다. 그래서 CRC를 다시 계산하지 않고 field를 바꾸면, 어느 field를 바꿔도 CRC error가 먼저 난다.

- read verifier는 read I/O가 error 없이 끝나고 ops가 있을 때 `__xfs_buf_ioend`가 부른다. 검사를 통과했을 때만 buffer에 `XBF_DONE`을 켠다.
- CRC는 crc field를 0으로 보고 buffer 전체에 crc32c를 계산한 값과 비교한다.
- 구조 검사 `xfs_agf_verify`는 v5에서 uuid와 LSN(지금 log head보다 앞서지 않는지)을 본다. 그 밖에 magic, version, AG 번호와 길이, AGFL index 범위, `freeblks`가 `longest` 이상이고 AG 길이 이하인지, btree level과 block 수를 본다. 실패하면 실패한 자리의 주소를 돌려준다.
- read verifier는 CRC가 틀리면 `-EFSBADCRC`를, 구조 검사가 실패하면 `-EFSCORRUPTED`를 `xfs_verifier_error`로 buffer의 error(`b_error`)에 남긴다.

verifier는 다른 block과의 관계를 보지 않는다. 예를 들어 AGF의 `freeblks`가 free space btree의 합계와 같은지는 보지 않는다. kernel 문서에도 같은 한계가 적혀 있다. 그런 검사는 [xfs_repair와 online fsck](32-fsck.md)에 있다.

### 실패의 보고

verifier가 실패하면 kernel은 어느 검사가 실패했는지 message로 남기지만, caller에게는 CRC error와 구조 error를 구분하지 않고 corruption으로 올린다. AGF와 AGI를 바꾸고 mount하면 이렇게 된다.

| 바꾼 것 | verifier message | `metadata I/O error` 줄 | caller가 받은 error |
|---|---|---|---|
| AGF `freeblks`를 0으로, CRC는 다시 계산 | `Metadata corruption detected at xfs_agf_verify+0x208/0x280` | error 117(EUCLEAN) | -117 |
| AGF `freeblks`를 0으로, CRC는 그대로 | `Metadata CRC error detected at xfs_agf_read_verify+0x80/0x110` | error 74(EBADMSG) | -117 |
| AGI의 388 bit를 무작위로, CRC는 그대로 | `Metadata CRC error detected at xfs_agi_read_verify+0x80/0x110` | error 74 | -117 |

AGF는 per-AG metadata reserve pool을 잡는 단계, AGI는 root inode를 찾는 단계가 받은 값이다.

- message는 `xfs_verifier_error`가 낸다. 함수 이름과 offset 자리에 verifier 안에서 실패한 주소가 들어가므로 어느 검사가 실패했는지 가릴 수 있다. error_level(기본 3)이 1 이상이면 buffer 앞 128 byte의 hex dump를, 5 이상이면 stack도 낸다.
- `-EFSBADCRC`는 EBADMSG와, `-EFSCORRUPTED`는 EUCLEAN과 같은 값이다. `xfs_buf_read_map`은 error 경로에서 `metadata I/O error in ...` 줄을 낸 뒤 `-EFSBADCRC`를 `-EFSCORRUPTED`로 바꿔 올린다. 그래서 표의 셋째 열은 74와 117로 갈리지만 넷째 열은 모두 -117이다.
- 같은 경로에서 buffer를 stale로 만들어 놓으므로 다음 read는 disk에서 다시 읽는다.
- 세 경우 모두 mount는 rc 32로 실패했다. AGF는 message 순서로 보아 그 단계에서 읽혔고, 그 단계의 `xfs_fs_reserve_ag_blocks`가 `-ENOSPC`가 아닌 error를 받아 file system을 shutdown했다. AGI 쪽은 그 뒤 root inode를 읽지 못했다는 `Failed to read root inode 0x80, error 117`이 나왔다.

code: xfs_alloc.c, xfs_ialloc.c, xfs_buf.c, xfs_error.c, xfs_fsops.c

### inode buffer의 검사

inode cluster buffer의 verifier는 inode마다 magic, version, `di_next_unlinked` 범위만 보고, CRC를 비롯한 나머지 검사는 iget이 inode 하나를 disk에서 읽을 때 한다([inode](06-inode.md)에 있다). v3 inode를 새로 만드는 iget은 disk에서 읽지 않으므로 이 검사를 거치지 않는다.

### 쓸 때

write verifier는 memory에서 잘못 바뀐 metadata를 disk에 쓰기 전에 잡아 file system을 shutdown한다. `xfs_buf_submit`은 pin이 풀리길 기다린 뒤 write verifier를 부른다. verifier가 실패하면 `SHUTDOWN_CORRUPT_INCORE`로 shutdown하고 I/O 없이 끝낸다. shutdown 뒤의 처리는 [shutdown, error 처리, health](31-shutdown-health.md)에 있다.

- AGF의 write verifier는 read 때와 같은 구조 검사를 한다. v5면 CRC를 다시 계산해 넣는다. 그 전에 bli가 있으면 `agf_lsn`에 bli의 `li_lsn`(마지막으로 log에 들어간 LSN)을 넣는다. AGI도 같다.
- 그래서 disk의 CRC와 LSN은 write 직전에 정해진다. log에는 그 시점의 buffer memory가 복사되므로 CRC field가 맞지 않을 수 있다. kernel 문서의 구조 예에도 crc와 lsn에는 "not logged"라는 주석이 붙어 있다.
- cache에 든 v5 buffer에 ops가 없으면 경고와 hex dump, stack을 내고 그대로 쓴다. uncached buffer는 경고 없이 쓴다.

code: xfs_buf.c, xfs_alloc.c, xfs_ialloc.c

## inode의 log와 disk 쓰기

inode 변경은 buffer와 다른 길로 log되고 disk에 쓰인다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

XFS는 inode 여러 개를 buffer 하나에 담아 함께 읽고 쓰고, 이 묶음을 inode cluster라고 부른다. 이 file system에서는 cluster buffer 하나가 16 KiB이고 512 byte inode 32개가 들어간다.

### log: buffer가 아니라 in-core inode를 적는다

- `xfs_trans_log_inode`는 transaction과 inode item을 dirty로 만들고 바뀐 종류를 모을 뿐, cluster buffer는 건드리지 않는다. cluster buffer는 commit의 precommit 단계에서 잡는다. 주석에 따르면 AGI, AGF, inode cluster buffer 순서로 lock하려면 cluster buffer를 transaction의 맨 끝에 잡아야 하기 때문이다. precommit은 item을 정렬한 뒤 dirty item마다 부르고, inode item은 inode 번호 순으로 정렬된다.
- `xfs_inode_item_precommit`은 inode item에 cluster buffer가 아직 붙지 않았을 때만 그 buffer를 `xfs_inode_buf_ops`로 읽어, inode item을 buffer의 목록(`b_li_list`)에 붙이고 write 완료 hook(`b_iodone`)을 단다. 그 뒤 buffer를 transaction에서 놓는다. 그래서 inode 변경만 있는 transaction은 cluster buffer를 dirty로 만들지 않고, buffer는 inode item이 깨끗해질 때까지 참조로 cache에 붙잡힌다.
- log에는 cluster buffer 안의 위치와, in-core inode에서 만든 core 전체와 fork를 적는다. core는 늘 들어간다. 앞 flush가 아직 disk에 닿지 않았으면 그때의 field도 다시 log한다.
- pin은 inode의 `i_pincount`만 올리고 buffer의 pin count는 올리지 않는다. file 200개를 만드는 동안 `xfs_inode_pin`은 201번 찍혔다.

code: xfs_trans_inode.c, xfs_inode_item.c, xfs_trans.c

### disk에 쓰기: AIL push에서 cluster flush로

xfsaild가 inode item 하나를 push하면 그 cluster buffer에 붙은 dirty inode가 모두 함께 쓰인다. `xfs_inode_item_push`는 inode나 buffer가 pin돼 있거나, 이미 flush 중이거나, buffer lock을 못 잡으면 flush하지 않고 돌아간다. 잡으면 `xfs_iflush_cluster`를 부르고, 성공하면 buffer를 delwri 목록에 넣는다.

- `xfs_iflush_cluster`는 buffer의 목록을 돌며 reclaim 중이거나 flush 중이거나 pin된 inode와, ILOCK_SHARED를 바로 잡지 못한 inode를 건너뛴다. dirty inode만 `xfs_iflush`로 쓰지만, lock을 잡은 inode는 깨끗해도 센다.
- `xfs_iflush`는 in-core inode를 buffer 안 dinode에 복사하고, 이번에 쓴 field와 지금 LSN을 기억해 두고, inode CRC를 여기서 계산한다. 그 뒤 buffer를 쓸 때는 inode buffer verifier가 write verifier로 실행되므로 magic, version, `di_next_unlinked`만 본다.
- write가 끝나면 `xfs_buf_inode_iodone`이 flush 뒤 다시 log되지 않은 inode를 AIL에서 빼고, 깨끗해진 inode의 buffer 연결과 참조를 끊는다. 다시 dirty해진 inode는 buffer에 남는다.

앞의 file 200개 관찰에서 inode 쪽 값은 다음과 같았다.

| 시점 | inode buffer write | cluster flush | flush가 센 inode |
|---|---|---|---|
| file 200개를 만들고 sync한 뒤 | 0(read 2번뿐) | 0 | 0 |
| freeze와 unfreeze 뒤 | 8 | 7 | 201 |

센 inode 201개가 서로 다른 inode라면 cluster 하나에 32개씩이므로 적어도 cluster 7개에 걸친다. flush도 7번이었다. 센 inode 수는 file을 만드는 동안의 `xfs_inode_pin` 수와 같지만, 깨끗한 inode도 세므로 실제로 쓴 inode 수라고 볼 수는 없다.

두 방식의 차이를 모으면 다음과 같다.

|  | buffer item(AGF 등) | inode item |
|---|---|---|
| log에 적는 것 | dirty 128 byte chunk의 buffer 내용 | in-core inode에서 만든 core 전체와 fork |
| transaction 중 lock | buffer lock(commit까지) | ILOCK. cluster buffer는 precommit 때 잠깐 |
| pin | `b_pin_count` | `i_pincount` |
| disk 쓰기 단위 | 그 buffer | cluster buffer(붙은 dirty inode 전부) |
| CRC 계산 | write verifier | `xfs_iflush` |

code: xfs_inode_item.c, xfs_inode.c, xfs_inode_buf.c

## 문서와 code가 다른 곳

| 문서 | 문서의 설명 | code와 관찰 |
|---|---|---|
| Documentation/filesystems/xfs/xfs-self-describing-metadata.rst | read verifier는 magic을 먼저 보고 CRC가 필요한지 정한 뒤 CRC를 본다 | AGF와 AGI의 read verifier는 `xfs_has_crc`로 정하고 CRC부터 본다. CRC를 다시 계산하지 않고 `freeblks`를 바꾼 AGF는 CRC error로 잡혔다. |

## 한계점

다음은 확인하지 못했다.

- dirty transaction에서 read가 실패하면 shutdown하는 이유. dirty transaction은 cancel할 수 없기 때문일 것이다(추정).
- AGI buffer가 언제 unpin되어 AIL에 들어갔는지. 관찰 출력에는 그 시점이 없다.
- recovery가 CRC와 LSN이 맞지 않는 log의 buffer 내용을 어떻게 다루는지.
- write verifier가 실제로 실패하는 경우. 쓰기 쪽 실패를 일부러 만드는 방법을 찾지 못했고, 관찰 image에 commit 때 검사를 하는 debug config가 있는지도 확인하지 못했다.
- online fsck가 쓰는 in-memory buftarg(xfile 위의 btree)의 세부. 같은 cache code를 쓰고 bio만 건너뛴다는 설계 주석만 읽었다.
- AGF 관찰에서 per-AG metadata reserve pool을 잡는 단계 안의 어느 호출이 AGF를 읽었는지.
- xfs_db의 crc 줄에 붙는 `(correct)` 표시의 기준.
- freeze 때 inode buffer의 write 8번과 cluster flush 7번의 차이, `xfs_buf_delwri_queue` 19번과 buffer item push 17번의 차이, 그리고 inode cluster buffer 8개가 inode item push와 buffer item push 가운데 어느 쪽으로 쓰였는지.
- file을 만드는 동안 inode cluster buffer가 `xfs_trans_log_buf`에 찍혔는지. 출력이 daddr별로 나뉘어 있지 않다.

## 실험 환경

공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.

### 이 page의 image

- 관찰마다 guest의 /dev/shm에 1 GiB file을 만들어 loop device로 붙이고, option 없는 mkfs.xfs로 새로 만들었다. 출력에 나온 geometry는 `isize=512`, `agcount=4`, `crc=1`, `finobt=1`, `sparse=1`, `rmapbt=1`이다.
- 관찰은 한 번씩 돌렸고, 시간은 재지 않았다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예시 |
|---|---|---|---|
| file 200개를 만들 때의 buffer | mount한 뒤 event를 켜고 trace_marker로 세 구간을 나눴다. 세 구간은 file 200개 touch(A), `sync`(B), `xfs_freeze -f`와 `-u`(C)다. 구간이 끝날 때마다 per-fs stats(`/sys/fs/xfs/<dev>/stats/stats`)의 `buf`와 `icluster` 줄을 읽었다. top-level tracefs를 썼고 trace buffer는 8192 KB였다. cpu 8개의 overrun은 모두 0이었다. 방법은 fstests common/tracing과 tests/xfs/177에서 가져왔다 | ```text<br>xfs_buf_find xfs_buf_get xfs_buf_read<br>xfs_trans_read_buf xfs_trans_get_buf xfs_trans_log_buf<br>xfs_buf_item_format xfs_buf_item_pin xfs_buf_item_release<br>xfs_buf_item_push xfs_buf_delwri_queue xfs_buf_submit<br>xfs_inode_pin xfs_inode_push_pinned<br>``` | cache 단위, AGI buffer의 상태, 구간별 log와 write, inode cluster flush |
| AGF의 freeblks를 0으로 바꾼 mount | unmount 상태에서 `xfs_db -x`로 AGF 0의 `freeblks`를 0으로 바꾸고 mount했다. 한 번은 `write -d`, 한 번은 `write -c`로 바꿨다. 바꾸기 전후에 xfs_db로 다시 읽고, mount 뒤 dmesg를 모았다. 방법은 fstests tests/xfs/520에서 가져왔다 | 없음 | 실패의 보고 표의 AGF 두 줄 |
| AGI를 무작위로 망가뜨린 mount | mount해 file 64개를 만들고 unmount했다. 그 뒤 `xfs_db -x`의 `blocktrash`로 AGI 0의 bit를 무작위로 바꾸고(seed 고정, CRC는 다시 계산하지 않음) mount했다. mount가 실패해 이어서 하려던 touch 6개는 돌지 않았다. 방법은 fstests tests/xfs/087에서 가져왔다 | 없음 | 실패의 보고 표의 AGI 줄 |

### 출력 읽는 법

- stats의 `buf` 줄에는 get, create, get_locked, get_locked_waited, busy_locked, miss_locked, page_retries, page_found, get_read 순서로 값 9개가 있다. file 200개 구간의 값은 앞뒤 줄의 차로 get 1265-37, get_locked 1231-19, miss_locked 34-18, get_read 26-18이다. 이 값은 event `xfs_buf_get`과 `xfs_buf_find`의 수와 맞는다.
- `icluster` 줄의 세 값은 `xs_iflush_count`, `xs_icluster_flushcnt`(cluster 수), `xs_icluster_flushinode`(센 inode 수)다. 첫 값은 이 tree에서 올리는 곳이 없고, 출력에서도 0이다.
- event `xfs_trans_read_buf`는 transaction이 그 buffer를 처음 붙일 때만 찍힌다.
- item 종류와 상관없이 성공한 push를 찍는 `xfs_ail_push`는 켜지 않았다.
- event `xfs_buf_submit`은 write verifier보다 먼저 찍히고, `bufops` 칸은 그 buffer에 붙은 ops다. 이 관찰에서는 dmesg를 모으지 않아 write verifier의 결과는 보지 않았다.
- trace의 daddr와 bbcount는 512 byte 단위다. daddr 0x2는 block 0의 sector 2(AGI), 0x18은 block 3, 0x20은 block 4, 0x80은 block 16이다. bbcount 0x20은 16 KiB다.
- 30초 주기 log worker나 xfsaild가 A, B 구간 중에 AIL을 push할 수 있지만, 이 실행에서는 A와 B 구간에 WRITE submit이 없었다.
- `xfs_db`의 `write`에서 `-c`는 write verifier와 CRC 재계산을 둘 다 건너뛰고, `-d`는 verifier만 건너뛰고 CRC는 다시 계산한다. 그래서 `-d`로 바꾼 AGF는 CRC가 맞고 구조 검사에서 실패한다.
- xfs_db의 crc 줄에 붙는 `(correct)`는 CRC error를 낸 직후에도 붙어 있어서 근거로 쓰지 않았다.
- AGF와 AGI 출력에서 hex dump 줄은 뺐다.

### 원본 출력: file 200개를 만들 때의 buffer

(원본 출력 "buffer cache 통계와 구간별 event 수": `raw/14-buffer-verifier.md`)

(원본 출력 "구간별 xfs_buf_submit의 ops와 flags": `raw/14-buffer-verifier.md`)

(원본 출력 "icluster 통계와 inode buffer의 submit": `raw/14-buffer-verifier.md`)

### 원본 출력: AGF의 freeblks를 0으로 바꾼 mount

(원본 출력 "AGF freeblks 변경 (write -d, CRC 재계산)": `raw/14-buffer-verifier.md`)

(원본 출력 "AGF freeblks 변경 (write -c, CRC 재계산 없음)": `raw/14-buffer-verifier.md`)

### 원본 출력: AGI를 무작위로 망가뜨린 mount

(원본 출력 "AGI blocktrash 뒤 mount": `raw/14-buffer-verifier.md`)
