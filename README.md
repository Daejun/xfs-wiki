# XFS wiki

Linux kernel의 XFS가 어떻게 동작하는지 code와 관찰로 설명한 한국어 wiki다. 본문 34개 page와 시작 page, 부록으로 되어 있다.

- 기준: kernel 8fde5d1d47f6 (v7.1-rc5), xfsprogs v7.1.1.
- 읽는 순서와 page마다 답하는 질문은 [INDEX.md](INDEX.md)에 있다. 시작 page는 [00-start.md](00-start.md)다.
- `raw/<page>.md`는 본문에서 접어 둔 명령 출력(xfs_db, tracepoint 등)이다.
- page 머리의 "원본"과 "code 근거" 줄이 가리키는 file(작성 원본, `파일:줄` 노트)은 이 저장소에 없다.
