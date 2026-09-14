---
name: ticket-review
description: 작성/수정한 코드를 (1) 기능 요구사항 위반, (2) 문법·정적 오류, (3) 코드 품질 개선 관점으로 검토하고, 명확한 사항은 즉시 수정·사용자 판단이 필요한 사항은 검토 후 반영. 구현 완료 후 자체 리뷰, PR 전 점검, "리뷰하고 고쳐줘"·"내가 짠 거 점검해줘" 요청 시 사용. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Review & Fix

코드 변경사항을 **3축 (기능 / 문법 / 품질)** 으로 검토하고, 발견 항목을 **명확 (Clear)** 과 **검토 필요 (Needs review)** 로 나눠서 처리합니다.

- Clear → 즉시 수정
- Needs review → 사용자 결정 받은 뒤 반영

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` 로드 → minimal 4 키 검증. 상세: `~/.claude/skills/ticket-plan/CONFIG.md`.

`${project_root}` 는 현재 worktree 가 아니라 **메인 worktree 루트**다 — 이 스킬은 `{N}-{slug}` worktree 안에서 호출되는 일이 잦은데 `.claude/` 는 대개 gitignore 라 거기에 없다.

이 스킬은 코드만 검토하므로 `gh`/git 원격 작업이 없어 계정 라우팅이 불필요하지만, 동일 family 의 규약 준수를 위해 Bootstrap 을 거칩니다. (멀티 계정 키 `project.gh_account` / `ssh_host` / `git_email` 은 본 스킬에서 무시) 빌드/테스트/린트 **명령은 프로젝트 CLAUDE.md** 의 "Common Commands" / "Testing" / "Linting" 섹션을 참조.

## 적용 시점

- 구현 완료 직후 자체 리뷰
- PR 생성 전 최종 검토
- "리뷰하고 고쳐줘", "내 코드 점검해줘", "개선할 거 있으면 고쳐줘" 같은 요청

## 입력

1. **인자가 있는 경우**: 지정된 파일/디렉터리/PR 을 검토 대상으로 사용
2. **인자가 없는 경우**: 현재 working directory 의 미커밋 변경사항 (`git status`, `git diff`, untracked) 검토
3. 변경사항이 비어 있으면 사용자에게 "어디를 검토할지" 묻기

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

## 분류 기준

### Clear — 즉시 수정
- **단 하나의 객관적으로 옳은 수정**이 존재
- 사실 기반 문제: 타입 에러, 오타, dead import, 명백한 버그
- 컨벤션 문서에 **명문화된** 규칙 위반 (예: 린터 룰)
- 판단/취향이 끼어들 여지가 없는 케이스

### Needs review — 사용자 확인 후 적용
- 여러 유효한 대안 존재 (네이밍, 시그니처, 구조 선택)
- 의미가 바뀌는 변경 (semantic change), 동작 차이 가능성
- 리팩터링 / 추상화 도입 여부
- 컨벤션이 명문화되지 않은 영역
- 트레이드오프가 있는 결정, 성능 vs 가독성 류

> **헷갈리면 무조건 "Needs review"**. 무단 수정의 비용이 사용자 시간을 약간 빼앗는 비용보다 큽니다.

## 실행 단계

### Step 1: 검토 범위 확정
- 인자 / git 상태 / 사용자 질의로 대상 파일·hunk 결정
- 변경된 라인 위주로 컨텍스트 파악 (Read, Grep, 심볼 단위는 Serena)
- 사용자가 직접 수정한 부분과 그 **영향 범위**까지만 검토. 무관 영역으로 확장 금지.
- **보드 동기화**: 현재 브랜치(`{N}-{slug}`)·worktree·인자에서 Issue 번호 N 을 해석할 수 있고 config 에 `board` 가 있으면 `Set-BoardStatus({N}, in_review)` (CONFIG.md "프로젝트 보드 Status 동기화") 실행. N 미상 또는 보드 미설정이면 skip.

### Step 2: 3축 검토
- 각 파일·hunk 마다 기능/문법/품질 3축으로 점검
- 발견 항목을 raw list 로 수집 (위치, 문제, 근거 메모)

### Step 3: 분류
- 항목별로 **Clear** vs **Needs review** 판정
- 분류 근거 1줄 첨부
- 헷갈리는 건 Needs review

### Step 4: Clear 즉시 수정
1. Edit 도구로 모든 Clear 항목 수정
2. 가능하면 가벼운 검증 실행 — 명령은 프로젝트 CLAUDE.md 참조:
   - 정적 검사 (린터)
   - 영향 범위 테스트 (변경된 영역의 단위/통합 테스트)
3. 수정 내역을 짧게 요약

### Step 5: Needs review 항목 제시
- 표 형태로 한 화면에 정리:

  ```
  ### Needs review (N건)

  | # | 위치 | 문제 | 제안 | 근거 |
  |---|------|------|------|------|
  | 1 | src/foo.py:55 | ... | ... | ... |
  ```

- 각 항목마다 **수정 후 코드 스니펫**도 함께 표시 → 사용자가 그 자리에서 판단 가능
- 항목이 많으면 영향 큰 것부터 정렬
- 응답 형식 안내: `1 accept`, `2 skip`, `3 modify: ...`, 일괄 응답 ("1,3 적용") 지원

### Step 6: 사용자 응답 처리
- accept → Edit 으로 적용
- modify → 사용자 지시 반영하여 적용
- skip → 패스 (보고서에는 남김)
- 응답이 모호하면 한 번 더 묻기

### Step 7: 최종 보고
다음을 짧게 요약:
- 적용된 모든 수정 (Clear + 사용자 승인)
- 보류된 항목
- 후속 조치 권고 (테스트 실행, 추가 검토 영역, 별도 티켓 분리 등)

## 가드레일

- **변경 범위 최소화**: 사용자가 만진 부분 + 직접 영향 범위만 봅니다. 인접 코드 "개선" 은 무조건 Needs review 로만 제시 (전역 CLAUDE.md "Surgical Changes")
- **사실과 의견 분리**: Clear 는 사실 기반만. 의견·취향은 무조건 Needs review.
- **의도가 불분명하면 묻기**: 사용자 코드의 의도를 추측해서 고치지 않음
- **신규 파일/추상화 금지**: 검토 범위 밖의 새 파일·모듈·도우미는 만들지 않음
- **테스트 자동 작성 금지**: 테스트가 없다는 지적은 OK, 자동 작성은 별도 요청 시에만
- **백업/임시 파일 만들지 않기**: Edit 으로 직접 수정. 원복은 git 이 합니다.
- **하나의 확인 사이클**: Step 5 의 응답을 받고 적용한 뒤에는 다시 새 항목을 발굴하지 않음. 추가 리뷰가 필요하면 사용자가 다시 호출.

## 출력 예시

```
## Review Summary

대상: src/data_collection/fetchers/binance_ws.py (3 hunks), tests/unit/test_binance_ws.py (1 hunk)

### 즉시 수정 (3건)
1. binance_ws.py:42 — 누락된 import `import asyncio` → 추가
2. binance_ws.py:88 — dead import `from typing import Optional` (미사용) → 제거
3. test_binance_ws.py:15 — 오타 `assert_called_onec` → `assert_called_once`

### Needs review (2건)

| # | 위치 | 문제 | 제안 | 근거 |
|---|------|------|------|------|
| 1 | binance_ws.py:55 | `_reconnect` 함수 60 줄, 단일 책임 분리 가능 | `_compute_backoff()` 추출 | 가독성 ↑, 호출부 1곳뿐이라 inline 유지도 OK |
| 2 | binance_ws.py:120 | 연결 실패 처리 시 raise vs return 선택 | 호출부 컨벤션 확인 필요 | 다른 fetcher 는 raise 사용 — 일관성 측면 |

응답해주세요: `1 accept`, `2 skip`, `1 modify: ...`
```

## 관련

- 전역 CLAUDE.md (`~/.claude/CLAUDE.md`) — Simplicity First / Surgical Changes / Goal-Driven Execution / Serena MCP 우선
- 프로젝트 CLAUDE.md — 빌드·테스트 명령, worktree 규칙, 프로젝트 코딩 컨벤션
- 프로젝트 린터/포매터 설정 파일
