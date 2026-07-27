---
name: ticket-doc
description: GitHub Issue 와 연동된 기술 문서를 GitHub Wiki 페이지로 생성·관리합니다. 배경 조사, 설계 결정, 구현 분석, 실험 결과, 브레인스토밍, 상세 기술 문서를 작성할 때 사용. ticket-plan 및 ticket-implement 가 자동 호출하거나, 브레인스토밍/사후 분석을 위해 수동 호출. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Documentation

GitHub Issue 와 연동된 기술 문서를 **GitHub Wiki** 에 생성 및 관리합니다.

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` (+ 로컬 `ticket-skill-configs.local.yaml` override, 있으면) 로드·병합 → minimal 4 키 검증 → **계정 라우팅** → `${var}` 치환. 상세: `~/.claude/skills/ticket-plan/CONFIG.md`.

**계정 라우팅** (멀티 계정 — 옵셔널): `project.gh_account` 가 있으면 `gh` 명령 전에 `gh auth switch --user ${project.gh_account}`. `${wiki.repo_url}` 은 `git@${project.ssh_host}:${project.repo}.wiki.git` 로 유도되므로 (ssh_host 기본 `github.com`), 두 번째 계정 repo 의 wiki clone/push 가 올바른 SSH 키를 사용. 세 키 모두 없으면 단일 계정 동작.

## 데이터 소스

- **Wiki repo**: `${wiki.repo_url}` (= `git@${project.ssh_host}:${project.repo}.wiki.git`)
- **로컬 클론 경로**: `${wiki.clone_path}` (main repo 의 sibling)
- **Wiki 는 main repo 와 별도의 git repo** — 자체 master 브랜치, 별도 commit/push
- **드래프트 → 발행 2단계**: 문서 작성 = 로컬 커밋(드래프트)까지만. 발행(push)은 명시적 단계 (Step 6) 에서만
- 페이지 = 마크다운 파일 (`.md`). 파일명이 곧 페이지 슬러그.

## 지연 초기화 — Wiki 클론 (첫 호출 시)

`${wiki.clone_path}` 디렉토리가 없으면 클론:

```bash
# main repo parent 디렉토리로 이동 후 clone
git clone ${wiki.repo_url} ${wiki.clone_path}
```

이미 클론되어 있으면 스킬 호출 시 `git pull` 만 수행.

## 호출 모드

| 모드 | 트리거 | 설명 |
|---|---|---|
| **자동 (ticket-plan)** | ticket-plan 의 문서화 단계에서 자동 호출 | 계획 단계 분석이 상세할 때 Type A 페이지 생성 |
| **자동 (ticket-implement)** | ticket-implement 의 Type B/C 에서 자동 호출 | 구현/완료 결과가 상세할 때 Type B 페이지 생성 |
| **독립 (Standalone)** | 사용자가 직접 `/ticket-doc` 호출 | 기존 Issue 기반 브레인스토밍·추가 분석 등 Type C 페이지 생성 |

**자동 모드**: ticket-plan/ticket-implement 가 판단 기준에 따라 자동 호출. 별도 사용자 개입 없이 페이지가 생성되고 Issue body 또는 comment 에 wiki URL 이 추가됩니다.

**독립 모드**: 사용자가 직접 호출. 예시:
- "ticket-doc 으로 #115 의 아이디어를 바탕으로 접근 관점만 바꿔서 테스트해볼거리 브레인스토밍해서 문서로 작성하자"
- "#58 의 실험 결과를 정리한 문서를 만들자"
- "#25 의 모델 아키텍처를 상세하게 문서화하자"

## 파일명 규칙

```
{issue-number}-{seq}-{title-kebab}.md
```

| 요소 | 설명 | 예시 |
|---|---|---|
| `issue-number` | Issue 번호 (자릿수 보장 불요) | `115`, `58`, `7` |
| `seq` | 해당 Issue 내 문서 순번 (1 부터) | `1`, `2`, `3` |
| `title-kebab` | 문서 내용을 나타내는 kebab-case 제목 | `market-condition-analysis` |

**예시 파일명**:
- `115-1-websocket-flow-analysis.md`
- `58-1-market-condition-analysis.md`
- `25-1-swing-model-architecture.md`

**Wiki URL 매핑**:
- 파일명 `115-1-websocket-flow-analysis.md`
- → URL `${wiki.url_prefix}115-1-websocket-flow-analysis`
- → 페이지 표시 제목: 파일의 `# ` 헤더가 표시되거나 (없으면) 파일명이 그대로 표시

## 사용 절차

### Step 1: 컨텍스트 확인

1. **Issue 번호 확인**:
   - **자동 모드**: 호출한 ticket-plan/ticket-implement 에서 현재 작업 중인 Issue 번호 이어받음
   - **독립 모드**: 사용자가 지정한 Issue 번호로 `gh issue view {N} --repo "${project.repo}"` 로 내용 파악
2. **Wiki repo 동기화** (지연 클론 포함):
   ```bash
   cd ${wiki.clone_path} || git clone ${wiki.repo_url} ${wiki.clone_path} && cd ${wiki.clone_path}
   git pull
   ```
3. **기존 문서 확인**: 같은 Issue 번호의 페이지가 있는지 확인
   ```bash
   ls ${wiki.clone_path}/{N}-*.md 2>/dev/null
   ```
4. **순번 결정**: 해당 Issue 의 마지막 순번 + 1 (첫 문서면 1)

### Step 2: 문서 유형 결정

#### Type A: 계획 단계 문서 (ticket-plan 연동)

**작성 시점**: ticket-plan 으로 계획 중, 상세한 배경/아이디어/로직 정리가 필요할 때

**포함 내용**:
- 문제 정의 및 배경 조사 결과
- 관련 코드/아키텍처 분석
- 설계 아이디어와 대안 비교
- 핵심 알고리즘/로직 설명
- 데이터 흐름 및 의존성 분석
- 참고 자료 및 레퍼런스

#### Type B: 구현/분석 단계 문서 (ticket-implement 연동)

**작성 시점**: ticket-implement 로 구현 중 또는 완료 후, 주요 결과/분석을 기록할 때

**포함 내용**:
- 실험 결과 및 데이터 분석
- 성능 측정 결과 (메트릭, 벤치마크)
- 구현 과정에서의 기술적 발견
- 디버깅/트러블슈팅 기록
- 최종 결론 및 인사이트
- 후속 작업 제안 (현재 Issue 범위 밖 발견사항 단순 리스트. 다른 Issue 에 자동 반영 X)

#### Type C: 독립 문서 (Standalone)

**작성 시점**: 사용자가 직접 `/ticket-doc` 을 호출하여 기존 Issue 기반으로 추가 문서를 작성할 때

**독립 모드 전용 Step**:
1. 사용자 요청에서 **대상 Issue 번호** 와 **문서 목적** 파악
2. `gh issue view {N} --repo "${project.repo}"` 로 현재 상태·배경·결과 컨텍스트 파악
3. 해당 Issue 의 기존 wiki 페이지도 읽어서 중복 방지
4. 필요 시 관련 코드/데이터 추가 탐색
5. 문서 목적에 맞는 내용을 **대화형** 으로 사용자와 함께 구성

**포함 내용** (목적에 따라 유연):
- 브레인스토밍: 아이디어 발산, 접근 관점 변경, 실험 설계
- 심화 분석: 기존 결과의 추가 해석, 다른 각도 분석
- 사후 정리: 완료된 작업의 종합 정리, 지식 문서화
- 비교 연구: 기존 접근법 vs 대안, 벤치마크 설계
- 문제 분해: 복잡한 문제를 서브 문제로 분해

### Step 3: 문서 작성

1. **템플릿 선택**: Step 2 에서 결정한 유형의 템플릿 사용
2. **하이레벨 + 디테일 균형**: 큰 그림 설명 → 각 섹션 디테일 빠짐없이
3. **구조화**: 표, 코드 블록, 다이어그램 적극 활용
4. **데이터 보존**: 수치, 파라미터, 설정값 등 재현에 필요한 정보 모두 포함
5. 파일 저장 위치: `${wiki.clone_path}/{N}-{seq}-{title-kebab}.md`

### Step 4: 상호 참조 링크 (필수 — 빠짐없이 실행)

문서 생성 후 아래 3가지 방향 링크를 **모두** 확인하고 추가합니다.

#### 4-1. 문서 → Issue (wiki 페이지 내부)
- 문서 본문 상단에 `> 관련 Issue: #{N}` 한 줄 — GitHub 가 자동으로 Issue 링크 렌더
- 별도 frontmatter 불필요 (GitHub Wiki 는 frontmatter 자동 처리 X)

#### 4-2. Issue → 문서 (Issue body 또는 comment)

**자동 (ticket-plan) 인 경우**: Issue body 의 `## Related` 섹션에 wiki URL 추가
```bash
gh issue edit {N} --repo "${project.repo}" --body-file /tmp/issue-updated.md
```
업데이트된 body 의 `## Related` 항목:
```markdown
- [WebSocket Flow Analysis](${wiki.url_prefix}115-1-websocket-flow-analysis) — 분석 문서
```

**자동 (ticket-implement) 또는 독립 모드인 경우**: Issue comment 로 추가
```bash
gh issue comment {N} --repo "${project.repo}" --body "관련 문서 추가: [WebSocket Flow Analysis](${wiki.url_prefix}115-1-websocket-flow-analysis)"
```

#### 4-3. 문서 ↔ 문서 (양방향)

- 같은 Issue 의 기존 wiki 페이지 확인: `ls ${wiki.clone_path}/{N}-*.md`
- 기존 문서가 있으면:
  1. **새 문서 → 기존 문서**: 새 문서 상단에 `> 기반 문서: [[{기존 페이지 파일명}]]` (wiki 내부에서는 `[[Page-Name]]` syntax 작동)
  2. **기존 문서 → 새 문서**: 기존 문서의 `## 후속 작업` 또는 본문 말미에 새 문서 링크 추가 (`[[새-페이지명]]`)
- 다른 Issue 의 문서라도 내용상 관련이 있으면 양방향 링크 추가

**Wiki 내부 링크 syntax**:
- `[[115-1-websocket-flow-analysis]]` — 페이지 파일명 그대로
- `[[표시 텍스트|115-1-websocket-flow-analysis]]` — 표시 텍스트 커스텀
- 단, **Issue body/comment 에서는 `[[]]` syntax 작동 안 함** — 풀 URL 사용

**체크리스트**:
- [ ] wiki 페이지 본문 `> 관련 Issue: #{N}` 포함
- [ ] Issue 의 `## Related` 또는 comment 에 wiki URL 추가됨
- [ ] 같은 Issue 의 기존 wiki 페이지에 새 문서 링크 추가됨 (기존 문서 있는 경우)
- [ ] 새 문서에 기존 wiki 페이지 링크 포함됨 (기존 문서 있는 경우)

### Step 5: Wiki 로컬 commit — 드래프트 (push 안 함)

문서는 **로컬 `.wiki` 클론에 commit 까지만** 하고 **push 하지 않습니다** (= 드래프트). 발행(push)은 Step 6 에서 명시적으로만 일어납니다.

```bash
cd ${wiki.clone_path}
git add {N}-{seq}-{title-kebab}.md
git commit -m "docs: #{N} - {문서 제목 요약}"   # 로컬 커밋만 — push X
cd -
```

**모든 모드 공통**: 이 스킬은 **절대 자동으로 push 하지 않습니다.** 자동(ticket-plan/ticket-implement)·독립 모드 모두 드래프트(로컬 커밋)에서 멈춥니다.

### Step 6: 발행 — 명시적 push (드래프트 → 공개)

로컬 드래프트를 GitHub Wiki 에 실제 공개하는 단계. **명시적 트리거로만** 실행합니다.

| 트리거 | 발행 주체 |
|---|---|
| 자동 파이프라인 (ticket-plan → implement → complete) | **ticket-complete 의 "Wiki 발행" 단계**가 누적 드래프트를 일괄 push |
| 독립 `/ticket-doc` | 사용자가 **발행을 명시적으로 요청**("발행" / "publish" / "push")할 때만 |

```bash
cd ${wiki.clone_path}
git push        # 여기서 비로소 공개
cd -
```

**독립 모드 종료 보고** (드래프트만 하고 멈춤 — 발행 여부는 사용자가 결정):
```
로컬 드래프트 저장됨: {N}-{seq}-{title-kebab}.md (아직 미발행)
발행하려면: (cd ${wiki.clone_path} && git push)  또는 "발행" 이라고 요청하세요.
```

## 템플릿

### 템플릿 A: 계획 단계 문서

```markdown
# {문서 제목}

> 관련 Issue: #{N}

## 개요

{이 문서의 목적과 범위를 1-2 문단으로 설명}

## 배경

{문제의 컨텍스트, 현재 상태, 왜 이 작업이 필요한지}

### 현재 구현 분석

{기존 코드/시스템의 관련 부분 분석}

```python
# 핵심 코드 스니펫 (필요시)
```

## 설계

### 핵심 아이디어

{주요 접근 방식과 그 이유}

### 대안 검토

**원칙**: 네이티브/공식 API 방식을 항상 첫 번째 후보로 평가. 기각 사유가 명확한 경우에만 커스텀 구현 선택.

| 방식 | 장점 | 단점 | 네이티브 여부 | 선택 여부 |
|---|---|---|:---:|---|
| {방식 A} | {장점} | {단점} | O/X | {선택/기각} |
| {방식 B} | {장점} | {단점} | O/X | {선택/기각} |

### 상세 로직

{알고리즘, 데이터 흐름, 처리 로직 상세 기술}

### 의존성 및 영향 범위

{변경으로 영향받는 모듈, 파일, 시스템}

## 참고 자료

- {레퍼런스 링크 또는 문서}
```

### 템플릿 B: 구현/분석 단계 문서

```markdown
# {문서 제목}

> 관련 Issue: #{N}

## 개요

{이 문서에서 다루는 내용 요약}

## 실행 환경

| 항목 | 값 |
|---|---|
| {설정 항목} | {값} |

## 결과

### {결과 섹션 1}

| 지표 | 값 |
|---|---|
| {metric} | {value} |

### {결과 섹션 2}

{추가 분석 결과}

## 핵심 발견

- **발견 1**: {설명}
- **발견 2**: {설명}
- **발견 3**: {설명}

## 결론 및 제안

{분석 결과 기반 결론과 다음 단계}

## 후속 작업

{현재 Issue 범위 밖에서 발견한 후속 항목. 단순 리스트만 — 다른 Issue 자동 수정 X. 별도 Issue 생성은 ticket-complete 단계에서 사용자 확인 후. 없으면 섹션 생략.}

- {후속 작업 1}
- {후속 작업 2}
```

### 템플릿 C: 독립 문서

```markdown
# {문서 제목}

> 관련 Issue: #{N}
> 기반 문서: [[{기존 wiki 페이지 파일명}]] (있는 경우)

## 배경 및 동기

{왜 이 문서를 작성하게 되었는지. 기존 Issue/문서의 어떤 내용에서 파생됐는지}

## 핵심 질문

1. {질문 1}
2. {질문 2}
3. {질문 3}

## 탐색

### {관점/아이디어 1}

{상세 분석 및 근거}

### {관점/아이디어 2}

{상세 분석 및 근거}

### {관점/아이디어 N}

{상세 분석 및 근거}

## 종합 정리

| 아이디어 | 실현 가능성 | 기대 효과 | 우선순위 |
|---|---|---|---|
| {아이디어 1} | {상/중/하} | {설명} | {1/2/3} |

## 다음 단계

- [ ] {액션 1}
- [ ] {액션 2}
```

## 작성 가이드라인

### 하이레벨 + 디테일 원칙

1. **개요로 시작**: 전체 맥락 먼저 파악할 수 있게
2. **섹션별 깊이**: 각 섹션 내 디테일 빠짐없이 기록
3. **수치 보존**: 실험 파라미터, 결과 수치, 설정값 반드시 포함
4. **재현 가능성**: 동일한 결과를 재현할 수 있는 수준의 정보
5. **결정 근거**: "왜 이렇게 했는지" 항상 기록

### 문서 품질 체크리스트

- [ ] 본문 `> 관련 Issue: #{N}` 한 줄 포함
- [ ] Issue 의 Related (body 또는 comment) 에 이 wiki URL 추가됨
- [ ] 같은 Issue 의 기존 wiki 페이지에 이 문서 링크 추가됨 (해당 시)
- [ ] 이 문서에 기존 관련 wiki 페이지 링크 포함 (해당 시)
- [ ] 개요에서 문서 목적이 명확한가
- [ ] 핵심 수치/파라미터가 모두 기록되었는가
- [ ] 표와 코드 블록으로 가독성 확보
- [ ] 결론/인사이트가 명확하게 정리되었는가

### 구조화 도구 활용

| 도구 | 용도 |
|---|---|
| **표** | 파라미터 비교, 메트릭 정리, 대안 비교 |
| **코드 블록** | 핵심 로직, 설정값, CLI 명령 |
| **ASCII 다이어그램** | 데이터 흐름, 아키텍처, 프로세스 |
| **인용 블록** | 원본 요구사항, 참고 문헌 |
| **체크리스트** | 검증 항목, TODO |

## 상호 링크 치트 시트

| 위치 | 대상 | 문법 |
|---|---|---|
| Wiki 페이지 → Issue | Issue #N | `#{N}` (자동 링크) |
| Wiki 페이지 → Wiki 페이지 | 다른 페이지 | `[[Page-Filename]]` 또는 `[[표시 텍스트\|Page-Filename]]` |
| Wiki 페이지 → 코드 파일 | 백틱 경로 | `` `src/foo.py` `` 또는 GitHub blob URL |
| Issue body/comment → Wiki | wiki 페이지 | 풀 URL `[제목](${wiki.url_prefix}{Page-Filename})` |
| Issue body/comment → Issue | 다른 Issue | `#{N}` |

## 권장 사항

- 문서 하나의 주제는 하나로 집중 (여러 주제면 여러 페이지로 분리)
- Issue 완료 후에도 wiki 페이지는 유지 (지식 자산)
- 기존 wiki 페이지와 중복 방지 (Step 1 의 기존 문서 확인 필수)
- 문서 내용이 짧으면 (1페이지 이하) Issue body/comment 에 직접 기록하는 것이 나음 → 별도 wiki 페이지 불필요
- **자동 모드**: 판단 기준에 해당하지 않으면 무조건 스킵 (과도한 문서 생성 방지)
- **독립 모드**: 반드시 기존 Issue 와 연동 (Issue 없는 독립 문서는 대상 X)
- 독립 모드로 브레인스토밍 문서 작성 후, 실행할 아이디어가 나오면 `ticket-plan` 으로 새 Issue 생성 권유
- **Type B 의 후속 작업**: 단순 리스트로만 — 다른 Issue 를 자동 수정하거나 라벨/comment 를 추가하지 않습니다 (정보량 폭증 방지)
- Wiki repo 는 main repo 와 별도 — **자동 push 안 함**. 발행은 명시적 단계(Step 6)에서만 (드래프트 → 발행 분리)
