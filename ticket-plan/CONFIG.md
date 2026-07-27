# ticket-* Skills Config Schema

5개 ticket-* 스킬 (`ticket-plan`, `ticket-implement`, `ticket-complete`, `ticket-doc`, `ticket-review`) 이 공유하는 프로젝트별 설정 파일의 스키마 정의.

## 파일 위치

```
{project_root}/.claude/ticket-skill-configs.yaml        # 공유 — 팀 공통(커밋 대상)
{project_root}/.claude/ticket-skill-configs.local.yaml  # 개인 — 머신/계정별(gitignore, 옵셔널)
```

- **공유 config** (`ticket-skill-configs.yaml`): 모든 팀원에게 **동일하게 유효한 범용 키만**. git 에 커밋해 공유.
- **로컬 override** (`ticket-skill-configs.local.yaml`): 개인·머신별 키(gh 계정·SSH host·커밋 이메일). 프로젝트 `.gitignore` 에 추가하고 **커밋하지 않음**. 없으면 무시(단일 계정 사용자는 불필요).
- **병합 규칙**: Bootstrap 이 공유 → 로컬 순으로 로드하고 **로컬이 공유를 덮어씀**(deep merge).
- **철칙**: 특정 사람에게만 맞는 값(개인 gh 계정명·`~/.ssh/config` alias·개인 이메일)은 공유 config 에 **절대 넣지 않는다** — 전부 로컬 override.
- **프로젝트 루트 감지**: `git rev-parse --show-toplevel` 결과 또는 현재 cwd

## Minimal Schema (4 키 — 출발점)

```yaml
version: 1
project:
  repo: <owner>/<repo>        # GitHub 풀 슬러그 (예: minhopark1271/asdf)
  default_branch: <branch>    # 기본 브랜치 (예: master / main)
worktree:
  prefix: <short-name>        # worktree 디렉토리 접두 (예: asdf, licensing)
wiki:
  clone_path: <relative-path> # main repo 의 sibling 경로 (예: ../asdf.wiki)
```

| 키 | 필수 | 타입 | 설명 |
|---|---|---|---|
| `version` | O | int | 스키마 버전. 현재 `1` 만 지원. 호환성 깨지는 변경 시 증가 |
| `project.repo` | O | string | `<owner>/<repo>` 형식. `gh` 명령의 repo 컨텍스트로 사용 |
| `project.default_branch` | O | string | PR base, worktree base 로 사용 |
| `worktree.prefix` | O | string | `{prefix}-{N}-{slug}` 디렉토리/브랜치 prefix |
| `wiki.clone_path` | O | string | wiki repo 로컬 클론 경로 (main repo sibling) |
| `project.gh_account` | X | string | (멀티 계정) **로컬 override 전용** — 활성화할 `gh` 계정명 → `gh auth switch --user <값>`. 생략 시 현재 active 계정 |
| `project.ssh_host` | X | string | (멀티 계정) **로컬 override 전용** — git/wiki SSH host alias (예: `github-numereal`). 생략 시 `github.com`. 머신 전역 `insteadOf` 로 대체 가능(아래) |
| `project.git_email` | X | string | (멀티 계정) **로컬 override 전용** — 로컬 커밋 author email → `git config user.email <값>`(로컬). 생략 시 전역 설정 |

> `gh_account`/`ssh_host`/`git_email` 세 키는 **사람·머신마다 다른 값**이므로 공유 config 가 아니라 `ticket-skill-configs.local.yaml`(gitignore)에만 적는다. 상세는 아래 "멀티 GitHub 계정" 절.

## Universal Defaults (config 에 적지 않음)

아래 값은 minimal 4 키에서 유도되거나 모든 프로젝트에 공통. **config 에 적지 않는다.**

| 항목 | 유도/기본값 |
|---|---|
| `project.ssh_host` | `github.com` (config 에 없을 때) |
| `wiki.repo_url` | `git@${project.ssh_host}:${project.repo}.wiki.git` (ssh_host 기본 `github.com`) |
| `wiki.url_prefix` | `https://github.com/${project.repo}/wiki/` (웹 브라우즈 URL — 항상 `github.com`) |
| `branch_pattern` | `{N}-{slug}` |
| `worktree_pattern` | `${worktree.prefix}-{N}-{slug}` |
| `pr.title_pattern` | `{N}-{Short description}` |
| `pr.close_keyword` | `Closes #{N}` |
| `pr.co_author` | `Claude Opus 4.7 (1M context) <noreply@anthropic.com>` |
| **Universal labels** (lazy 생성) | `~/.claude/skills/ticket-plan/SKILL.md` 의 "지연 초기화" 섹션 참조 |

## Optional — Branch Override

특정 브랜치에 대해서만 minimal 키를 덮어쓸 때 사용. 보통 비워둠.

```yaml
branches:
  release/v2:
    default_branch: release/v2
  develop:
    default_branch: develop
```

Bootstrap 단계에서 `git branch --show-current` 가 `branches.{name}` 키와 일치하면 **선택적으로** merge.

## Optional — 멀티 GitHub 계정 / SSH 호스트 (로컬 override)

여러 GitHub 계정을 한 머신에서 쓰고 **repo 마다 다른 계정**으로 작업할 때 사용. 이 값들은 **머신·사람마다 다르므로 공유 config 에 넣지 않고**, 반드시 **로컬 override 파일**(`ticket-skill-configs.local.yaml`, gitignore)에 둔다. 단일 계정 사용자는 파일 자체가 불필요.

```yaml
# {project_root}/.claude/ticket-skill-configs.local.yaml   ← 커밋 금지(.gitignore)
project:
  gh_account: minhopark-lgtm        # gh auth switch 대상 (gh issue/pr/label 권한)
  ssh_host: github-numereal         # ~/.ssh/config 의 Host alias (git/wiki push 키 선택)
  git_email: minhopark@numereal.ai  # 이 repo 커밋 author (로컬)
```

**전제 조건 (1회 셋업, config 와 무관)**:
1. `gh auth login` 으로 해당 계정들 `gh` 에 로그인 (`gh auth status` 로 확인)
2. `~/.ssh/config` 에 계정별 Host alias + IdentityFile 등록 (예: `Host github-numereal` → `IdentityFile ~/.ssh/id_rsa_numereal`)
3. 해당 repo 의 `origin` remote 가 alias 를 쓰도록 설정 — `git remote set-url origin git@${ssh_host}:<owner>/<repo>.git`

**동작** (공유+로컬 병합된 config 기준):
- `gh_account` → Bootstrap 이 `gh` 명령 전에 `gh auth switch --user ${project.gh_account}`. 전역 active 계정을 바꾸는 side effect 이므로 repo 이동 시 각 Bootstrap 이 다시 전환.
- `ssh_host` → `${wiki.repo_url}` 이 `git@${ssh_host}:...` 로 유도되어 wiki clone/push 가 올바른 SSH 키 사용. main repo push 는 `origin` URL 을 따름(전제조건 3).
- `git_email` → Bootstrap 이 `git config user.email ${project.git_email}`(로컬, `--global` 아님) 로 커밋 author 를 맞춤.

**`ssh_host` 대안 — `insteadOf` (권장, config 불요)**: `ssh_host` 를 로컬 파일에 두는 대신, **머신 전역 git 설정**으로 특정 org 의 표준 URL 을 alias 로 투명 라우팅하면 config 에 아무것도 안 적어도 된다:
```bash
git config --global url."git@<alias>:<Org>/".insteadOf "git@github.com:<Org>/"
```
이러면 공유 config 의 기본 유도값(`git@github.com:<Org>/...`)이 그대로 alias 키를 타므로 **공유 config 를 완전 범용**으로 유지한다. (gh 계정 전환·커밋 이메일은 `insteadOf` 로 안 되므로, 필요하면 `gh_account`/`git_email` 만 로컬에 둔다.)

## Optional — GitHub Projects 보드 (Status 동기화)

이슈가 GitHub Projects v2 보드에 올라가 있고, **ticket-* 생명주기 단계에 따라 카드의 Status(레인)를 자동 이동**시키고 싶을 때 사용. 없으면 모든 보드 동기화를 건너뛴다(보드 미사용 프로젝트 무영향).

config 에 적는 건 **`owner` + `number` 두 줄만**. 나머지(`project_id`·`status_field_id`·`status_options`)는 첫 보드 동기화 때 `Set-BoardStatus` 가 자동 해석·캐시한다(아래 "Board 좌표 lazy 해석"). 보드별 값은 **프로젝트 config** 에, 해석 로직은 **global 스킬(본 문서)** 에 둔다.

```yaml
board:
  owner: <org-or-user>      # 프로젝트 소유 org/user (예: NohLabWorks) — 수기 필수
  number: <project-number>  # 프로젝트 번호 (URL .../projects/<number>) — 수기 필수
  # ↓ 첫 호출 시 자동 해석·캐시(수기 작성 불요). 보드 재생성으로 ID 가 바뀌면 지우면 재해석.
  # project_id: <PVT_...>
  # status_field_id: <PVTSSF_...>
  # status_options: { backlog, ready, in_progress, in_review, done → 옵션 id }
```

| 키 | 작성 | 설명 |
|---|---|---|
| `board.owner` / `board.number` | **수기 필수**(보드 사용 시) | `gh project` 프로젝트 지정 |
| `board.project_id` | 자동 캐시 | `item-edit` 의 `--project-id` |
| `board.status_field_id` | 자동 캐시 | Status 단일선택 필드 id |
| `board.status_options.<key>` | 자동 캐시 | 논리키(backlog/ready/in_progress/in_review/done) → 옵션 id |

### 생명주기 → Status 매핑 (스킬 직접 구동)

라벨이 아니라 **스킬 단계**가 레인을 움직인다(GitHub 기본 워크플로엔 "라벨 트리거" 가 없고, 초안/리뷰완료는 라벨로 구분 불가하기 때문). 각 스킬이 해당 시점에 아래 `Set-BoardStatus` 를 호출:

| 스킬 / 시점 | status_key |
|---|---|
| `ticket-plan` Issue 생성 직후 | `backlog` |
| `ticket-plan` 리뷰(R4~R5) 반영 완료 | `ready` |
| `ticket-implement` 작업 시작(Type A) | `in_progress` |
| `ticket-review` 검토 시작 | `in_review` |
| `ticket-complete` 머지/완료 처리 | `done` |

### Set-BoardStatus(N, status_key) 절차

0. `config.board` 없으면 **즉시 종료**(보드 미설정 — skip).
0b. `board.project_id` / `status_field_id` / `status_options` 중 하나라도 누락 → **Board 좌표 lazy 해석**(아래)을 먼저 수행해 config 에 캐시한 뒤 진행.
1. 보드 내 item id 조회:
   ```bash
   gh project item-list ${board.number} --owner ${board.owner} --format json --limit 300 \
     --jq ".items[] | select(.content.number==N and .content.repository==\"${project.repo}\") | .id"
   ```
   - 결과가 비어 있으면(보드 미등록) 추가 후 그 id 사용:
     ```bash
     gh project item-add ${board.number} --owner ${board.owner} \
       --url https://github.com/${project.repo}/issues/N --format json --jq .id
     ```
2. Status 설정:
   ```bash
   gh project item-edit --id <item_id> --project-id ${board.project_id} \
     --field-id ${board.status_field_id} --single-select-option-id <status_options[status_key]>
   ```
3. 실패(토큰 `project` 스코프 부족 등)는 **경고만 남기고 본 작업을 막지 않는다** — 보드 동기화는 보조 기능.

> 전제: `gh` 토큰에 `project` 스코프 필요 (`gh auth refresh -s project`). 멀티 계정이면 라우팅된 계정 토큰 기준.

### Board 좌표 lazy 해석 (1회 자동 캐시)

`board.owner` + `number` 만 있고 ID 가 미캐시일 때, `Set-BoardStatus` 가 처음 호출되는 스킬(보통 ticket-plan)에서 1회 수행:

1. `project_id` ← `gh project view ${board.number} --owner ${board.owner} --format json --jq .id`
2. Status 필드/옵션 ← `gh project field-list ${board.number} --owner ${board.owner} --format json` 에서
   `type=="ProjectV2SingleSelectField"` 이고 `name=="Status"` 인 필드의 `id`(= status_field_id)와 `options[]`(name→id).
3. 레인명 → 논리키 매핑(대소문자·공백 무시): `Backlog→backlog`, `Ready→ready`, `In progress→in_progress`, `In review→in_review`, `Done→done`. 매칭 안 되는 레인명이 있으면 그 키는 비워두고 사용자에게 보고(비표준 레인명 프로젝트).
4. 위 값을 **프로젝트 config**(`.claude/ticket-skill-configs.yaml`)의 `board` 블록에 Edit 로 써넣어 캐시. 이후 호출은 캐시 사용(재해석 X).

> 일반화된 해석 로직은 global 스킬(본 CONFIG.md)에 있고, 해석 결과(프로젝트별 값)는 프로젝트 config 에 캐시된다.

## 변수 치환 — `${var}` 참조 표

스킬 SKILL.md 안에서 사용하는 placeholder. Bootstrap 단계에서 config 값으로 치환.

| Placeholder | 출처 | 예시 |
|---|---|---|
| `${project.repo}` | config | `minhopark1271/asdf` |
| `${project.default_branch}` | config | `master` |
| `${worktree.prefix}` | config | `asdf` |
| `${wiki.clone_path}` | config | `../asdf.wiki` |
| `${project.gh_account}` | config (옵셔널) | `minhopark-lgtm` |
| `${project.ssh_host}` | config (옵셔널, 기본 `github.com`) | `github-numereal` |
| `${project.git_email}` | config (옵셔널) | `minhopark@numereal.ai` |
| `${wiki.repo_url}` | universal default | `git@${project.ssh_host}:minhopark1271/asdf.wiki.git` |
| `${wiki.url_prefix}` | universal default | `https://github.com/minhopark1271/asdf/wiki/` |
| `${project_root}` | **메타 변수** (Bootstrap 1번 결과 `git rev-parse --show-toplevel`) | `/Users/minhopark/Projects/asdf` |

## Bootstrap 절차 (5개 스킬 공통)

각 SKILL.md 상단에 동일한 Bootstrap 섹션을 포함. 스킬 실행 시 LLM 이 다음 순서로 처리:

1. **프로젝트 루트 감지**
   ```bash
   git rev-parse --show-toplevel
   ```
   실패 시 현재 cwd 사용
2. **Config 로드 + 로컬 override 병합**
   - 공유: `{project_root}/.claude/ticket-skill-configs.yaml` — 없으면 에러 보고 후 중단 (minimal 4 키 작성 안내)
   - 로컬: `{project_root}/.claude/ticket-skill-configs.local.yaml` — 있으면 deep merge (**로컬 우선**). 없으면 skip
3. **Branch override merge** (옵셔널)
   - 현재 브랜치 = `git branch --show-current`
   - `branches.{현재브랜치}` 가 있으면 해당 키만 덮어쓰기
4. **필수 키 검증**
   - `project.repo`, `project.default_branch`, `worktree.prefix`, `wiki.clone_path` 존재 확인
   - 누락 시 에러 보고
5. **계정 라우팅** (멀티 계정 — 옵셔널)
   - `project.gh_account` 가 있으면: `gh auth switch --user ${project.gh_account}` 실행 (이후 모든 `gh` 명령이 해당 계정 토큰 사용). 전환 실패 시 (해당 계정 미로그인 등) 보고 후 중단 — `gh auth login` 안내
   - `project.git_email` 이 있으면: `git config user.email ${project.git_email}` (로컬). 코드 변경을 다루는 스킬에서만 필요
   - `project.ssh_host` 는 별도 명령 없이 `${wiki.repo_url}` 유도에 반영
   - 세 키 모두 없으면 이 단계 skip (단일 계정)
6. **`${var}` 치환 self-check**
   - SKILL.md 본문의 모든 `${...}` 가 위 표의 키 또는 universal default 로 매핑 가능한지 확인
   - 매핑 불가 placeholder 있으면 보고 후 중단

## Lazy Bootstrap (스킬별 첫 호출 시)

별도 setup 스크립트 없이, 각 스킬이 첫 호출 시 필요한 외부 리소스를 inline 으로 준비:

| 스킬 | Lazy 액션 |
|---|---|
| `ticket-plan` | `gh label create --force` 로 universal 라벨 12개 보장 (이미 있으면 no-op) + (`board` 있으면) 첫 `Set-BoardStatus` 호출 시 board 좌표 lazy 해석·캐시 (위 "Board 좌표 lazy 해석") |
| `ticket-doc` | `${wiki.clone_path}` 디렉토리 없으면 `git clone ${wiki.repo_url} ${wiki.clone_path}` |
| 그 외 | board 좌표가 미캐시면 `Set-BoardStatus` 가 첫 호출 시 lazy 해석 (진입점이 ticket-plan 이 아니어도 동작) |

## 확장 정책 — "한 항목씩 추가"

minimal 4 키로 출발하고, **프로젝트별로 필요해진 항목만** 사용자 요청을 받아 한 개씩 schema 에 추가.

추가 후보 (모두 처음에는 schema 외부):
- `labels.components` — 프로젝트 고유 컴포넌트 라벨 (asdf: `comp:api`, `comp:dashboard` 등)
- `sources.obsidian_public` — Mode 2 (Obsidian 추출) source 경로
- `migration.legacy_tickets` / `migration.legacy_notes` — 구버전 호환 링크
- `build_test.test` / `lint` / `typecheck` — 빌드/검증 명령 (현재는 각 프로젝트의 CLAUDE.md 참조)
- `pr.base_branch_override` — feature flag PR 등 비표준 base
- `project.visibility` — private/public

**원칙**: 추가 전에 universal default 로 해결 가능한지 먼저 검토. 정말 프로젝트별로 다를 때만 추가.

## 예제

### asdf

```yaml
version: 1
project:
  repo: minhopark1271/asdf
  default_branch: master
worktree:
  prefix: asdf
wiki:
  clone_path: ../asdf.wiki
```

### Licensing_Automation

```yaml
version: 1
project:
  repo: <owner>/Licensing_Automation
  default_branch: main
worktree:
  prefix: licensing
wiki:
  clone_path: ../Licensing_Automation.wiki
```

### 두 번째 계정 repo (멀티 계정 — 2파일 분리)

`minhopark1271` 외 `minhopark-lgtm` 계정 소속 repo. **공유엔 범용 키만, 개인 키는 로컬 override 로 분리.**

```yaml
# .claude/ticket-skill-configs.yaml   (공유·커밋)
version: 1
project:
  repo: numereal/some-repo
  default_branch: main
worktree:
  prefix: someproj
wiki:
  clone_path: ../some-repo.wiki
```

```yaml
# .claude/ticket-skill-configs.local.yaml   (개인·gitignore)
project:
  gh_account: minhopark-lgtm
  ssh_host: github-numereal          # 또는 머신 전역 insteadOf 로 대체
  git_email: minhopark@numereal.ai
```

## Dry-run 검증

```bash
# 1. config 파일 존재
test -f .claude/ticket-skill-configs.yaml || echo "missing config"

# 2. SKILL.md 의 ${var} 잔존이 모두 위 표에 정의되어 있는지
grep -oE '\$\{[a-z_.]+\}' ~/.claude/skills/ticket-*/SKILL.md | sort -u

# 3. (멀티 계정) gh_account 는 로컬 override 에 있음 → 있으면 먼저 전환
ACC=$(yq -r '.project.gh_account // ""' .claude/ticket-skill-configs.local.yaml 2>/dev/null)
[ -n "$ACC" ] && gh auth switch --user "$ACC"

# 4. project.repo 라우팅 확인 (전환된 계정 토큰으로 접근되는지)
gh issue list --repo "$(yq '.project.repo' .claude/ticket-skill-configs.yaml)" --limit 1
```
