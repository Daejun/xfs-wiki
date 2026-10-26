# create, link, rename

> XFS 전반 wiki, 묶음 "흐름". 기준 kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
> 원본 out/xfs/24-create-link-rename.xhtml. code 근거(파일:줄, 기준 tree)는 12-xfs-general/24-create-link-rename.md, 접힌 원본 출력은 raw/24-create-link-rename.md.

## 개요

data device 하나와 internal log만 쓰고 parent pointer를 켠 file system에서 이름 하나를 만들거나 옮길 때를 본다. [inode](06-inode.md), [directory와 symlink](07-directory.md), [xattr와 parent pointer](08-xattr-parent.md), [transaction](11-transaction.md)을 먼저 읽어 두면 따라가기 쉽다.

- create, link, rename은 inode, directory entry, parent pointer를 transaction 하나에서 함께 바꾼다. 그래서 log에 이름 없는 inode가 commit되는 일은 없다. 예외는 whiteout rename이 미리 만들어 두는 tmpfile inode 하나다(O_TMPFILE은 원래 이름 없는 file을 만드는 기능이라 셈에서 뺀다).
- whiteout rename 말고도 transaction이 둘 이상으로 나뉘는 경우가 세 가지 있다. inode chunk를 새로 할당할 때, commit 안에서 deferred operation이 roll할 때, create 뒤에 security xattr와 ACL을 설정할 때다. 세 경우 모두 inode 하나를 고르는 일과 이름 추가는 같은 transaction에 들어가고, 첫째에서는 chunk 할당만 앞 transaction으로 갈라진다.
- link와 rename은 관련 inode의 ILOCK을 inode 번호 순으로 잡는다. parent pointer는 shortform attr fork 안에서 끝나면 같은 transaction에서 바로 바뀌고, 안 되면 intent로 미룬다. 관찰한 create, link, rename 열 경우에서는 모두 바로 바뀌었다.

예의 값은 새로 만든 1 GiB file system에서 얻었다. 이름 예는 `-n parent=1`을 주어, inode chunk 예는 option 없이 만들었다.

## create

XFS는 inode 할당과 이름 추가를 묶어서 commit한다. transaction을 먼저 잡고, 고칠 것을 모두 그 transaction에 join한 뒤 한 번에 commit한다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### touch d1/f2 한 번

shortform directory d1(inode 131)에 file f2를 만들면 transaction 하나에서 다음이 바뀌었다. roll은 없었다.

| 대상 | 바뀌는 것 | 찍힌 event |
|---|---|---|
| AGI, inobt, finobt | inode 1791을 할당한다. AGI의 free inode 수와 inobt, finobt의 free bit가 바뀐다 | 없음(code로만 확인) |
| 새 inode 1791 | inode core를 초기화하고, attr fork에 parent pointer(부모 131, 이름 f2, 값 12 byte)를 넣는다 | `xfs_attr_sf_add`(filter `PARENT`) |
| 부모 d1 | shortform entry f2 → 1791을 더하고 mtime, ctime을 바꾼다 | `xfs_dir2_sf_addname` |

`mkdir d1/sub`도 같은 모양이었다. block, leaf, node 형식의 directory에 만들면 addname event만 형식에 맞게 바뀌고(`xfs_dir2_block_addname` 등) parent pointer 쪽은 같았다.

### inode가 놓이는 AG

새 directory는 부모의 AG를 보지 않고, mount 전체에서 하나씩 늘어나는 rotor로 시작 AG를 고른다. 그 밖의 inode는 부모 directory의 AG에서 시작한다. 예의 file system에서는 inode 번호를 2^19로 나눈 몫이 AG 번호가 된다([XFS 전반](00-start.md)).

| 만든 것 | 부모 | inode 번호(trace의 표기) | AG |
|---|---|---|---|
| d1, d2, blk, lf, nd(`mkdir` 한 번) | root | 131(0x83), 524416(0x80080), 1179776(0x120080), 1572992(0x180080), 132(0x84) | 0, 1, 2, 3, 0 |
| d1/sub(`mkdir`) | d1 | 524467(0x800b3) | 1 |
| d1/f2 | d1 | 1791(0x6ff) | 0 |
| blk/new | blk | 1179920(0x120110) | 2 |
| lf/new | lf | 1573419(0x18022b) | 3 |
| nd/new | nd | 1793(0x701) | 0 |

### create의 순서

1. **예약.** create용(mkdir이면 mkdir용) log reservation, block 수, quota를 한 번에 예약한다. log reservation 한 몫은 chunk 할당, 이름 추가, (parent pointer가 켜져 있으면) xattr 추가 가운데 가장 큰 쪽이다. permanent reservation이고 log count는 2다(parent pointer가 켜져 있으면 xattr 추가의 log count를 더한다).
2. **부모 directory는 inode를 고른 뒤에 transaction에 넣는다.** 부모의 ILOCK을 먼저 잡고 `xfs_dialloc`으로 inode를 고른 다음, 새 inode를 잠근 채 초기화하고 나서야 부모를 join한다. 주석에 따르면 inode를 고르는 도중에 transaction이 commit될 수 있기 때문이다. lock flag 0으로 join하므로 부모의 ILOCK은 roll이 있어도 create가 끝날 때까지 잡혀 있다(code로만 확인).
3. **이름, 시각, parent pointer.** `xfs_dir_create_child`가 directory 형식에 맞는 addname을 부르고, 부모의 mtime과 ctime을 바꾸고, parent pointer를 더한다. directory가 shortform에서 넘치면 block 형식으로 바꾸는 일도 같은 transaction에서 한다. mkdir이면 `.`과 `..`을 초기화하고 부모의 nlink를 1 올린다.
4. **새 file의 parent pointer는 intent로 미루지 않는다.** parent pointer가 켜져 있으면 inode를 초기화할 때 빈 attr fork를 미리 만든다. 빈 attr fork는 shortform으로 취급되므로 parent pointer 추가는 같은 transaction에서 끝난다. -ENOSPC가 날 때만 attr intent를 건다.
5. **commit.** permanent transaction이면 commit 전에 deferred operation을 마저 처리한다. 이때는 intent를 지금 transaction에 넣고 roll한 뒤 다음 transaction에서 작업을 끝낸다. 그래서 이름과 inode의 묶음은 intent와 함께 앞 transaction에 들어간다(code로만 확인).

**묶이지 않는 것.** security xattr와 ACL은 create가 commit된 뒤에 따로 설정한다. 설정에 실패하면 방금 만든 file을 별도 transaction으로 지운다(주석: "we must back out"). 그래서 이 부분은 원자적이지 않다.

code: xfs_inode.c, xfs_trans.c, xfs_trans_resv.c, xfs_dir2.c, xfs_inode_util.c, xfs_attr.c, xfs_iops.c

## inode chunk를 새로 할당하는 create

AG에 free inode가 없으면 create는 transaction 둘로 나뉜다. 앞 transaction에서 inode chunk를 할당하고 AGI buffer를 쥔 채 roll하면, 뒤 transaction에서 inode를 고른다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### root chunk가 찬 뒤

새 file system의 root directory 아래에 file 64개(dummy1~dummy64)를 만들면 AG 0의 AGI는 이렇게 바뀐다. mkfs 직후 첫 chunk(inode 128~191)의 64개 가운데 3개는 mkfs가 이미 썼다. 그중 하나가 root인 128이다.

| 시점 | 만든 inode | count | freecount | newino |
|---|---|---|---|---|
| mkfs 직후 |  | 64 | 61 | 128 |
| dummy61까지 | 131~191 | 64(계산) | 0(계산) | 128(계산) |
| dummy62의 앞 transaction | 새 chunk 192~255 | 128(계산) | 64(계산) | 192(계산) |
| dummy62의 뒤 transaction | 192 | 128(계산) | 63(계산) | 192(계산) |
| dummy64와 umount 뒤 | 193, 194 | 128 | 61 | 192 |

- 가운데 세 행의 count, freecount, newino는 AGI를 읽은 값이 아니라 code의 규칙으로 계산한 값이다.
- AGI buffer를 쥐는 event(`xfs_trans_bhold`)와 roll은 dummy62의 create에서만 이어서 찍혔다.
- umount 뒤 inobt에는 record가 둘 있었고, finobt에는 free inode가 남은 둘째 chunk의 record만 있었다.
- 새 chunk는 full chunk였다(holemask 0, count 64). sparse record를 합칠 때 찍히는 `xfs_irec_merge_pre`도 없었다.
- roll은 dummy20의 create 뒤에도 bhold 없이 한 번 찍혔다. chunk는 dummy62에서 하나만 늘었으므로 이 roll은 chunk 할당의 roll이 아니다. root directory가 shortform에서 block 형식으로 커지며 생긴 rmap intent를 commit 안에서 처리한 roll이다(추정).

**도식: 새 chunk를 할당하는 create의 transaction 둘**

```text
create dummy62 (log ticket 하나)

tx 1 : 예약 -> 부모 ILOCK 잡음
       AGI lock -> free inode 0 -> inode chunk 할당 (inode 192~255)
       AGF, free space btree, rmapbt, inobt, finobt, ICREATE item
       AGI bhold -> roll (tx 1 commit)
                      |  같은 log ticket
                      v
tx 2 : AGI 다시 join -> inode 192를 고름 -> 새 inode 초기화
       -> 부모 join -> root에 entry (dummy62, 192) -> parent pointer
       -> commit
부모 ILOCK 풂

부모는 tx 1에 들어가지 않지만 ILOCK은 roll 동안에도 잡혀 있다.
```

### chunk 할당의 순서

1. **chunk 할당.** AGI buffer를 lock하고 free inode가 0이면 `xfs_ialloc_ag_alloc`이 앞 transaction에서 chunk를 할당한다. AG를 고르는 규칙과 chunk 자리를 찾는 순서는 [inode](06-inode.md)에 있다.
2. **v3 inode이면 inode buffer 내용을 log에 쓰지 않는다.** ICREATE log item 하나만 넣고, 초기화한 inode cluster buffer는 "ordered"로만 표시한다. log recovery는 ICREATE item을 보고 같은 초기화를 다시 한다.
3. **roll.** `xfs_dialloc_roll`은 AGI buffer를 commit 뒤에도 쥐도록 표시하고(`xfs_trans_bhold`), quota 정보를 다음 transaction으로 넘긴 뒤, `xfs_trans_roll`로 앞 transaction을 commit한다. 주석에 따르면 다른 할당이 방금 만든 free inode를 가져가지 못하게 하려고 AGI를 쥔다.
4. **inode 고르기.** 다음 transaction은 같은 log ticket으로 log 공간을 다시 받고, AGI를 다시 붙여 free inode 하나를 골라 가져간다. 그 뒤는 create와 같다.

같은 log ticket으로 transaction을 이어 가는 이 방식을 rolling transaction이라 한다([transaction](11-transaction.md)). create는 이 roll 때문에 log count를 2로 잡는다.

| transaction | commit하는 곳 | 들어 있는 것 |
|---|---|---|
| 앞 | roll | AGF, free space btree, rmapbt, AGI(count, freecount, newino), inobt와 finobt record, ICREATE log item, ordered inode buffer, superblock의 inode 계수 |
| 뒤 | create의 commit | AGI의 freecount 감소, inobt와 finobt의 free bit, 새 inode core, directory entry, 부모 inode core, parent pointer xattr, quota |

- 두 transaction 사이에 crash가 나면 chunk는 할당되어 있고 모든 inode가 free인 상태로 복구된다. 이름 없는 inode는 생기지 않는다(code로만 확인).

code: xfs_ialloc.c, xfs_trans.c, xfs_icreate_item.c, xfs_alloc.c

## link와 rename

link와 rename은 관련 inode의 ILOCK을 모두 inode 번호 순으로 잡고, 이름, nlink, parent pointer를 transaction 하나에서 바꾼다. 이 장의 동작은 code로 확인했고, 예의 값은 관찰한 것이다.

### lock

VFS가 먼저 directory와 대상 inode의 `i_rwsem`을 잡고, 그 뒤 XFS가 transaction 안에서 ILOCK을 잡는다.

|  | link | rename |
|---|---|---|
| ILOCK | 둘 다 exclusive | 2~5개 모두 exclusive |
| 순서 | inode 번호가 작은 쪽 먼저 | `xfs_sort_for_rename`이 source dir, target dir, source inode, target inode, whiteout inode를 모아 inode 번호 순으로 정렬한다. 같은 inode는 한 번만 잡는다 |
| 먼저 잡은 inode가 AIL에 있을 때 | 둘째 inode는 trylock하고, 실패하면 풀고 다시 시도한다(5번마다 `delay(1)`) | 같다 |
| AGI | O_TMPFILE inode를 unlinked list에서 뺄 때 | whiteout inode와, nlink가 0이 되는 target(또는 덮이는 directory)의 AGI를 AG 번호 순으로 directory 작업 전에 잡는다 |

- whiteout rename `d2/f2-longer-name → d1/wo`에서는 131(d1), 1791(source), 1792(whiteout), 524416(d2) 순으로 잡는다(규칙을 적용한 순서이고 관찰하지 않음).
- 먼저 잡은 inode가 AIL에 있으면 log tail을 밀어야 하는 다른 transaction이 그 inode를 기다리고 있을 수 있어서 trylock한다.
- directory 작업이 block을 할당하면 AGF를 잡는다. AGI는 AGF보다 먼저 잡아야 하므로(주석) rename은 AGI를 directory 작업 전에 잡는다.
- 모든 inode를 lock flag 0으로 join하므로 caller가 commit 뒤에 푼다. rename은 잡은 역순으로 풀고 중복은 건너뛴다.

### link가 바꾸는 것

1. 예약 block이 0이면 directory에 공간이 있는지만 먼저 본다.
2. source의 nlink가 0이면(O_TMPFILE에 `linkat`) AGI의 unlinked list에서 뺀다.
3. directory entry를 더하고 directory의 시각을 바꾼다. source의 nlink를 1 올리고, parent pointer를 하나 더 넣는다.
4. 둘째 parent pointer는 source의 attr fork가 shortform이고 자리가 있으면 바로 들어간다. 아니면 attr intent로 미뤄 commit이 roll하며 처리한다(code로만 확인).

directory에는 hard link를 걸 수 없다. VFS가 -EPERM으로 먼저 막고, XFS는 ASSERT로 그것을 가정한다.

### rename이 바꾸는 것

1. transaction을 더럽히기 전에 검사한다. target이 없고 예약이 0이면 공간을 보고, target이 directory이면 비어 있고 nlink가 2 이하인지 본다.
2. whiteout이면 whiteout inode를 unlinked list에서 빼고 nlink를 1 올린다.
3. target이 없으면 이름을 새로 만든다(directory를 다른 부모로 옮기면 target 부모의 nlink를 1 올린다). target이 있으면 entry의 inode 번호만 바꾸고 덮인 inode의 nlink를 1 내린다(directory면 한 번 더).
4. directory를 다른 부모로 옮기면 그 directory의 `..`을 새 부모로 바꾼다.
5. source inode의 ctime과 core를 log하고, 필요하면 source 부모의 nlink를 내린다. source entry는 whiteout이면 whiteout inode 번호로 바꾸고, 아니면 지운다.
6. parent pointer는 whiteout inode에 source 부모의 이름으로 더하고, source inode는 (source 부모, 옛 이름)에서 (target 부모, 새 이름)으로 바꾸며, 덮인 target inode에서는 지운다.

parent pointer 값은 늘 12 byte(inode 번호와 generation)이므로, 이름 길이가 같을 때만 shortform attr fork 안에서 제자리로 덮는다. 다르면 -ENOSPC를 돌려 지운 뒤 다시 넣고, 그것도 안 되면 attr intent를 건다.

**RENAME_EXCHANGE**는 두 entry의 inode 번호를 서로 바꾸고, directory가 섞여 있으면 `..`과 nlink를 맞추고, parent pointer를 두 번 바꾼다.

**RENAME_WHITEOUT**은 whiteout inode를 rename transaction 전에 tmpfile로 따로 만든다. 그 transaction은 inode를 할당하고 unlinked list에 넣은 뒤 commit한다. 그 사이에 crash가 나면 log recovery가 unlinked inode를 정리한다(주석). 그래서 rename 하나가 commit을 두 번 이상 하고, tmpfile을 만들 때 chunk 할당이 끼면 roll이 더 붙는다. 이 tmpfile의 부모로는 target dir를 넘기므로, whiteout entry는 source dir에 들어가지만 inode의 AG와 상속 속성은 target dir를 따른다. 관찰에서도 whiteout inode 1792는 source d2의 AG 1이 아니라 target d1과 같은 AG 0에 놓였다.

### inode 1791의 이름이 바뀌는 차례

shortform directory d1(131), d2(524416)에서 f2(1791)에 아래 명령을 차례로 했다. d1에는 file f1(133)이 미리 있었다. 이 관찰 내내 `xfs_attr_defer_add`와 `xfs_defer_trans_roll`은 찍히지 않았다.

| 명령 | directory entry | 1791의 바뀐 parent pointer (부모, 이름) | attr event |
|---|---|---|---|
| `touch d1/f2` | d1: f2 → 1791 | (131, f2) | add |
| `ln d1/f2 d2/f2ln` | d2: f2ln → 1791 | (524416, f2ln) | add |
| `mv d1/f2 d2/f2` | d2: f2를 더함, d1: f2를 지움 | (131, f2) → (524416, f2), 이름 길이 2 그대로 | replace(제자리) |
| `mv d2/f2 d2/f2-longer-name` | d2: f2-longer-name을 더하고 f2를 지움 | (524416, f2) → (524416, f2-longer-name), 길이 2 → 14 | replace, remove, add |
| `renameat2 -w d2/f2-longer-name d1/wo` | d1: wo → 1791, d2: f2-longer-name → 1792(whiteout) | (524416, f2-longer-name) → (131, wo). 1792에는 (524416, f2-longer-name)을 더함 | replace, remove, add |
| `renameat2 -x d1/f1 d2/f2ln` | d1: f1 → 1791, d2: f2ln → 133 | (524416, f2ln) → (131, f1), 길이 4 → 2. 133은 반대 | inode마다 replace, remove, add |

- whiteout rename에서는 whiteout inode 1792를 만드는 transaction이 rename transaction보다 먼저 있었고, 그 안에서 `xfs_dialloc`이 roll을 한 번 했다. 1792가 1791의 다음 번호이고 64의 배수이므로, 이 roll은 새 chunk를 할당한 뒤에 한 것이다(추정).
- 같은 조건의 새 file system에서 directory 131 아래에 file1을 만들고, 524416으로 mv한 뒤, 131에 file1_link로 ln했다. parent pointer는 (131, file1) 하나에서 mv 뒤 (524416, file1) 하나로 바뀌었고, ln 뒤 (131, file1_link)가 더해져 둘이 되었다. umount 뒤 nlink는 2였고, shortform attr fork에 parent entry가 둘 있었다.

code: xfs_inode.c, xfs_trans.c, xfs_dir2.c, xfs_attr.c, xfs_attr_leaf.c, fs/namei.c

## 갈라지는 곳

이 표는 code로 확인했다.

| 조건 | 바뀌는 것 |
|---|---|
| parent pointer 꺼짐 | parent pointer 단계를 건너뛰고, 예약 없는 link와 rename을 허용한다. parent pointer 때문에 attr fork를 미리 만들지는 않는다. ACL이 있거나 LSM이 superblock 정보를 단 kernel이면 create, mkdir, mknod는 그래도 미리 만든다. |
| `wsync` 또는 `dirsync` mount | commit을 동기로 한다(log force까지). |
| mkdir | mkdir용 reservation(log count 3, parent pointer가 켜져 있으면 xattr 추가의 log count를 더한다), `.`과 `..` 초기화, 부모 nlink 1 증가. AG는 rotor로 고른다. |
| O_TMPFILE | directory entry가 없고 unlinked list에 넣는다. `O_EXCL`이면 attr fork도 만들지 않는다. unlinked list는 [inode 수명](15-inode-lifecycle.md)에 있다. |
| inode 수 상한에 걸림 | chunk 할당 없이 이미 있는 free inode만 찾는다. |
| 공간이 1% 아래 | AG를 처음 한 바퀴 돌 때는 chunk를 할당하지 않고 빈 inode가 있는 AG만 찾는다. 못 찾으면 다시 한 바퀴 돌며 할당을 허용한다. |
| 공간 예약 실패(link, rename) | 예약 0으로 다시 시도한다. parent pointer가 켜져 있으면 실패로 돌려준다. |

metadir이 꺼진 경우만 보았고, reflink된 inode, realtime device, zoned, quota 경로는 다루지 않았다. create의 quota 예약은 [quota](30-quota.md)에, 나머지 경로와의 차이는 [reflink, COW, dedupe, exchange-range](26-reflink-cow.md), [realtime, rtgroup, metadir](28-realtime.md), [zoned XFS](29-zoned.md)에 있다.

code: xfs_inode.c, xfs_ialloc.c, xfs_iops.c, xfs_inode_util.c, xfs_trans.c

## 한계점

다음은 확인하지 못했다.

- `xfs_trans_roll` 주석의 조건과 chunk 할당 roll이 어떻게 맞는지. 주석에 따르면 roll 동안 잠가 둔 것은 앞뒤 transaction 모두에 log되어야 한다. 그런데 chunk 할당의 roll 동안에는 앞 transaction에 없는 부모의 ILOCK이 잡혀 있다.
- chunk를 새로 할당한 직후 `xfs_dialloc_ag`의 finobt 선택 경로가 어느 inode를 고르는지.
- block, leaf, node 형식 directory의 addname이 directory block을 새로 할당할 때 같은 transaction에서 끝나는지, deferred operation이 생기는지.
- attr intent로 미룬 parent pointer가 commit 때 몇 번 roll하는지. 큰 xattr가 있는 file의 link는 관찰하지 않았다.
- security xattr나 ACL 설정이 실패해 create를 되돌리는 사이에 crash가 나면 무엇이 남는지.
- whiteout inode의 부모로 target dir를 넘기는 의도.
- ICREATE log item을 log에서 직접 보는 방법(관찰하지 않음).
- function_graph에서 `xfs_create` 10건이 빠진 이유. graph에는 file 이름이 없고 10건이 빠져 있어서, graph의 첫 roll(rmap intent를 처리한 roll)이 dummy20 뒤의 roll과 같은 것인지는 맞춰 보지 않았다.

## 실험 환경

- 공통 환경은 [XFS 전반](00-start.md)의 실험 환경에 있다.
- 관찰마다 1 GiB file system을 새로 만들었다. 이름 관찰과 parent pointer 관찰은 `mkfs.xfs -n parent=1`로, inode chunk 관찰은 option 없이 만들었다. 기본 mkfs도 parent pointer, rmapbt, reflink를 켜므로 inode chunk 관찰의 graph에도 rmap intent와 parent pointer 추가가 있다.
- event는 tracefs instance에서 켰다(방법은 [관찰 도구](02-observation-tools.md)에 있다). inode chunk 관찰만 top-level에서 function_graph(`xfs_create` 아래 깊이 6)를 함께 켰다. 관찰은 한 번씩 실행했다.

### 관찰 목록

| 관찰 | 절차 | 켠 event | 본문의 예 |
|---|---|---|---|
| 이름 | `mkdir d1 d2 blk lf nd`, `touch d1/f1` 뒤 fstests `popdir.pl --end`(102, 341, 1638)로 blk, lf, nd를 block, leaf, node 형식으로 키우고 trace를 켰다. 그다음 `touch d1/f2`, `mkdir d1/sub`, 본문 표의 나머지 명령, blk, lf, nd의 touch를 차례로 하고 단계마다 `trace_marker`를 남겼다 | `xfs_create`, `xfs_link`, `xfs_rename`, 형식별 addname, `xfs_dir2_sf_*`, `xfs_attr_sf_*`, `xfs_attr_defer_add`, `xfs_iunlink*`, `xfs_trans_roll`, `xfs_defer_trans_roll` | touch d1/f2 한 번, inode가 놓이는 AG, inode 1791의 이름이 바뀌는 차례 |
| inode chunk | fstests `common/populate`처럼 `popdir.pl`로 root 아래에 file 64개를 만들고, mkfs 직후와 umount 뒤에 `xfs_db`로 AGI 0과 inobt, finobt record를 읽었다 | `xfs_create`, `xfs_trans_bhold`, `xfs_trans_roll`, `xfs_irec_merge_pre` | root chunk가 찬 뒤 |
| parent pointer | fstests xfs/618처럼 file을 만들고 mv, ln 뒤마다 `xfs_io -x -c parent`와 `parent -p`로 읽고, umount 뒤 `xfs_db`로 attr fork를 읽었다 | 없음 | inode 1791의 이름이 바뀌는 차례 |

### 출력 읽는 법

- `xfs_trans_bhold`의 `daddr 0x2`는 AG 0의 AGI buffer다.
- `xfs_trans_roll`의 caller 칸은 function_graph를 켜면 `return_to_handler`로 찍혀 roll을 부른 곳을 알 수 없다. 켜지 않은 이름 관찰에서는 `xfs_dialloc`이 찍혔다.

### 도구에서 생긴 일

- function_graph에는 `xfs_create`가 64건 가운데 54건만 남았다. instance trace에는 64건이 모두 있다.
- `xfs_dialloc_try_ag`, `xfs_dialloc_roll`, `xfs_dialloc_ag`는 inline이라 `available_filter_functions`에 없었고 graph에도 나오지 않는다.

### 원본 출력

#### 이름

(원본 출력 "관찰에 쓴 inode 번호와 file 종류": `raw/24-create-link-rename.md`)

(원본 출력 "create와 mkdir의 event 순서": `raw/24-create-link-rename.md`)

(원본 출력 "link와 rename(같은 길이, 긴 이름)의 event 순서": `raw/24-create-link-rename.md`)

(원본 출력 "whiteout rename과 exchange의 event 순서": `raw/24-create-link-rename.md`)

(원본 출력 "block, leaf, node directory의 create": `raw/24-create-link-rename.md`)

#### inode chunk

(원본 출력 "chunk 할당 전후의 AGI와 inode 번호": `raw/24-create-link-rename.md`)

(원본 출력 "bhold와 roll event": `raw/24-create-link-rename.md`)

(원본 출력 "chunk 할당 뒤 inobt와 finobt record": `raw/24-create-link-rename.md`)

(원본 출력 "function_graph에서 첫 xfs_trans_roll 부근": `raw/24-create-link-rename.md`)

#### parent pointer

(원본 출력 "xfs_io와 xfs_db로 본 parent pointer": `raw/24-create-link-rename.md`)
