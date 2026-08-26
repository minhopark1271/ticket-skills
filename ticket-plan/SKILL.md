---
name: ticket-plan
description: GitHub Issues 로 Claude Code 가 실행할 개발 티켓을 작성합니다. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다. 사용 트리거 — "티켓 만들어줘", "Issue 생성", "ticket-plan", "기획", "계획 세우자".
---

# Ticket Plan

GitHub Issue 로 Claude Code 가 실행할 개발 티켓을 구조적으로 작성합니다. 프로젝트별 설정 (`repo`, `default_branch`, `worktree.prefix`, `wiki.clone_path`) 은 `{project_root}/.claude/ticket-skill-configs.yaml` 에서 읽습니다.

## Bootstrap (모든 호출 시 첫 단계)

이 스킬은 `{project_root}/.claude/ticket-skill-configs.yaml` 의 설정을 읽어 동작합니다. 첫 호출 시 다음 절차를 수행:

1. **프로젝트 루트 감지**: `git rev-parse --show-toplevel` (실패 시 cwd)
2. **Config 로드 + 로컬 override 병합**: 공유 `${project_root}/.claude/ticket-skill-configs.yaml` (없으면 minimal 4 키 (`project.repo`, `project.default_branch`, `worktree.prefix`, `wiki.clone_path`) 작성 안내 후 중단) → 로컬 `ticket-skill-configs.local.yaml` 있으면 deep merge (로컬 우선; 개인 gh 계정·SSH·이메일 키). 없으면 skip
3. **브랜치 override merge** (옵셔널): `git branch --show-current` 결과가 `branches.{name}` 에 있으면 해당 키만 덮어쓰기
4. **필수 키 검증**: 4 키 모두 존재 확인. 누락 시 보고 후 중단
5. **계정 라우팅** (멀티 계정 — 옵셔널): `project.gh_account` 가 있으면 `gh` 명령 전에 `gh auth switch --user ${project.gh_account}` 실행 (실패 시 `gh auth login` 안내 후 중단). 없으면 현재 active 계정 사용
6. **치환 매핑 준비** — 본문의 `${var}` 는 다음으로 치환:
   - `${project.repo}` ← config
   - `${project.default_branch}` ← config
   - `${worktree.prefix}` ← config
   - `${wiki.clone_path}` ← config
   - `${project.ssh_host}` ← config (없으면 `github.com`)
   - `${wiki.repo_url}` ← `git@${project.ssh_host}:${project.repo}.wiki.git` (universal default)
   - `${wiki.url_prefix}` ← `https://github.com/${project.repo}/wiki/` (universal default)
7. **자가 검증**: 본문에 매핑 불가 `${...}` 가 있으면 보고 후 중단

상세 스키마: `~/.claude/skills/ticket-plan/CONFIG.md`

## 데이터 소스

- **티켓**: GitHub Issues (`gh issue ...`). 티켓 번호 = Issue 번호
- **레포**: `${project.repo}`, 기본 브랜치 `${project.default_branch}`
- **Wiki (분석 문서)**: `ticket-doc` 스킬이 GitHub Wiki 에서 관리. 본 스킬은 Issue body 에서 wiki 페이지 URL 로만 참조

## 지연 초기화 — Universal 라벨 (첫 호출 시)

레포에 필요한 라벨이 없으면 첫 호출 시 inline 으로 생성. 이미 존재하면 `--force` 로 no-op.

```bash
# 상태 라벨 (open 상태에서 추가 표시. completed = closed)
gh label create "in-progress" --color "0E8A16" --description "구현 진행 중" --force
gh label create "needreview" --color "FBCA04" --description "리뷰 대기" --force
gh label create "backlog" --color "BFD4F2" --description "당장 진행하지 않음" --force

# 우선순위 라벨 (medium 은 라벨 없음)
gh label create "priority:high" --color "D93F0B" --description "중요, 빠르게" --force
gh label create "priority:urgent" --color "B60205" --description "긴급, 즉시" --force
gh label create "priority:low" --color "CCCCCC" --description "여유 시 진행" --force

# 카테고리 라벨
gh label create "feature" --color "A2EEEF" --description "새 기능" --force
gh label create "bugfix" --color "D73A4A" --description "버그 수정" --force
gh label create "refactor" --color "C5DEF5" --description "리팩터링" --force
gh label create "performance" --color "FBCA04" --description "성능 개선" --force
gh label create "docs" --color "0075CA" --description "문서" --force
gh label create "infra" --color "BFD4F2" --description "인프라 / DevOps" --force
```

**프로젝트 고유 라벨 (컴포넌트 등)**: schema 외부. 프로젝트마다 다르므로, 필요해질 때 사용자가 직접 추가하거나 schema 에 `labels.components` 항목을 한 번 확장해서 자동화. (`## 라벨` 표의 "프로젝트 고유 라벨" 행 참조)

## 사용 절차

### 모드 1: 사용자 요청에서 생성 (계획 포함)

#### Step 1: 요청 분석
1. 개발 요구사항 파악
2. Claude Code 가 실행 가능한 단위로 분해
3. 관련 파일 및 의존성 식별

#### Step 2: 코드베이스 탐색 및 구현 계획

1. **관련 코드 탐색**:
   - Glob/Grep 으로 관련 파일 찾기
   - 주요 파일을 Read 로 읽어 현재 구현 파악
   - 사용 중인 패턴 및 컨벤션 식별
   - 심볼 단위 작업이 필요하면 Serena (`find_symbol`, `get_symbols_overview`, `find_referencing_symbols`) 활용

2. **구현 접근 설계**:
   - 변경 순서 결정
   - 변경 간 의존성 식별
   - 엣지 케이스 및 잠재 이슈 고려

3. **Implementation Plan 초안 작성**:
   - phase/step 단위로 분해
   - 각 phase 마다: 변경 파일, 구체 수정 내용, 고려 사항

**탐색 중 범위 밖 발견사항**: 현재 티켓 범위와 무관한 버그·미비점을 발견했다면 **현재 티켓 본문에 흡수하지 말고**, 리뷰 진입 전 사용자에게 한 줄로만 보고합니다. 자동 전파·자동 Issue 생성은 금지 — 사용자가 별도 티켓을 원하면 그때 새 Issue 를 생성합니다.

#### Step 2.5: 설계 평가 (조건부 필수)

Step 2 의 분석 결과, 아래 트리거 중 **하나 이상 해당**하면 Issue body 에 **`## Design` 섹션을 필수로 작성**합니다.

| 트리거 | 예 |
|---|---|
| 새 자료 구조 정의 | dataclass / DB 스키마 / 상태 머신 / 캐시 키 구조 / 메시지 포맷 |
| 논리적 구조 설계 | 워크플로우 / 파이프라인 / 데이터 흐름 / 이벤트 순서 |
| 함수·클래스 hierarchy 설계 | 책임 분리 / 호출 관계 / 인터페이스·상속 / 모듈 경계 |
| 비자명한 파라미터 설계 | API 시그니처 / 인자 기본값 정책 / config schema |

해당 트리거가 전혀 없는 단순 변경 (버그 픽스, config 한 줄 변경, 문구 수정 등) 은 이 단계 스킵.

**내부 도출 절차** (Issue 작성 전 반드시 수행):

1. **현황 분석** — 기존 코드 구조 / 자료 구조 / 호출 관계를 Glob·Grep·Read (심볼 단위는 Serena `get_symbols_overview`, `find_referencing_symbols`) 로 파악. 무엇이 이미 있고, 어디까지 재사용 가능한지.
2. **대안 도출** — **최소 2개 이상** 설계 후보를 작성. 네이티브/공식 API 방식을 항상 한 후보로 포함.
3. **대안 비교** — 각 후보의 장점·단점·복잡도·확장성·기존 코드와의 정합성을 표로 평가.
4. **최적 선택** — 어떤 후보를 채택할지 + **선택 근거** 명시. 후순위 대안은 기각 사유 1줄.
5. **`## Design` 작성** — 아래 Template 의 sub-section 형식 (현황 분석 / 대안 비교 / 선택 + 근거 / 자료 구조 / 클래스 hierarchy / 핵심 시그니처) 으로 Issue body 에 포함.

**설계가 길어 Issue body 만으로 부족할 때**: 문서화 단계에서 ticket-doc Type A 자동 호출 → wiki 페이지에 상세 작성. 단, Issue body 의 `## Design` 에는 항상 **요약 + 선택 근거 + 핵심 시그니처** 는 남기고, 풀 상세만 wiki 로 위임.

#### Step 3: 티켓 분할 평가

분석/계획이 완료되면 **단일 Issue vs 복수 Issue** 를 판단합니다.

**분할 기준 — 아래 중 하나 이상 해당하면 분할 검토:**

| 기준 | 예시 |
|------|------|
| **덩어리가 큰 경우** | Phase 4개 이상이거나, 예상 변경 파일 10개 이상 |
| **복잡도가 섞인 경우** | 간단한 config 변경 + 복잡한 알고리즘 구현이 하나에 섞임 |
| **종류가 명확히 나뉘는 경우** | DB 스키마 / API 엔드포인트 / 대시보드 UI 등 별도 레이어 |
| **독립 배포 가능** | 일부만 먼저 머지해도 시스템이 정상 동작 |
| **의존 관계가 순차적** | A를 먼저 완료해야 B를 시작할 수 있는 구조 |

**분할 불필요 기준:**
- Phase 3개 이하이고 변경 범위가 좁은 경우
- 모든 변경이 하나의 기능/목적에 긴밀히 결합된 경우
- 분할하면 오히려 맥락 파악이 어려워지는 경우

**분할 판단 결과 처리:**

1. **분할 불필요** → Step 4 로 진행 (단일 Issue 생성)
2. **분할 필요** → 사용자에게 분할안 제시 후 확인:

```markdown
이 작업은 다음과 같이 Issue 를 나누는 것을 제안합니다:

| # | 제목 | 유형 | 의존성 |
|---|---|---|---|
| 1 | {제목} | {feature/refactor/...} | - |
| 2 | {제목} | {feature/bugfix/...} | #1 완료 후 |
| 3 | {제목} | {feature/...} | #1 완료 후 (병렬 가능) |

분할안대로 진행할까요?
```

- 사용자 승인 → Step 4-Multi
- 사용자 수정 제안 → 분할안 조정 후 재확인
- 사용자가 단일 선호 → Step 4

#### Step 4: GitHub Issue 생성 (단일)

```bash
gh issue create \
  --repo "${project.repo}" \
  --title "{Short description}" \
  --label "feature,priority:high" \
  --body-file /tmp/issue-body.md
```

본문 (`/tmp/issue-body.md`) 은 아래 **Template** 형식으로 작성. `gh issue create` 의 출력에서 Issue URL/번호를 받아 사용자에게 안내.

**중요**:
- 라벨은 카테고리 (필수 1개) + 우선순위 (medium 이면 생략) + 프로젝트 고유 라벨 (있는 경우) 조합
- `--body-file` 사용 권장 (긴 본문/특수문자 안전)
- `--repo` 명시 권장 (worktree 등 cwd 가 main repo 가 아닐 때 안전)
- body 최하단에 `## History` + `created` 한 줄 포함 (STYLE.md "History · 세션 서명")

**보드 동기화**: config 에 `board` 가 있으면 Issue 생성 직후 `Set-BoardStatus({N}, backlog)` (CONFIG.md "프로젝트 보드 Status 동기화") 실행. 복수 생성(Step 4-Multi)은 각 Issue 마다. 보드 미설정이면 skip.

#### Step 4-Multi: 복수 Issue 생성

1. **의존성 순서대로** 순차 생성. 앞 Issue 번호를 다음 Issue 의 `## Related` 에 `#{앞번호}` 로 참조
2. **모든 분할 Issue 의 `## Original Request`** 에 동일한 원본 프롬프트 인용
3. **작업 분배**: 전체 작업을 각 Issue 의 Implementation Plan Phase 로 배분 (중복 금지)
4. 생성 후 각 Issue body 를 `gh issue edit {N} --body-file ...` 로 상호 참조 보강 + 각 body `## History` 에 `edited` 한 줄

분할 그룹 추적은 라벨 `split-group:{대표번호}` 를 별도 생성해 묶거나, Issue body 의 `## Related` 에 명시적으로 기록.

### 모드 2: 기존 문서에서 추출

프로젝트 문서 (예: 블로그 포스트, 설계 문서) 에서 작업 항목을 추출하여 Issue 로 생성합니다. 소스 경로는 프로젝트마다 다를 수 있어 schema 외부.

#### Step 1: 소스 문서 읽기
1. 사용자가 지정한 문서 경로 (예: `docs/design.md`, `octomore/public/...`) 를 Read 로 확인
2. TODO / "향후 계획" / "Next Steps" 섹션 식별

#### Step 2: 항목 추출
1. 실행 가능한 항목별로 파싱
2. 판단: 신규 Issue 인지 기존 Issue 업데이트인지, 우선순위, 관련 파일

#### Step 3: 기존 Issue 확인
```bash
gh issue list --repo "${project.repo}" --search "{키워드 in:title,body}" --state all --limit 20
```
유사 Issue 가 있고 open 이면 업데이트 후보로, completed/closed 면 신규 생성 후보로 분류.

#### Step 4: 티켓 분할 평가
모드 1 의 Step 3 기준 적용.

#### Step 5: Issue 생성 또는 업데이트
1. 신규: 모드 1 의 Step 4 (또는 4-Multi)
2. 업데이트: `gh issue edit {N} --body-file ...` (`## History` 에 `edited` 한 줄) 또는 `gh issue comment {N} --body-file ...` (최하단 세션 서명 footer)
3. 모든 신규 Issue body 에 `source: {문서 경로}` 한 줄을 `## Background` 인용 형태로 포함

### 모드 3: 기존 계획에서 생성 (.plan.md)

#### Step 1: 기존 plan 확인
1. 현재 작업 디렉토리에서 `.plan.md` 찾기
2. 없으면 → 사용자에게 plan 파일 경로 요청 또는 모드 1 로 전환

#### Step 2: Plan 읽기 및 파싱
- Goal / Background / Implementation Steps / Files to Modify / Considerations 추출

#### Step 3: 티켓 분할 평가
모드 1 의 Step 3 기준 적용.

#### Step 4: Issue 생성
1. 단일/복수에 따라 모드 1 의 Step 4 또는 4-Multi
2. 매핑:
   - Plan 의 objective → **Goal**
   - Plan 의 context → **Background**
   - Plan 의 considerations → **Technical Details**
   - Plan 의 steps → **Implementation Plan** Phase 들 (옵셔널)
   - Plan 의 files → **Related** (파일 경로)
3. Issue body 에 `source: .plan.md` 한 줄 포함
4. Issue 생성 성공 후 `.plan.md` 삭제/아카이브 (사용자 확인)

### 문서화 단계 (모든 모드 공통)

Issue 생성 후, 리뷰 흐름 진입 전에 자동 실행.

**자동 호출 판단 기준** — 아래 중 하나 이상 해당 시 `ticket-doc` 호출:
- 배경 조사/코드 분석이 Issue body 에 다 담기엔 상세한 경우
- 설계 대안 비교가 여러 개인 경우
- 알고리즘 수식이나 상세 플로우 설명 필요
- 데이터 흐름 / 아키텍처 다이어그램 수준 포함
- Step 2.5 의 `## Design` 상세 (대안 표·자료구조·hierarchy·시그니처) 가 Issue body 1 화면 분량을 초과하는 경우 — Issue body 의 `## Design` 에는 요약 + 선택 근거 + 핵심 시그니처만 남기고, 풀 설계는 wiki 에 위임

**실행 순서**:
1. `ticket-doc` Type A 호출 → GitHub Wiki 페이지 **드래프트** 생성 (로컬 커밋만; **push 안 함**)
2. 페이지명: `{issue-number}-{seq}-{analysis-title}` (예: `115-1-websocket-flow-analysis`)
3. 생성된 wiki 페이지 URL 을 Issue body 의 `## Related` 섹션에 추가 (URL 은 파일명 기반이라 발행 전에도 확정 — 실제 발행은 `ticket-complete` 에서)

**해당 없으면** 스킵. Background/Technical Details 에 충분히 담긴 경우 별도 wiki 페이지 불필요.

---

### 리뷰 흐름 (Issue 생성 후)

복수 Issue 인 경우 각 Issue 를 순차 리뷰. Issue 간 전환 시 `--- 다음 Issue: #{number} ---` 로 구분.

#### Step R1: 리뷰 진입 여부 확인
- "Issue #{N} 초안이 생성되었습니다. 리뷰를 진행할까요?"
- 거부 → Step R2
- 수락 → Step R3

#### Step R2: 구현 시작 여부 확인 (리뷰 스킵 시)
- "리뷰 없이 바로 구현을 시작할까요?"
- 수락 → worktree 생성 안내
- 거부 → 종료

#### Step R3: Issue 전문 브리핑

`gh issue view {N} --repo "${project.repo}" --json title,body,labels,url` 로 가져온 후, `##` 헤더 기준 섹션별 발췌. 본문 5줄 초과 시 핵심 1-3 줄 요약. Implementation Plan 은 Phase 단위 유지. Original Request 는 원문 인용. 옵셔널 섹션 없으면 생략. **History 는 발췌 · 리뷰 대상에서 제외** (기계 기록).

브리핑 후 바로 Step R4 로 진행.

#### Step R4: 항목별 순차 리뷰

각 섹션을 **하나씩 순서대로** 리뷰. 한 번에 하나의 섹션만 보여주고 피드백 받은 후 다음으로.

**리뷰 순서** (필수 4 + 옵셔널 4; 옵셔널은 있는 경우만):
1. Original Request (확인만)
2. Goal
3. Background
4. Technical Details
5. Design — Step 2.5 트리거 시 필수 (sub-section 별로: 현황 분석 / 대안 비교 / 선택 + 근거 / 자료 구조 / hierarchy / 핵심 시그니처)
6. Implementation Plan — 있는 경우만 (Phase 별)
7. Acceptance Criteria — 있는 경우만
8. Related — 있는 경우만

**섹션 리뷰 출력 형식:**

```markdown
**[{섹션명}]**
{현재 내용을 일반 텍스트로 표시. 인용 블록 (>) 사용 금지}

수정사항 있으면 입력:
```

**사용자 응답 처리:**

| 응답 | 처리 |
|---|---|
| `ok` 또는 빈 입력 | 승인. 다음 섹션 이동 |
| 수정 내용 입력 | 변경사항 메모 + 연관 섹션 영향 즉시 판별. "확인. {변경 요약}" 후 다음 섹션 |
| 추가 설명 | "확인. {요약}" 후 다음 섹션 |

연관 매핑은 Step R5 의 동기화 규칙 표 참조.

#### Step R5: 피드백 반영 및 재리뷰

모든 섹션 리뷰 후:

1. **변경사항 표 정리 (직접 피드백)**

```markdown
### 직접 변경사항

| 섹션 | 항목 | 변경 전 | 변경 후 |
|---|---|---|---|
```

2. **연관 섹션 동기화 (Cross-Section Sync)**

| 트리거 | 연동 액션 |
|---|---|
| **Goal** 변경 | → Design / Implementation Plan / Acceptance Criteria 정합성 확인 |
| **Technical Details** 변경 | → Design 의 현황 분석·선택 근거 / Implementation Plan Phase Notes 반영 |
| **Design** 의 자료 구조 / hierarchy / 시그니처 변경 | → Implementation Plan Phase Files / Notes 동기화. Acceptance Criteria 의 검증 대상이 새 시그니처 / 자료구조와 정합한지 확인 |
| **Design** 의 선택 변경 (방안 교체) | → 대안 비교 표 갱신 + 기각 사유 보정 + Implementation Plan 재구성 |
| **Implementation Plan** Phase 추가/삭제/수정 | → Acceptance Criteria 검증대상 / Related 파일 동기화. Design 의 Files 정합성 확인 |
| **Acceptance Criteria** 추가/수정 | → Implementation Plan 검증대상 존재 확인 |
| **파일 경로** 추가/삭제 | → Related 업데이트 |

연관 변경 없으면 "연관 변경 없음" 표기.

3. **Issue body 일괄 갱신**:
```bash
gh issue edit {N} --repo "${project.repo}" --body-file /tmp/issue-body-updated.md
```

수정된 본문은 직접 변경 + 연관 변경을 **모두** 누락 없이 반영. 라벨 변경이 있으면 `--add-label` / `--remove-label` 추가. 기존 `## History` 보존 + `edited` 한 줄.

**보드 동기화**: config 에 `board` 가 있으면 리뷰 반영 완료 시점에 `Set-BoardStatus({N}, ready)` (CONFIG.md) 실행. 보드 미설정이면 skip.

4. **재리뷰 여부 확인**:
- "다시 리뷰할까요?"
- 수락 → R3
- 거부 → 종료

---

### Worktree 생성 (리뷰 완료 후)

리뷰 완료 (또는 리뷰 없이 바로 구현) 시 worktree 생성.

**Worktree 규칙**:
- **Worktree 이름**: `${worktree.prefix}-{issue-number}-{slug}` (예: `asdf-115-add-websocket`)
- **브랜치 이름**: `{issue-number}-{slug}` (예: `115-add-websocket`)
- **slug**: Issue 제목에서 kebab-case 추출 (짧게)
- **Base 브랜치**: `${project.default_branch}`

**실행 절차**:

```bash
git worktree add ../${worktree.prefix}-{issue-number}-{slug} -b {issue-number}-{slug} ${project.default_branch}
```

사용자 안내:
```
Worktree 생성: ../${worktree.prefix}-{issue-number}-{slug}
브랜치: {issue-number}-{slug} (base: ${project.default_branch})
연결 Issue: #{issue-number}

구현은 ticket-implement {issue-number} 로 시작하세요.
완료는 ticket-complete {issue-number} 로 마무리됩니다.
```

worktree 생성을 원하지 않으면 스킵.

---

## Template (Issue body)

Issue body 는 **필수 5 섹션 + 옵셔널 4 섹션** 구조. 옵셔널은 필요할 때만 작성 (빈 섹션 금지). `## Design` 은 Step 2.5 트리거 해당 시 **조건부 필수**.

**필수**: Original Request / Goal / Background / Technical Details / History (최하단 고정)
**옵셔널**: Design (Step 2.5 트리거 시 필수) / Implementation Plan / Acceptance Criteria / Related

**Technical Details 는 `as-is` / `to-be` / `not-to-be` 3 파트**. 현재 상태와 변경 방향이 한 목록에 섞이면 불릿마다 시점이 바뀌어 무엇이 사실이고 무엇이 결정인지 구분되지 않습니다. 내용 없는 파트는 소절째 생략 (신규 코드면 `as-is` 생략). 3 파트에 담기지 않는 내용만 다른 포맷 (표 · 코드 블록) 또는 규칙 없는 나열 — 예: 원인 미확정 조사 티켓, 계약 변경이 아닌 설정 · 의존성 목록.

```markdown
## Original Request
> {사용자가 입력한 원래 프롬프트를 그대로 인용}

## Goal
{달성하고자 하는 결과를 명확히 기술}

## Background
{왜 필요한가? 컨텍스트와 동기}

## Technical Details

### as-is
{건드릴 코드의 현재 계약 · 구조. 결정을 읽는 데 필요한 사실만}

### to-be
{본 티켓이 바꾸는 것. 계약 · 규칙 단위}

### not-to-be
{바꾸지 않는 것 — 범위 경계, 타 Issue 소유, 의도적 미수정, 후속 후보}

<!-- ── 이하 옵셔널: 필요한 경우만 ── -->

## Design  <!-- Step 2.5 트리거 (자료구조/논리구조/hierarchy/파라미터 설계) 시 필수 -->

### 현황 분석
- {기존 코드/자료 구조의 핵심 요약}
- {영향받는 파일/심볼/모듈}

### 대안 비교

| 방안 | 핵심 | 장점 | 단점 |
|---|---|---|---|
| A. {이름} | {1줄 요약} | {1-2 항목} | {1-2 항목} |
| B. {이름} | {1줄 요약} | {1-2 항목} | {1-2 항목} |

### 선택 — 방안 {X}

**선택 근거**: {2-3 줄. 기존 코드와의 정합성/확장성/복잡도 측면}
**기각된 대안**: {타 방안 1줄 기각 사유}

#### 자료 구조
```python
@dataclass
class Example:
    field1: Type   # 의미/단위
    field2: Type | None = None
```

#### 컴포넌트 / 클래스 Hierarchy
```
ModuleA
├── ClassRoot           (책임: ...)
│   ├── ChildA          (책임: ...)
│   └── ChildB          (책임: ...)
└── HelperFunc          (책임: ...)
```

#### 핵심 시그니처 / 파라미터
```python
def core_op(x: T, *, opt: U = DEFAULT, retries: int = 3) -> R:
    """간단 설명. 핵심 인자 정책 (기본값 / None 의미 / 예외)."""
```

## Implementation Plan
- [ ] **Phase 1: {Phase 제목}**
  - Files: `path/to/file.py`
  - Changes: {구체 수정 내용}
  - Notes: {엣지 케이스, 의존성}
- [ ] **Phase 2: {Phase 제목}**
  - Files: `path/to/another.py`
  - Changes: {구체 수정 내용}

## Acceptance Criteria
{Implementation Plan 완료만으로 자명한 경우 생략. 구현 외 검증 기준이 있을 때만 작성}

## Related
- #123 — 의존 / 관련 Issue
- [WebSocket Flow Analysis](${wiki.url_prefix}115-1-websocket-flow-analysis) — 분석 문서 (wiki URL 사용; Issue body 에서는 `[[]]` 미동작)
- `path/to/relevant/file.py`

<!-- ── 이하 필수: body 최하단 고정 ── -->

## History
- `2026-08-12T14:32+09:00` · created · ticket-plan · MinhoWindowPC · `6204fbb6-3a3f-4494-8ed6-aedba0a33410`
```

History 는 생성 시 `created` 한 줄, 이후 body 를 고칠 때마다 `edited` 한 줄 append. 세션 ID 뒤 변경 서술 금지 (남길 때도 명사구 3개 이하). 상세: `STYLE.md`.

**상호 링크 규칙**:
- Issue → Issue: `#123` (자동 링크)
- Issue → Wiki: 전체 URL (`[표시 텍스트](${wiki.url_prefix}{Page-Name})`)
- Issue → 파일: 백틱 경로 (`src/foo.py`) 또는 GitHub blob URL

## 라벨

| 종류 | 라벨 |
|---|---|
| **상태** (open 시 추가) | `in-progress`, `needreview`, `backlog` |
| **상태** (closed) | 라벨 없음, GitHub state = closed = completed |
| **우선순위** | `priority:high`, `priority:urgent`, `priority:low` (medium 은 라벨 없음 = default) |
| **카테고리** (1개 필수) | `feature`, `bugfix`, `refactor`, `performance`, `docs`, `infra` |
| **프로젝트 고유 라벨** | schema 외부. 필요해질 때 사용자가 직접 추가하거나 `labels.components` 항목 확장 |

## 예시

### 예시 1: Feature 티켓

`gh issue create --repo "${project.repo}" --title "Add WebSocket Real-time Price Updates" --label "feature,priority:high" --body-file /tmp/issue.md`

본문:
```markdown
## Original Request
> 대시보드에 실시간 BTC 가격 업데이트가 필요해. WebSocket으로 처리하자.

## Goal
대시보드에서 BTC 가격을 WebSocket 기반 실시간 스트림으로 표시.

## Background
현재 30초 폴링 방식으로 가격을 가져온다. UX 개선을 위해 실시간 푸시로 전환.

## Technical Details

### as-is
- `PriceFetcher.poll()` 이 30 초 주기 REST 폴링, 대시보드는 폴링 결과를 렌더

### to-be
- `websockets` 라이브러리로 Binance WebSocket 가격 스트림 구독
- 연결 끊김 시 exponential backoff 자동 재연결

### not-to-be
- REST 폴링 경로는 WebSocket 미지원 환경 fallback 으로 유지 — 제거는 별건

## Implementation Plan
- [ ] **Phase 1: WebSocket 클라이언트 구현**
  - Files: `src/data_collection/fetchers/binance_ws.py`
- [ ] **Phase 2: API 스트림 엔드포인트 추가**

## Acceptance Criteria
- 가격 변동이 500ms 이내 대시보드 반영
- 연결 끊김 시 자동 복구
- 장시간 실행 (24h+) 시 메모리 leak 없음

## History
- `2026-08-12T14:32+09:00` · created · ticket-plan · MinhoWindowPC · `6204fbb6-3a3f-4494-8ed6-aedba0a33410`
```

### 예시 2: 버그 픽스 (필수 5 만)

`gh issue create --repo "${project.repo}" --title "Fix LSTM Memory Leak During Training" --label "bugfix,priority:urgent" --body-file /tmp/issue.md`

```markdown
## Original Request
> 50 epoch 넘기면 OOM 떨어진다. 학습 중 메모리 누수 잡아줘.

## Goal
LSTM 학습이 200 epoch 이상 진행되어도 메모리가 안정 유지되도록 누수 제거.

## Background
50 epoch 이후 메모리 무한 증가하다 OOM. 세션 정리 누락 의심.

## Technical Details
- 의심 지점: epoch 사이 `K.clear_session()` 누락
- 검증: `tracemalloc` 으로 200 epoch 메모리 추세 측정

## History
- `2026-08-12T09:07+09:00` · created · ticket-plan · MinhoWindowPC · `df10f8c9-7204-4eca-86aa-d8fe0a0b0fa4`
- `2026-08-12T11:20+09:00` · edited · ticket-implement · MinhoWindowPC · `a3f1c802-55e9-4d1b-9c7a-2b6e0f4d8813`
```

**3 파트 미적용 사례**: 원인이 미확정이라 `as-is` (무엇이 사실인가) 와 `to-be` (무엇을 바꾸는가) 가 아직 갈리지 않음. 조사 결과가 나오면 3 파트로 재작성.

## PR 컨벤션

PR 생성 형식은 `ticket-complete` Step 4 참조. 핵심:
- PR 제목: `{N}-{Short description}` (예: `115-Add WebSocket real-time price updates`)
- PR 본문: `Closes #{N}` 포함 → 머지 시 Issue 자동 close

## 권장 사항

- Goal 은 결과 (outcome) 를 기술하지 활동 (activity) 을 적지 않습니다
- Technical Details 는 as-is / to-be / not-to-be 3 파트 기본. 미적용은 3 파트가 성립하지 않을 때만
- Implementation Plan Phase 는 작고 체크 가능한 단위
- Related 에 코드 파일 경로 + Issue/Wiki 링크 혼재 허용
- 옵셔널 섹션은 필요할 때만, 빈 섹션 금지
- `## History` 는 body 최하단 필수 — 편집마다 한 줄
- 우선순위 medium 은 라벨 없이 default
- 작성 후 한 번 더 사용자 확인 → 그 다음 worktree 생성
- 분할 시 의존성을 본문 `## Related` 와 라벨 `split-group:*` 둘 다로 추적
