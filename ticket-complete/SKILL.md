---
name: ticket-complete
description: GitHub Issue 를 end-to-end 로 마무리합니다. 커밋, 푸시, Closes #N 포함 PR 생성, 머지, worktree/브랜치 정리, 구현 결과 comment 게시. 모든 acceptance criteria 가 충족된 후 사용. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Complete

GitHub Issue 구현 완료 후 전체 마무리 프로세스를 한 번에 실행합니다.
커밋 → PR 생성 (Closes #N) → PR 머지 → worktree/브랜치 정리 → 구현 결과 comment → Issue 라벨 정리·close → wiki 발행(push).

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` (+ 로컬 `ticket-skill-configs.local.yaml` override, 있으면) 로드·병합 → minimal 4 키 검증 → **계정 라우팅** → `${var}` 치환. 상세: `~/.claude/skills/ticket-plan/CONFIG.md`.

`${branch}`·`${base}` 는 config 가 아니라 Issue body 에서 나온다 — Step 1 에서 확정한다.

**계정 라우팅** (멀티 계정 — 옵셔널): `project.gh_account` 가 있으면 PR/커밋/push 전에 `gh auth switch --user ${project.gh_account}` 실행. `project.git_email` 이 있으면 커밋 전 `git config user.email ${project.git_email}` (로컬). `${wiki.repo_url}` 은 `git@${project.ssh_host}:...` 로 유도 (ssh_host 기본 `github.com`) — wiki push 가 올바른 SSH 키 사용. main repo push 는 `origin` remote URL 을 따름. 세 키 모두 없으면 단일 계정 동작.

## 데이터 소스

- **티켓**: GitHub Issues (`${project.repo}`)
- **PR ↔ Issue 연동**: PR body 의 `Closes #{N}` 으로 머지 시 Issue 자동 close. **base 가 repo 기본 브랜치일 때만 동작** — 아니면 Step 9-6 이 명시 close (CONFIG.md "브랜치 해석")
- **Wiki**: ticket-doc 가 드래프트(로컬 커밋)로 남긴 wiki 페이지를 이 스킬이 **발행(push)** — 자동 파이프라인의 유일한 발행 지점

## 사용 시점
- Issue 구현이 끝나고 모든 acceptance criteria 가 통과된 후
- `/ticket-complete {issue-number}` 또는 `/ticket-complete {issue-number}-{slug}` 형태로 호출

## 사전 조건
- Worktree 에서 작업 중 (`git worktree list` 로 확인) — 코드 변경 흐름인 경우
- 코드 변경 완료 + 빌드/테스트 통과 — 코드 변경 흐름인 경우
- Issue 가 open 상태이고 `in-progress` 라벨이 있음 (선택)

## 변경 유형

- **코드 변경 + 머지 흐름 (기본)**: Step 1~11 전체 수행
- **draft PR 흐름** (`pr.draft: true`): Step 1~4 로 draft PR 을 열고 **Step 5 에서 중단**. Step 6~8(머지·pull·정리) 스킵, Step 9~11 은 수행하되 Issue 는 열어 둔다. 사람이 draft 해제·CI 확인 후 재호출하면 Step 5 부터 이어간다
- **Docs-only 변경** (글로벌 스킬, 외부 문서 등 worktree·PR 불필요): Step 1 의 Issue 확인 후 **Step 2~8 스킵**. Step 9 (라벨 정리 / 체크리스트 / 구현 결과 comment / close), Step 10 (wiki 발행) 만 수행. PR/머지가 없으므로 Issue 자동 close 도 발생하지 않음 → Step 9-6 의 명시 close 수행

## 사용 절차

### Step 1: 상태 확인

1. **Issue 확인**: `gh issue view {N} --repo "${project.repo}" --json title,body,labels,state,url,comments`
2. **브랜치 해석**: body `## Related` 의 브랜치 줄로 `${branch}`·`${base}` 확정 (CONFIG.md "브랜치 해석")
3. **Worktree 확인**: `git worktree list` 로 `${worktree.prefix}-{N}-{slug}` 존재 여부
4. **현재 위치 확인**: `git branch --show-current`
5. **repo 기본 브랜치 확인**: `gh repo view "${project.repo}" --json defaultBranchRef --jq .defaultBranchRef.name` — `${base}` 와 다르면 `Closes #{N}` 이 안 먹으므로 Step 9-6 에서 명시 close

### Step 2: Worktree 에서 커밋

```bash
cd ../${worktree.prefix}-{N}-{slug}
git status -u
```

1. 미커밋 변경이 있으면 → 스테이징 후 커밋
2. 커밋 메시지 형식:
   ```
   feat: {구현 요약}

   - {주요 변경 1}
   - {주요 변경 2}

   Refs #{N}

   Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
   ```
3. 이미 커밋된 경우 스킵

### Step 3: 브랜치 푸시

```bash
git push -u origin ${branch}
```

### Step 4: PR 생성

```bash
gh pr create --repo "${project.repo}" --base ${base} ${pr.draft ? --draft} --title "{N}-{Short description}" --body "$(cat <<'EOF'
## Summary
- {변경사항 요약}

## Related Issue
Closes #{N}

## Changes
- {주요 변경 1}
- {주요 변경 2}

## Test Plan
- [x] {테스트 항목}

🤖 Generated with [Claude Code](https://claude.com/claude-code)

---
<sub>ticket-complete · MinhoWindowPC · `6204fbb6-3a3f-4494-8ed6-aedba0a33410`</sub>
EOF
)"
```

**필수**:
- `--base ${base}` (Issue 브랜치 줄이 없으면 `${project.default_branch}`)
- `pr.draft: true` 면 `--draft` 추가 (CONFIG.md "PR 운용"). 기본은 붙이지 않음
- 본문에 `Closes #{N}` 한 줄 — base 가 repo 기본 브랜치면 머지 시 자동 close, 아니면 기록용이고 Step 9-6 이 닫는다
- PR 제목: `{N}-{Short description}`
- 최하단 세션 서명 footer (머신 · 세션 ID) — `~/.claude/skills/ticket-plan/STYLE.md` "History · 세션 서명"

### Step 5: PR 머지

**`pr.draft: true` 면 여기서 중단한다.** draft PR 은 CI 가 안 돌아 머지 판단 근거가 없다. `gh pr ready` 를 **호출하지 말고** PR URL 과 "draft 해제 → CI 통과 확인 후 재호출" 을 보고한 뒤 Step 6~8(머지 의존 단계)을 건너뛴다. Step 9~11(Issue comment·라벨·wiki 발행)은 그대로 수행하되 Issue 는 열어 둔다 (9-6 도 건너뛴다) — 나중에 머지될 때 닫힌다.

**중요: 사용자 확인 필수**
PR 머지 전 사용자에게 빌드/테스트 결과 확인 여부를 묻습니다. "머지해", "테스트 완료", "확인 완료" 등 명시 승인 없으면 머지 금지.

```bash
gh pr merge {pr-number} --repo "${project.repo}" --${pr.merge_method}
```

`pr.merge_method` 기본값은 `squash`. "티켓 1개 = 커밋 1개" 를 쓰는 프로젝트는 config 에서 `merge` 로 바꾼다 — squash 는 커밋 단위 bisect·부분 revert 를 죽인다.

`${base}` 가 repo 기본 브랜치면 머지 성공 시 Issue `#{N}` 은 GitHub 가 자동 close 한다. 아니면 열린 채로 남으니 Step 9-6 에서 명시 close.

### Step 6: Main Repo 이동 및 Pull

```bash
cd ${project_root}
git checkout ${base}
git pull
```

### Step 7: Worktree 정리

**`git worktree remove` 전에 worktree 안의 symlink / junction 을 먼저 제거한다.** 링크가 남아 있으면 삭제가 링크를 타고 들어가 원본 데이터를 지운다.

```powershell
$wt = "../${worktree.prefix}-{N}-{slug}"
$links = Get-ChildItem $wt -Recurse -Force -Attributes ReparsePoint
$targets = $links.Target                                    # 삭제 전에 타깃 기록
$links | ForEach-Object { $_.Delete() }                     # 링크 자체만 삭제 (비재귀)
Get-ChildItem $wt -Recurse -Force -Attributes ReparsePoint  # 잔존 0 확인 (남으면 중단)
git worktree remove $wt
$targets | ForEach-Object { Test-Path $_ }                  # 원본 보존 확인 (전부 True)
```

- POSIX 환경: `find "$wt" -type l -delete`
- 실패 시 (미커밋 변경 등) 사용자 보고 후 `--force` 옵션 사용 여부 확인
- **`rm -rf` / `Remove-Item -Recurse` 금지** — 링크를 타고 원본이 삭제됨

### Step 8: 브랜치 정리

```bash
git branch -d ${branch}
git push origin --delete ${branch}
```

이미 삭제됐으면 스킵.

### Step 9: Issue 완료 처리

`${base}` 가 repo 기본 브랜치면 Issue 는 머지로 이미 closed 다. 아니면 9-6 이 닫는다. 공통으로 다음을 수행:

#### 9-1. 라벨 정리
```bash
gh issue edit {N} --repo "${project.repo}" --remove-label "in-progress,needreview"
```

#### 9-1b. 보드 동기화
config 에 `board` 가 있으면 `Set-BoardStatus({N}, done)` (CONFIG.md "프로젝트 보드 Status 동기화") 실행. 코드 흐름(머지 후)·docs-only 흐름 공통. 보드 미설정이면 skip.

#### 9-2. Implementation Plan 체크리스트 검증
Issue body 에 Implementation Plan 이 있으면:
- 모든 `- [ ]` 항목을 `- [x]` 로 변경하여 `gh issue edit {N} --repo "${project.repo}" --body-file ...` 로 본문 업데이트
- 실제 구현되지 않은 항목은 `(스킵: {사유})` 표기 후 체크
- Acceptance Criteria 도 동일하게 처리
- `## History` 에 `edited` 한 줄 (스킬 = `ticket-complete`)

#### 9-3. 구현 결과 comment 추가

```bash
gh issue comment {N} --repo "${project.repo}" --body-file /tmp/results.md
```

`/tmp/results.md` 내용은 `ticket-implement` SKILL.md 의 **구현 결과 Comment Template** 참조. 필드:
- PR (번호 + URL)
- Changed Files (표)
- Test Results
- Notes / Issues
- Follow-up (있으면)
- Completed (오늘 날짜)
- 세션 서명 footer

#### 9-4. 진행 comment 의 후속 작업 종합

이전 진행 comment 들에서 Follow-up 항목을 모아 9-3 의 구현 결과 comment 의 Follow-up 섹션에 통합 (중복 제거).

#### 9-5. 후속 작업 확인 (사용자 결정)

구현 결과의 Follow-up 섹션을 사용자에게 그대로 보여주고 별도 Issue 생성 여부를 묻습니다.

**표시 형식**:
```
완료 처리 전 Follow-up 확인:

1. 다른 collector 에도 동일한 retry 누락 있음
2. TF 메모리 정리 패턴은 #45 와 관련 가능

각 항목에 대해 응답해주세요:
- `1 new`: 1번을 새 Issue 로 생성
- `2 skip`: 2번은 그냥 기록만
- `1 new, 2 skip`: 일괄 응답 가능
```

**처리 원칙 (정보량 폭증 방지)**:
- **`new` 로 표시된 항목만** `ticket-plan` 절차로 새 planned Issue 생성. 새 Issue body 의 `## Related` 에 `#{N}` 을 `출처` 로 링크.
- **`skip` 은 구현 결과 comment 의 Follow-up 에 그대로 남겨두기만** 합니다 (다른 Issue 에 자동 적용·전파 X).
- 사용자가 응답하지 않거나 모호하면 모두 `skip` 처리.
- **다른 Issue 를 스캔하거나, 자동으로 라벨/comment 를 추가하지 않습니다.**

#### 9-6. 명시 close (자동 close 가 없는 경우)

`${base}` 가 repo 기본 브랜치가 아니거나 docs-only 흐름이면 GitHub 가 Issue 를 닫지 않는다:

```bash
gh issue close {N} --repo "${project.repo}" --reason completed
```

머지가 끝난 경우에만 호출한다 — draft 흐름은 Issue 를 열어 둔다 (Step 5).

### Step 10: Wiki 발행 (해당 시)

구현/계획 중 `ticket-doc` 이 남긴 wiki **드래프트**(로컬 커밋)를 여기서 **발행(push)** 합니다 — 자동 파이프라인의 유일한 발행 지점:

```bash
cd ${wiki.clone_path}
git status
# 드래프트가 로컬 커밋으로 쌓여 있음 → push 로 발행
# (혹시 미커밋 변경이 남아 있으면 함께 커밋)
git add . && git diff --cached --quiet || git commit -m "docs: #{N} - {요약}"
git push
cd -
```

발행할 드래프트(로컬 미push 커밋)도, 미커밋 변경도 없으면 스킵.

### Step 11: 최종 확인

```bash
git worktree list    # worktree 정리 확인
git branch           # 브랜치 정리 확인
git log --oneline -5 # 머지 결과 확인
gh issue view {N} --repo "${project.repo}"    # Issue closed + 라벨 정리 확인
```

사용자에게 완료 보고:
```
Issue #{N} 완료:
- PR #{PR번호} 머지 완료: {PR URL} (base: ${base})
- Issue close 완료 ({자동 / 명시})
- Worktree 정리 완료
- 브랜치 정리 완료 (local + remote)
- 구현 결과 comment 추가 완료
- Follow-up: {새 Issue N건 생성: #X, #Y / 기록만 / 없음}
- Wiki: {N개 드래프트 발행(push) 완료 / 발행할 드래프트 없음}
```

## 오류 처리

- **PR 머지 실패**: CI 체크 실패 또는 conflict → 원인 안내. `Pull request is in draft state` 면 Step 5 의 draft 게이트를 안 탄 것 — 머지하지 말고 보고
- **`--merge` 거부**: repo 설정이 merge commit 을 막고 있다 → Settings > General > Pull Requests 의 "Allow merge commits" 확인 안내. 임의로 `--squash` 대체 금지
- **worktree 제거 실패**: 미커밋 변경 → 사용자 확인 후 `--force`
- **브랜치 삭제 실패**: 이미 삭제됨 → 스킵
- **Issue 자동 close 실패**: `${base}` 가 repo 기본 브랜치가 아니거나 PR body 의 `Closes #{N}` 누락 → Step 9-6 의 명시 close
- **Wiki push 실패**: 충돌 시 `git pull --rebase` 후 재시도

## 주요 참고 사항

- **브랜치·base**: `${branch}` / `${base}` — Issue body 브랜치 줄, 없으면 `{N}-{slug}` / `${project.default_branch}` (CONFIG.md "브랜치 해석")
- **draft 해제는 사람이 한다**: `pr.draft: true` 일 때 스킬이 `gh pr ready` 를 호출하면 CI 발동 시점 통제가 사라진다. 금지
- **머지 방식**: `${pr.merge_method}` (기본 `squash`)
- **링크/junction 주의**: worktree 정리는 링크 제거 후 `git worktree remove` (Step 7). `rm -rf` / `Remove-Item -Recurse` 금지
- **PR body 의 `Closes #{N}` 필수**: 누락 시 Issue 가 자동으로 close 되지 않음 (base 가 기본 브랜치일 때). 그 외 경우는 Step 9-6 이 닫는다
- **후속 작업 처리**: 사용자가 `new` 로 명시한 항목만 새 Issue. 다른 Issue 자동 수정·전파 금지
- **Wiki 와 main repo 는 별도 git repo**: wiki 변경은 `${wiki.clone_path}` 에서 별도로 commit/push
