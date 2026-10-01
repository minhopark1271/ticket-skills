---
name: ticket-implement
description: GitHub Issue 를 end-to-end 로 구현합니다. issue comment 와 라벨로 진행 상황을 추적. 작업 시작, 진행 기록, 블로커 기록 시 사용. 인자로 Issue 번호 전달 시 (예 `ticket-implement 115`) 별도 확인 없이 전체 파이프라인 자동 실행 — Issue 파악 → 코드 분석 → 계획 재정비 → 구현 → 테스트 → 자체 리뷰. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Implement

GitHub Issue 의 진행 상황을 업데이트합니다. 작업 시작, 진행 중 기록, 완료 처리에 사용합니다. 프로젝트별 설정은 `{project_root}/.claude/ticket-skill-configs.yaml` 에서 읽습니다.

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` (+ 로컬 `ticket-skill-configs.local.yaml` override, 있으면) 로드·병합 → minimal 4 키 검증 → **계정 라우팅** → `${var}` 치환. 상세 절차와 변수 매핑: `~/.claude/skills/ticket-plan/CONFIG.md`.

`${project_root}` 는 현재 worktree 가 아니라 **메인 worktree 루트**다 — Type B(진행 기록)는 worktree 안에서 호출되는데 `.claude/` 는 대개 gitignore 라 거기에 없다.

핵심 치환:
- `${project.repo}`, `${project.default_branch}`, `${worktree.prefix}`, `${wiki.clone_path}` ← config
- `${branch}`, `${base}` ← Issue body `## Related` 의 브랜치 줄, 없으면 `{N}-{slug}` · `${project.default_branch}` (CONFIG.md "브랜치 해석")
- `${wiki.repo_url}`, `${wiki.url_prefix}` ← universal default (`${wiki.repo_url}` 은 `git@${project.ssh_host}:...`, ssh_host 기본 `github.com`)

**계정 라우팅** (멀티 계정 — 옵셔널): `project.gh_account` 가 있으면 `gh`/git 작업 전에 `gh auth switch --user ${project.gh_account}` 실행. `project.git_email` 이 있으면 worktree 진입 후 `git config user.email ${project.git_email}` (로컬). 없으면 현재 계정/전역 설정 사용.

## 데이터 소스

- **티켓**: GitHub Issues (`${project.repo}`). 티켓 번호 = Issue 번호.
- **진행 로그**: Issue comments (`gh issue comment`)
- **상태**: Issue 라벨 + open/closed state
- **Wiki (분석 문서)**: `ticket-doc` 스킬이 담당

## 자동 파이프라인 (인자로 Issue 번호 전달 시)

`ticket-implement {issue_number}` 형태로 호출되면 (예: `/ticket-implement 115`),
별도의 안내·확인 없이 아래 파이프라인을 **순차적으로 끝까지** 실행합니다.
이는 사용자가 다음과 같은 프롬프트를 직접 입력한 것과 동일하게 동작합니다:

> "#{issue_number} Issue 를 파악하고, 관련 코드를 분석한 뒤, 구현 계획을 재정비하고, 구현하고, 테스트하고, 자체 리뷰까지 진행해 줘."

### Phase 1: Issue 파악
```bash
gh issue view {N} --repo "${project.repo}" --json number,title,body,labels,state,url,comments
```
1. body 의 현상 / 범위 / 현재 구조 / 문제 / 변경 / 기대 + Implementation Plan (옵셔널) 섹션 확인. `변경` 에 wiki 링크가 있으면 설계 상세는 그 페이지에 있음
2. 라벨에서 상태/우선순위/카테고리/컴포넌트 파악
3. body 의 `## Related` 에 `#{N}` 으로 참조된 다른 Issue 가 있으면 `gh issue view` 로 간단히 확인
4. `## Related` 의 브랜치 줄로 `${branch}`·`${base}` 확정 (CONFIG.md "브랜치 해석")
5. Issue 요약을 사용자에게 1~2 문단으로 브리핑 (목표, 범위, 핵심 제약)

### Phase 2: 코드 파악
1. Issue 가 다루는 영역의 관련 파일/심볼 식별 (Glob/Grep, 심볼 단위 작업 시 Serena 활용 — 전역 CLAUDE.md "Serena MCP 우선" 규칙 참조)
2. 핵심 심볼의 시그니처와 호출 관계 (`find_referencing_symbols`)
3. 기존 코드의 패턴, 컨벤션, 유사 구현 확인
4. 파악 내용을 사용자에게 짧게 요약 (수정 대상 파일/심볼 목록 + 영향 범위)

### Phase 3: 구현 계획 재정비
1. Issue 의 변경 / 기대 / Implementation Plan (+ 링크된 설계 wiki) 을 Phase 2 의 실제 코드 상태와 비교
2. 차이가 있으면 구현 단계를 재구성 (어떤 순서로 어떤 파일을 수정할지)
3. 발견된 이슈/제약/추가 작업 항목 정리
4. 재정비된 계획을 사용자에게 짧은 단계 목록으로 제시 (3~7 단계 권장)
5. **사용자 확인을 기다리지 않고** 다음 단계로 진행 (단, 명백한 위험·범위 변경이 있으면 그때만 확인)

### Phase 4: 작업 시작 처리
아래 **Type A: 작업 시작** 절차 실행 (라벨 `in-progress` 추가, worktree 생성/확인, 진행 로그 시작 comment).

### Phase 5: 구현
1. 재정비된 계획대로 코드 작성/수정 (심볼 단위 편집은 Serena 우선)
2. 의미 있는 진행 단위마다 진행 로그 comment 추가 (Type B 절차 참조)

`pr.merge_method: merge` 면 **Implementation Plan Phase 하나씩** 구현하고 Phase 5→6→6.5→6.7 을 그 Phase 마다 돈다. 그 외에는 전체를 구현하고 6.7 을 건너뛴다.

### Phase 6: 빌드 & 테스트
1. 변경 코드 영역의 테스트 실행 (명령은 프로젝트 CLAUDE.md 의 "Common Commands" / "Testing" 섹션 참조)
2. 신규 기능 / 버그 수정 시 검증 테스트 추가 후 실행
3. 실패 시 **Phase 5 로 돌아가 수정**. 통과까지 Phase 5↔6 루프
4. 통과 시 곧바로 Phase 6.5 (사용자 확인 불필요)

### Phase 6.5: 자체 리뷰

`ticket-review` 스킬을 인자 없이 호출 → 현재 worktree 의 미커밋 변경사항을 3축 (기능 / 문법 / 품질) 으로 검토. ticket-review 는 의견만 출력하므로 반영은 이 Phase 가 한다.

1. **결함 · 기타 수정사항** 은 즉시 수정
2. 수정으로 코드가 바뀌었으면 Phase 6 의 테스트 재실행
3. **검토 필요 사항** 이 **없으면** 멈추지 않고 다음 Phase (6.7 또는 7) 로 진행. **있으면** 그대로 제시하고 사용자 응답을 받아 반영 — 자동 파이프라인에서 유일하게 허용되는 대기 지점
4. **참고** 는 Phase 7 보고의 후속 작업 후보로 넘긴다

중점 축: 주석 정책 (전역 CLAUDE.md 5절), 변수·함수·클래스 네이밍, 프로젝트 코딩 컨벤션·포맷.

### Phase 6.7: Phase 단위 커밋 (`pr.merge_method: merge` 일 때만)

"티켓 1개 = 커밋 1개" 운용에서는 커밋 분할이 구현의 일부다 — ticket-complete 가 나중에 뭉쳐서 커밋하면 bisect·부분 revert 가 죽는다.

1. 방금 끝낸 Implementation Plan Phase 의 변경만 스테이징해 커밋 1개. 메시지 형식은 `ticket-complete` Step 2, 제목은 그 Phase 제목
2. 빌드가 깨진 상태로 커밋하지 않는다 — Phase 6 통과가 전제
3. 남은 Phase 가 있으면 Phase 5 로 돌아간다. 없으면 Phase 7

`merge_method` 가 `merge` 가 아니면 이 단계 전체를 건너뛴다 (커밋은 ticket-complete Step 2 가 한다).

### Phase 7: 작업 보고 (자동 완료 처리 금지)

**중요**: 자동 파이프라인은 빌드/테스트 통과 후 **반드시 멈춥니다**.
완료 처리 (머지/정리) 는 사용자가 명시적으로 요청한 경우에만 진행.

1. 사용자에게 보고:
   - 구현 완료된 항목 (Implementation Plan 이 있으면 Phase 단위) + `## 기대` 항목별 충족 여부
   - 변경된 파일 목록 + (Phase 단위 커밋 운용이면) 커밋 목록 `git log --oneline ${base}..HEAD`
   - 빌드/테스트 결과 요약
   - 자체 리뷰 결과 (즉시 수정 건수, 보류 항목)
   - 후속 작업 후보 (현재 Issue 범위 밖에서 발견한 명백한 이슈가 있을 때만 짧게 1-3 줄)
   - 다음 단계 안내: "완료 처리는 `/ticket-complete {issue-number}` 또는 '완료해줘'·'마무리해줘' 로 명시 요청해 주세요."
2. **절대 자동으로 ticket-complete 를 실행하지 않습니다.**
3. 사용자가 명시적으로 완료 요청 시 → **`ticket-complete` 스킬 호출**
4. 사용자가 추가 작업/수정 요청 시 그에 따라 진행

## 사용 절차

### Step 1: Issue 찾기
```bash
gh issue view {N} --repo "${project.repo}" --json number,title,body,labels,state,comments,url
```
state 가 `CLOSED` 면 사용자에게 확인 (이미 완료된 Issue 일 수 있음).

### Step 2: 업데이트 유형 결정

#### Type A: 작업 시작 (planned → in-progress)
1. **라벨 변경**:
   ```bash
   gh issue edit {N} --repo "${project.repo}" --add-label "in-progress"
   ```
   라벨이 없으면 지연 초기화 (ticket-plan 의 "지연 초기화 — Universal 라벨" 섹션 참조) 로 생성 후 재시도.
2. **Worktree 확인/생성** (ticket-plan 에서 이미 생성된 경우 스킵):
   ```bash
   git branch --track ${base} origin/${base}   # 로컬에 없을 때만
   git worktree add ../${worktree.prefix}-{N}-{slug} -b ${branch} ${base}
   ```
   slug 는 Issue 제목에서 kebab-case 추출 — **디렉토리명에만** 쓴다. 브랜치명은 `${branch}` (CONFIG.md "브랜치 해석").
3. **작업 시작 comment 추가**:
   ```bash
   gh issue comment {N} --repo "${project.repo}" --body-file /tmp/progress.md
   ```
   `/tmp/progress.md` 내용은 아래 **진행 Comment Template** 의 "작업 시작" 형식.
4. **보드 동기화**: config 에 `board` 가 있으면 `Set-BoardStatus({N}, in_progress)` (CONFIG.md "프로젝트 보드 Status 동기화") 실행. 보드 미설정이면 skip.

#### Type B: 진행 기록 (ongoing work)
1. 관련 PR 링크 확인: `gh pr list --repo "${project.repo}" --search "head:${branch}" --state all`
2. **진행 comment 추가**:
   ```bash
   gh issue comment {N} --repo "${project.repo}" --body-file /tmp/progress.md
   ```
3. comment 본문에 기록:
   - 완료된 작업
   - 변경된 파일
   - 발견된 이슈/블로커
   - 다음 단계
   - PR 링크 (있으면)
   - 후속 작업 (현재 Issue 범위 밖에서 발견한 항목 — 있을 때만)
4. **문서화 (선택)**: 진행 중 발견한 중요한 기술적 인사이트, 실험 결과, 또는 분석 데이터가 comment 에 담기엔 상세하면 `ticket-doc` Type B 자동 호출 → wiki 페이지 **드래프트** 생성(로컬 커밋만; **push 안 함** — 발행은 `ticket-complete`) 후 comment 에서 wiki URL 참조

#### Type C: 완료 (in-progress → completed)

**완료 처리는 `ticket-complete` 스킬이 담당합니다.**

- 코드 변경 + 머지 / PR / worktree 정리: 사용자가 `ticket-complete` 호출 → 내부에서 Issue close + 머지 / 정리 일괄 실행
- 문서 전용 변경 (코드 변경 없음, 머지 불필요): 사용자가 `ticket-complete` 호출 → PR / 머지 단계 스킵 옵션 사용
- Issue 완료 절차 (체크리스트 검증 / 라벨 정리 / Implementation Results comment / close) 의 구체 단계는 `ticket-complete` Step 9 참조

직접 머지 (`git merge`) 또는 직접 `gh issue close` 는 협업 환경에서 PR 리뷰 단계를 건너뛰므로 **사용 금지**입니다.

#### Type D: 리뷰 필요 (in-progress → needreview)
```bash
gh issue edit {N} --repo "${project.repo}" --remove-label "in-progress" --add-label "needreview"
gh issue comment {N} --repo "${project.repo}" --body "..."
```
comment 본문에 구현 요약 (커밋 목록, 변경 파일, 테스트 결과, PR 링크) 포함.

#### Type E: 보류 처리 (planned → backlog)
```bash
gh issue edit {N} --repo "${project.repo}" --add-label "backlog"
```
당장 진행하지 않는 Issue 를 보류 상태로 전환.

## 진행 Comment Template

각 진행 comment 는 헤더로 날짜와 상태를 표시:

```markdown
### Progress: {YYYY-MM-DD} · {작업 시작 / 진행 중 / 블로커 / 완료}

**완료된 작업**:
- {작업 내용}

**변경된 파일**:
- `path/to/file.py` — {변경 내용}

**이슈/블로커**:
- {발견된 문제 또는 "없음"}

**PR**:
- #{PR 번호} 또는 "없음"

**Follow-up**:
- {현재 Issue 범위 밖 후속 항목. 없으면 이 항목 자체 생략}

**다음 단계**:
- {남은 작업}
```

## 구현 결과 Comment Template (완료 시)

`ticket-complete` 가 close 직전에 추가하는 마지막 comment:

```markdown
## Implementation Results

### PR
- #{PR 번호} ({PR URL})

### Changed Files
| File | Changes |
|---|---|
| `path/to/file.py` | {간단한 변경 설명} |

### Test Results
{테스트 실행 결과, 커버리지 등. 변경 없으면 "No tests modified"}

### Notes / Issues
{구현 중 알아낸 주요 사항. 없으면 "없음"}

### Follow-up
- {후속 작업 후보. 별도 Issue 생성은 ticket-complete 에서 사용자 확인. 없으면 섹션 생략}

### Completed
{YYYY-MM-DD}
```

## 예시

### 예시: 작업 시작

```bash
gh issue edit 115 --repo "${project.repo}" --add-label "in-progress"
git worktree add ../${worktree.prefix}-115-add-websocket -b 115-add-websocket ${project.default_branch}   # 브랜치 줄 없는 경우
gh issue comment 115 --repo "${project.repo}" --body-file /tmp/progress.md
```

`/tmp/progress.md`:
```markdown
### Progress: 2026-06-12 · 작업 시작

**완료된 작업**:
- 요구사항 분석 완료
- 외부 API 문서 검토

**변경된 파일**:
- 없음 (분석 단계)

**이슈/블로커**:
- 없음

**다음 단계**:
- WebSocket 클라이언트 클래스 구현
- API 엔드포인트 추가
```

## PR 컨벤션

PR 생성 형식은 `ticket-complete` Step 4 참조. 핵심:
- PR 제목: `{N}-{Short description}`
- PR base: `${base}`
- PR 본문: `Closes #{N}` 포함 → base 가 repo 기본 브랜치일 때만 머지 시 Issue 자동 close (아니면 ticket-complete Step 9-6 이 명시 close)

## 후속 작업 처리 원칙

Issue 범위 밖에서 발견한 사항은 **현재 Issue 를 오염시키지 않고** 처리합니다.

**원칙**:
- 발견 즉시 다른 Issue 를 자동 수정하지 않는다 (정보량 폭증 방지)
- 발견사항은 **현재 Issue 의 진행 comment / 구현 결과의 Follow-up** 에만 기록
- ticket-complete 단계에서 사용자에게 후속 작업 목록을 보여주고, 별도 Issue 생성 여부를 **사용자가 결정**
- 명백한 안전 이슈/버그가 즉시 처리되어야 하면 사용자에게 즉시 알리고 별도 Issue 생성 여부를 묻는다

이외 자동 전파·자동 라벨 추가·완료 시 전 Issue 스캔은 **수행하지 않습니다**.

## 권장 사항

- 작업 시작 시 반드시 `in-progress` 라벨 추가
- 의미 있는 진행이 있을 때마다 진행 comment 추가 (커밋 단위와 비슷한 빈도)
- 블로커 발생 시 즉시 comment 로 기록하여 컨텍스트 보존
- 완료 시 구현 결과 comment 로 전체 변경 사항 요약 (ticket-complete 가 추가)
- 원본 Issue body 는 그대로 유지 (수정 시 `gh issue edit`)
- 후속 작업은 **현재 Issue 에 comment 로 기록만** — 다른 Issue 자동 수정 금지
