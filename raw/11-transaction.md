# transaction: 원본 출력

> 본문은 ../11-transaction.md. 관찰 image와 조건은 본문의 "실험 환경" 절에 있다.

## xfs_db logres와 kernel의 reservation 계산값 비교

**관찰 출력**

```text
== xfs_db logres (xfsprogs 7.1.1, before mount)
type 0 logres 142840 logcount 5 flags 0x4
type 1 logres 226176 logcount 5 flags 0x4
type 2 logres 428160 logcount 5 flags 0x4
type 3 logres 247204 logcount 5 flags 0x4
type 4 logres 230820 logcount 5 flags 0x4
type 5 logres 283044 logcount 6 flags 0x4
type 6 logres 281892 logcount 5 flags 0x4
type 7 logres 201216 logcount 2 flags 0x4
type 8 logres 281892 logcount 6 flags 0x4
type 9 logres 219512 logcount 2 flags 0x4
type 10 logres 2168 logcount 0 flags 0x0
type 11 logres 56832 logcount 2 flags 0x4
type 12 logres 91512 logcount 2 flags 0x4
type 13 logres 760 logcount 0 flags 0x0
type 14 logres 225408 logcount 1 flags 0x4
type 15 logres 23288 logcount 3 flags 0x4
type 16 logres 21760 logcount 0 flags 0x0
type 17 logres 113792 logcount 3 flags 0x4
type 18 logres 640 logcount 0 flags 0x0
type 19 logres 86520 logcount 2 flags 0x4
type 20 logres 4224 logcount 0 flags 0x0
type 21 logres 6512 logcount 0 flags 0x0
type 22 logres 232 logcount 1 flags 0x0
type 23 logres 147063 logcount 5 flags 0x4
type 24 logres 640 logcount 1 flags 0x0
type 25 logres 760 logcount 0 flags 0x0
type 26 logres 226176 logcount 5 flags 0x4
minlogsize logres 428160 logcount 5
== kernel xfs_log_get_max_trans_res
dev 7:0 logres 428160 logcount 5
== diff kernel resv_calc vs xfs_db logres (type lines)
kernel=27 xfs_db=27
kernel and xfs_db logres identical
```

## 하한 계산용 표의 앞부분과 고른 항목

**관찰 출력**

```text
== kernel xfs_trans_resv_calc_minlogsize
dev 7:0 type 0 logres 142840 logcount 5 flags 0x4
dev 7:0 type 1 logres 226176 logcount 5 flags 0x4
dev 7:0 type 2 logres 428160 logcount 5 flags 0x4
== kernel xfs_log_get_max_trans_res
dev 7:0 logres 428160 logcount 5
```

## xfs_io truncate의 처음 roll들: xfs_log_reserve에서 steady state까지

**관찰 출력**

```text
== xfs_io: events=25713, first 90
xfs_log_reserve tic=0xff1100010ebd0600 ocnt=5 cnt=5 curr=231944 unit=231944 rsv=212992 wr=212992
xfs_log_reserve_exit tic=0xff1100010ebd0600 ocnt=5 cnt=5 curr=231944 unit=231944 rsv=1372712 wr=1372712
xfs_trans_alloc tid=9725394f flags=0x4 caller=xfs_vn_setattr_size+0x1eb/0x4f0
xfs_defer_trans_roll tp=0000000080eb1345 caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=9725394f flags=0x5 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=9725394f flags=0x25 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=5 curr=222892 unit=231944 rsv=1372712 wr=1372712
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=4 curr=231944 unit=231944 rsv=1149820 wr=1149820
xfs_defer_trans_roll tp=000000007b911a3e caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=97253950 flags=0x85 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253950 flags=0xa5 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=4 curr=223668 unit=231944 rsv=1149820 wr=1149820
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=3 curr=231944 unit=231944 rsv=926152 wr=926152
xfs_defer_trans_roll tp=0000000080eb1345 caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=97253951 flags=0x85 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253951 flags=0xa5 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=3 curr=216852 unit=231944 rsv=926152 wr=926152
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=2 curr=231944 unit=231944 rsv=709300 wr=709300
xfs_defer_trans_roll tp=000000002a0884a2 caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=97253952 flags=0x85 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253952 flags=0xa5 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=2 curr=227812 unit=231944 rsv=709300 wr=709300
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=1 curr=231944 unit=231944 rsv=481488 wr=481488
xfs_defer_trans_roll tp=0000000080eb1345 caller=xfs_defer_finish+0x3e/0x70
xfs_trans_roll tid=97253953 flags=0x85 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253953 flags=0xa5 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=1 curr=227796 unit=231944 rsv=481488 wr=481488
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=253692 wr=253692
xfs_log_ticket_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=485636 wr=253692
xfs_log_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=485636 wr=253692
xfs_log_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=485636 wr=485636
xfs_defer_trans_roll tp=000000007b911a3e caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=97253954 flags=0x5 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253954 flags=0x25 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=227544 unit=231944 rsv=485636 wr=485636
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=258092 wr=258092
xfs_log_ticket_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=490036 wr=258092
xfs_log_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=490036 wr=258092
xfs_log_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=490036 wr=490036
xfs_defer_trans_roll tp=0000000080eb1345 caller=xfs_defer_finish_noroll+0x243/0x440
xfs_trans_roll tid=97253955 flags=0x85 caller=xfs_defer_trans_roll+0x77/0x130
xfs_trans_commit tid=97253955 flags=0xa5 caller=xfs_trans_roll+0x38/0xe0
xfs_log_ticket_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=227812 unit=231944 rsv=490036 wr=490036
xfs_log_ticket_regrant_sub tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=262224 wr=262224
xfs_log_ticket_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=494168 wr=262224
xfs_log_regrant tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=494168 wr=262224
xfs_log_regrant_exit tic=0xff1100010ebd0600 ocnt=5 cnt=0 curr=231944 unit=231944 rsv=494168 wr=494168
```

## xfs_io truncate의 t_cnt 분포와 ticket 크기

**관찰 출력**

```text
== xfs_io: t_cnt at xfs_log_ticket_regrant (count by value)
   3210 0
      1 1
      1 2
      1 3
      1 4
      1 5
== xfs_io: ocnt and unit at xfs_log_reserve (count by value)
      1 ocnt=5 unit=231944
```

## task별 event 수

**관찰 출력**

```text
== event counts by task
-- xfs_trans_alloc
      3 kworker
      1 rm
      2 umount
      1 xfs_io
-- xfs_trans_roll
   3217 kworker
   3215 xfs_io
-- xfs_trans_commit
   3220 kworker
      1 rm
      2 umount
   3216 xfs_io
-- xfs_trans_cancel
-- xfs_defer_trans_roll
   3217 kworker
   3215 xfs_io
-- xfs_log_reserve
      3 kworker
      1 rm
      3 umount
      1 xfs_io
-- xfs_log_regrant
   3213 kworker
   3211 xfs_io
-- xfs_log_ticket_regrant
   3217 kworker
   3215 xfs_io
-- xfs_log_ticket_regrant_exit
   3213 kworker
   3211 xfs_io
-- xfs_log_ticket_ungrant
      8 kworker
      1 rm
      3 umount
      1 xfs_io
-- xfs_log_grant_sleep
```

## truncate와 rm 전후의 stats

**관찰 출력**

```text
== stats before
trans 0 7716 0
push_ail 7716 0 1 0 0 0 0 0 0 0
== truncate b (xfs_io task)
== stats after truncate
trans 0 10932 0
push_ail 10932 0 2 0 0 0 0 0 0 0
== rm c (inodegc worker)
== stats after rm
trans 0 14153 0
push_ail 14153 0 41 0 0 0 0 0 0 0
```

