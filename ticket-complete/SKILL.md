---
name: ticket-complete
description: GitHub Issue 를 end-to-end 로 마무리합니다. 커밋, 푸시, Closes #N 포함 PR 생성, 머지, worktree/브랜치 정리, 구현 결과 comment 게시. 모든 acceptance criteria 가 충족된 후 사용. 프로젝트별 설정은 {project_root}/.claude/ticket-skill-configs.yaml 에서 읽습니다.
---

# Ticket Complete

GitHub Issue 구현 완료 후 전체 마무리 프로세스를 한 번에 실행합니다.
커밋 → PR 생성 (Closes #N) → PR 머지 → worktree/브랜치 정리 → 구현 결과 comment → Issue 라벨 정리 (PR 머지로 자동 close) → wiki 발행(push).

## Bootstrap (모든 호출 시 첫 단계)

`{project_root}/.claude/ticket-skill-configs.yaml` (+ 로컬 `ticket-skill-configs.local.yaml` override, 있으면) 로드·병합 → minimal 4 키 검증 → **계정 라우팅** → `${var}` 치환. 상세: `~/.claude/skills/ticket-plan/CONFIG.md`.

**계정 라우팅** (멀티 계정 — 옵셔널): `project.gh_account` 가 있으면 PR/커밋/push 전에 `gh auth switch --user ${project.gh_account}` 실행. `project.git_email` 이 있으면 커밋 전 `git config user.email ${project.git_email}` (로컬). `${wiki.repo_url}` 은 `git@${project.ssh_host}:...` 로 유도 (ssh_host 기본 `github.com`) — wiki push 가 올바른 SSH 키 사용. main repo push 는 `origin` remote URL 을 따름. 세 키 모두 없으면 단일 계정 동작.

## 데이터 소스

- **티켓**: GitHub Issues (`${project.repo}`)
- **PR ↔ Issue 연동**: PR body 의 `Closes #{N}` 으로 머지 시 Issue 자동 close
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
- **Docs-only 변경** (글로벌 스킬, 외부 문서 등 worktree·PR 불필요): Step 1 의 Issue 확인 후 **Step 2~8 스킵**. Step 9 (라벨 정리 / 체크리스트 / 구현 결과 comment / close), Step 10 (wiki 발행) 만 수행. PR/머지가 없으므로 Issue 자동 close 도 발생하지 않음 → Step 9 에서 `gh issue close {N}` 를 명시적으로 호출

## 사용 절차

### Step 1: 상태 확인

1. **Issue 확인**: `gh issue view {N} --repo "${project.repo}" --json title,body,labels,state,url,comments`
2. **Worktree 확인**: `git worktree list` 로 `${worktree.prefix}-{N}-{slug}` 존재 여부
3. **현재 위치 확인**: `git branch --show-current`

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
git push -u origin {N}-{slug}
```

### Step 4: PR 생성

```bash
gh pr create --repo "${project.repo}" --base ${project.default_branch} --title "{N}-{Short description}" --body "$(cat <<'EOF'
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
EOF
)"
```

**필수**:
- `--base ${project.default_branch}`
- 본문에 `Closes #{N}` 한 줄 — 머지 시 Issue 자동 close
- PR 제목: `{N}-{Short description}`

### Step 5: PR 머지

**중요: 사용자 확인 필수**
PR 머지 전 사용자에게 빌드/테스트 결과 확인 여부를 묻습니다. "머지해", "테스트 완료", "확인 완료" 등 명시 승인 없으면 머지 금지.

```bash
gh pr merge {pr-number} --repo "${project.repo}" --squash
```

머지 성공 시 Issue `#{N}` 은 GitHub 가 자동 close 처리. 추가 `gh issue close` 불필요.

### Step 6: Main Repo 이동 및 Pull

```bash
cd ${project_root}
git checkout ${project.default_branch}
git pull
```

### Step 7: Worktree 정리

```bash
git worktree remove ../${worktree.prefix}-{N}-{slug}
```

- 실패 시 (미커밋 변경 등) 사용자 보고 후 `--force` 옵션 사용 여부 확인
- **`rm -rf` 금지** — symlink 가 있을 경우 원본 파일이 삭제될 수 있음

### Step 8: 브랜치 정리

```bash
git branch -d {N}-{slug}
git push origin --delete {N}-{slug}
```

이미 삭제됐으면 스킵.

### Step 9: Issue 완료 처리

PR 머지로 Issue 가 이미 closed 상태이지만, 추가로 다음을 수행:

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
- PR #{PR번호} 머지 완료: {PR URL}
- Issue 자동 close 완료
- Worktree 정리 완료
- 브랜치 정리 완료 (local + remote)
- 구현 결과 comment 추가 완료
- Follow-up: {새 Issue N건 생성: #X, #Y / 기록만 / 없음}
- Wiki: {N개 드래프트 발행(push) 완료 / 발행할 드래프트 없음}
```

## 오류 처리

- **PR 머지 실패**: CI 체크 실패 또는 conflict → 원인 안내
- **worktree 제거 실패**: 미커밋 변경 → 사용자 확인 후 `--force`
- **브랜치 삭제 실패**: 이미 삭제됨 → 스킵
- **Issue 자동 close 실패**: PR body 의 `Closes #{N}` 누락 가능 → 수동 `gh issue close {N} --repo "${project.repo}"`
- **Wiki push 실패**: 충돌 시 `git pull --rebase` 후 재시도

## 주요 참고 사항

- **Base 브랜치**: `${project.default_branch}`
- **Symlink 주의**: worktree 내 symlink 존재 가능. `rm -rf` 금지, `git worktree remove` 만 사용
- **PR body 의 `Closes #{N}` 필수**: 누락 시 Issue 가 자동으로 close 되지 않음
- **후속 작업 처리**: 사용자가 `new` 로 명시한 항목만 새 Issue. 다른 Issue 자동 수정·전파 금지
- **Wiki 와 main repo 는 별도 git repo**: wiki 변경은 `${wiki.clone_path}` 에서 별도로 commit/push
