---
name: ticket-review
description: 다른 사람의 PR 을 (1) 기능 요구사항 위반, (2) 문법·정적 오류, (3) 코드 품질 개선 관점으로 검토하고 리뷰 의견을 작성합니다. 코드는 수정하지 않으며 자명한 수정도 의견으로만 출력. "PR 리뷰해줘", "#N 리뷰", "리뷰 의견 써줘" 요청 시 사용. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Review

다른 사람의 PR 을 **3축 (기능 / 문법 / 품질)** 으로 검토하고 리뷰 의견을 출력합니다. 코드는 고치지 않습니다 — 자명한 수정도 의견으로 남기고, 반영은 PR 작성자가 합니다.

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` 로드 → minimal 4 키 검증. 상세: `~/.claude/skills/ticket-plan/CONFIG.md`.

`${project_root}` 는 현재 worktree 가 아니라 **메인 worktree 루트**다 — 이 스킬은 `{N}-{slug}` worktree 안에서 호출되는 일이 잦은데 `.claude/` 는 대개 gitignore 라 거기에 없다.

PR 조회와 보드 동기화에 `gh` 를 쓰므로 `project.gh_account` 계정 라우팅을 거칩니다. commit·push 가 없어 `ssh_host` / `git_email` 은 무시. 빌드/테스트/린트 **명령은 프로젝트 CLAUDE.md** 의 "Common Commands" / "Testing" / "Linting" 섹션을 참조.

## 입력

1. **PR 번호 / URL**: `gh pr view {N} --json title,body,author,baseRefName,headRefName,commits,closingIssuesReferences` + `gh pr diff {N}`. 연결 Issue 의 요구사항도 읽음
2. **파일 / 디렉터리**: 지정된 대상
3. **인자 없음**: 현재 working directory 의 미커밋 변경사항 (`git status`, `git diff`, untracked) — `ticket-implement` Phase 6.5 의 경로
4. 변경사항이 비어 있으면 사용자에게 "어디를 검토할지" 묻기

### PR worktree

PR 은 전용 worktree 에 checkout 해서 검토합니다 — 현재 worktree 의 브랜치·미커밋 변경을 건드리지 않고 PR 버전의 파일을 읽기 위함.

```bash
wt=../${worktree.prefix}-pr-{N}

# 최초
git fetch origin pull/{N}/head
git worktree add --detach $wt FETCH_HEAD

# 이미 있으면 (재리뷰) 갱신 — FETCH_HEAD 는 worktree 별이라 fetch 도 그 안에서 한다
git -C $wt fetch origin pull/{N}/head
git -C $wt checkout --detach FETCH_HEAD
```

- **`pr-` 접두**: Issue worktree (`${worktree.prefix}-{N}-{slug}`) 와 구분 — PR 번호와 Issue 번호는 다르다
- **detached HEAD**: 로컬 브랜치를 만들지 않아 작성자 브랜치명과 충돌하지 않고, fork PR 도 같은 명령으로 받는다
- **읽기는 전부 이 worktree 경로 기준**: 메인 worktree 의 파일은 PR 버전이 아니다
- **리뷰 의견 파일**: `$wt/review-pr-{N}.md` — 다른 에이전트 인계용 임시파일. 터미널 출력과 같은 내용, 재리뷰 시 덮어씀. untracked 로 두고 커밋하지 않음
- **검토 후 유지**: 재리뷰·인계에 재사용. 정리는 사용자 요청 시 `ticket-complete` Step 7 절차 (링크 제거 후 `git worktree remove`)

## 검토 축

### 1. 기능 (Functional)
구현이 요구사항/의도와 일치하는가?
- 누락된 acceptance criteria, 빠진 분기
- 호출 컨트랙트 (시그니처/반환값/예외) 어긋남
- 엣지 케이스 누락 (경계값, None/빈값, 예외 경로)
- 스펙/티켓과 다른 동작

### 2. 문법 / 정적 오류 (Syntax / Correctness)
실행이 가능하고 정적으로 옳은가?
- 타입 에러, undefined name, dead import, 누락된 import
- 잘못된 참조, 변수 섀도잉, 잘못된 비교 (`is` vs `==`)
- 명백한 버그 (off-by-one, 잘못된 연산자, 누락된 return/raise, resource leak)
- async/await 미사용 또는 잘못 사용

### 3. 품질 (Quality)
프로젝트 컨벤션·유지보수성에 부합하는가?
- 프로젝트 코딩 컨벤션 (프로젝트 CLAUDE.md, 전역 `~/.claude/CLAUDE.md`, 린터 설정 파일)
- 중복 / 사용되지 않는 코드
- 네이밍, 매직넘버, 로깅 레벨, 에러 메시지
- 불필요한 추상화·과도한 방어 코드 (전역 CLAUDE.md "Simplicity First", "Surgical Changes" 원칙)
- 프로젝트 고유 패턴 (DB / API 호출 방식, 모듈 구조 등)

> **컨벤션 참조**: `CLAUDE.md` (프로젝트 / 전역) 와 프로젝트 설정 파일 (`pyproject.toml`, `eslint.config.*`, `tsconfig.json` 등) 을 우선 참조합니다.

## 리뷰 의견 구성

순서 고정. 내용이 없는 블록은 제목째 생략. PR 이 아닌 대상 (미커밋 변경사항·파일 지정) 은 앞 3개 블록을 생략합니다.

| 블록 | 내용 |
|---|---|
| **목적** | PR 이 푸는 문제와 범위, 1~3문장. 코드에서 확인한 것으로 서술 — PR 본문과 실제가 다르면 그 차이를 적음 |
| **구조의 변경** | 이전 구조 (base) → 변경된 구조 → 개선되는 한계점 표 (`{base} 의 한계` / `개선`, base 코드에서 확인한 것만) → 평가 (적절 / 보완 필요 / 재검토 필요 + 근거 + 남는 비용) |
| **동작의 변경** | 사용자·호출자가 관찰하는 차이, `이전 → 이후`. PR 본문에 없는 부수 변경은 표시 |
| **결함** | 발화 경로를 댈 수 있는 사실 — 버그, 요구사항·컨트랙트 위반, 정적 오류. 머지 전 수정 대상 |
| **검토 필요 사항** | 판단이 갈리는 것 — 설계 선택·트레이드오프, 조건부 위험, 의미가 바뀌는 변경. 의도가 불분명한 코드는 질문으로 |
| **기타 수정사항** | 수정이 하나로 정해지고 동작이 바뀌지 않는 것 — 오타, dead import, 명문화된 컨벤션 위반, 죽은 링크 |
| **참고** | 이 PR 에서 고치지 않아도 되는 것 — 범위 밖·현재 발화 불가·다른 티켓 소관. 소관 (기존 Issue 번호 / 신규 Issue) 명시 |

> 결함인지 헷갈리면 **검토 필요 사항**. 사실로 단정한 지적이 틀리면 리뷰 전체의 신뢰가 깎입니다.

### 문장 규칙

- 존댓말 평서문. 주어는 코드·동작 — 작성자를 주어로 쓰지 않음
- 결함 · 검토 필요 사항: 번호 + 3문장 — **현상** (볼드) → 영향·발화 조건 → 제안. 영향 큰 순
- 제안은 지시가 아닌 방향 — "~하는 방향이 좋겠습니다". 스니펫은 문장으로 수정이 특정되지 않을 때만
- 기타 수정사항 · 참고: 한 줄 — `대상 — 현상 — 권장 조치 또는 소관`
- 근거는 식별자 (함수·타입명, `파일:라인`, 커밋 해시, Issue 번호) 와 개수. "일부"·"여러 곳" 금지
- 확인한 것만 단언, 미확인은 추정이라고 밝힘
- 평가어·강조어 금지 (심각한 / 잘못된 / 반드시 / 훌륭한)

## 실행 단계

1. **범위 확정** — 인자 / git 상태 / 사용자 질의로 대상 파일·hunk 결정. PR 이면 "PR worktree" 준비. 변경된 라인 위주로 컨텍스트 파악 (Read, Grep, 심볼 단위는 Serena)
2. **보드 동기화** — Issue 번호 N 을 해석할 수 있고 (PR 이면 `closingIssuesReferences`, 아니면 worktree 디렉토리명 `${worktree.prefix}-{N}-{slug}` · 현재 브랜치) config 에 `board` 가 있으면 `Set-BoardStatus({N}, in_review)` (CONFIG.md "프로젝트 보드 Status 동기화"). N 미상 또는 보드 미설정이면 skip
3. **3축 검토** — 파일·hunk 마다 점검해 발견 항목 수집 (위치, 문제, 근거)
4. **분류** — 발견 항목을 블록에 배정. 같은 원인의 항목은 하나로 병합
5. **출력** — PR 코멘트에 그대로 붙일 수 있는 마크다운으로 출력하고 종료. PR 이면 같은 내용을 리뷰 의견 파일로 저장하고, 의견 밖에 파일 경로를 한 줄로 알림

## 가드레일

- **코드 수정 · PR 게시 금지**: Edit · commit · push · `gh pr review` / `gh pr comment` 는 사용자가 명시 요청할 때만. 쓰는 파일은 리뷰 의견 파일 하나뿐
- **검토 범위 최소화**: PR 이 만진 부분 + 직접 영향 범위만. 인접 코드의 기존 문제는 참고로만
- **테스트 부재는 지적만**: 대신 작성하지 않음

## 출력 예시

```
## 리뷰 의견 — #128 Binance WS 재연결 백오프

### 목적
WebSocket 이 끊기면 고정 1초 간격으로 무한 재시도해 거래소 rate limit 에 걸리던 문제 (#121) 를 지수 백오프로 바꿉니다. 범위는 `binance_ws.py` 의 재연결 경로와 그 테스트입니다.

### 구조의 변경
**이전 구조 (main).** `_reconnect` 하나가 대기·재접속·구독 복원을 겸했고, 대기 시간은 상수 `RETRY_DELAY` 였습니다.

**변경된 구조.** 대기 시간 계산을 `BackoffPolicy` 로 분리했고, `_reconnect` 는 정책이 준 값만큼 대기한 뒤 재접속합니다. 구독 복원은 `_resubscribe` 로 나뉘었습니다.

| main 의 한계 | 개선 |
|---|---|
| 고정 간격 재시도로 rate limit 초과 (#121) | 지수 백오프, 상한 60초 |
| 대기 로직이 `_reconnect` 안에 있어 단독 테스트 불가 | `BackoffPolicy` 단위 테스트 |

**평가: 적절합니다.** 재시도 간격이 `BackoffPolicy` 한 곳에서 정해지고 `_reconnect` 의 분기가 줄었습니다. 남는 비용은 fetcher 간 불일치 하나이고, 소재는 참고에 적었습니다.

### 동작의 변경
- 재시도 간격: 고정 1초 → 1·2·4…초, 상한 60초
- 재시도 한도: 무제한 → 10회 초과 시 `ConnectionError`. PR 본문에 없는 변경입니다

### 결함
1. **재접속에 성공해도 백오프가 초기화되지 않습니다.** `_reconnect` 의 성공 경로 (`binance_ws.py:97`) 가 `policy.reset()` 을 부르지 않아, 다음 단절은 직전 간격에서 시작합니다. 성공 직후 reset 을 호출하는 방향이 좋겠습니다.

### 검토 필요 사항
1. **한도 초과 예외가 수집 루프를 종료시킵니다.** `run()` 이 `ConnectionError` 를 잡지 않아 10회 실패 후 프로세스가 내려가는데, 다른 fetcher 는 로그 후 재시도를 계속합니다. 종료가 의도라면 PR 본문에 명시하고, 아니라면 상한 간격으로 재시도를 이어 가는 방향이 좋겠습니다.

### 기타 수정사항
- `binance_ws.py:12` — `from typing import Optional` 미사용 — 제거
- `binance_ws.py:140` — 로그 메시지 오타 `reconect` — `reconnect`

### 참고
- 다른 fetcher 의 고정 간격 재시도 — 같은 rate limit 위험, 이 PR 범위 밖 — 신규 Issue 권장입니다.
```

## 관련

- 전역 CLAUDE.md (`~/.claude/CLAUDE.md`) — Simplicity First / Surgical Changes / Goal-Driven Execution / Serena MCP 우선
- 프로젝트 CLAUDE.md — 빌드·테스트 명령, worktree 규칙, 프로젝트 코딩 컨벤션
- 프로젝트 린터/포매터 설정 파일
