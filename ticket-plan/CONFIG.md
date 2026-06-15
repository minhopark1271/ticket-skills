# ticket-* Skills Config Schema

5개 ticket-* 스킬 (`ticket-plan`, `ticket-implement`, `ticket-complete`, `ticket-doc`, `ticket-review`) 이 공유하는 프로젝트별 설정 파일의 스키마 정의.

## 파일 위치

```
{project_root}/.claude/ticket-skill-configs.yaml
```

- **커밋 대상**: 팀이 공유 (git 에 포함)
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

## Universal Defaults (config 에 적지 않음)

아래 값은 minimal 4 키에서 유도되거나 모든 프로젝트에 공통. **config 에 적지 않는다.**

| 항목 | 유도/기본값 |
|---|---|
| `wiki.repo_url` | `git@github.com:${project.repo}.wiki.git` |
| `wiki.url_prefix` | `https://github.com/${project.repo}/wiki/` |
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

## 변수 치환 — `${var}` 참조 표

스킬 SKILL.md 안에서 사용하는 placeholder. Bootstrap 단계에서 config 값으로 치환.

| Placeholder | 출처 | 예시 |
|---|---|---|
| `${project.repo}` | config | `minhopark1271/asdf` |
| `${project.default_branch}` | config | `master` |
| `${worktree.prefix}` | config | `asdf` |
| `${wiki.clone_path}` | config | `../asdf.wiki` |
| `${wiki.repo_url}` | universal default | `git@github.com:minhopark1271/asdf.wiki.git` |
| `${wiki.url_prefix}` | universal default | `https://github.com/minhopark1271/asdf/wiki/` |
| `${project_root}` | **메타 변수** (Bootstrap 1번 결과 `git rev-parse --show-toplevel`) | `/Users/minhopark/Projects/asdf` |

## Bootstrap 절차 (5개 스킬 공통)

각 SKILL.md 상단에 동일한 Bootstrap 섹션을 포함. 스킬 실행 시 LLM 이 다음 순서로 처리:

1. **프로젝트 루트 감지**
   ```bash
   git rev-parse --show-toplevel
   ```
   실패 시 현재 cwd 사용
2. **Config 로드**: `{project_root}/.claude/ticket-skill-configs.yaml`
   - 없으면 에러 보고 후 중단 — 사용자에게 minimal 4 키 작성 안내
3. **Branch override merge** (옵셔널)
   - 현재 브랜치 = `git branch --show-current`
   - `branches.{현재브랜치}` 가 있으면 해당 키만 덮어쓰기
4. **필수 키 검증**
   - `project.repo`, `project.default_branch`, `worktree.prefix`, `wiki.clone_path` 존재 확인
   - 누락 시 에러 보고
5. **`${var}` 치환 self-check**
   - SKILL.md 본문의 모든 `${...}` 가 위 표의 키 또는 universal default 로 매핑 가능한지 확인
   - 매핑 불가 placeholder 있으면 보고 후 중단

## Lazy Bootstrap (스킬별 첫 호출 시)

별도 setup 스크립트 없이, 각 스킬이 첫 호출 시 필요한 외부 리소스를 inline 으로 준비:

| 스킬 | Lazy 액션 |
|---|---|
| `ticket-plan` | `gh label create --force` 로 universal 라벨 12개 보장 (이미 있으면 no-op) |
| `ticket-doc` | `${wiki.clone_path}` 디렉토리 없으면 `git clone ${wiki.repo_url} ${wiki.clone_path}` |
| 그 외 | 별도 lazy 액션 없음 |

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

## Dry-run 검증

```bash
# 1. config 파일 존재
test -f .claude/ticket-skill-configs.yaml || echo "missing config"

# 2. SKILL.md 의 ${var} 잔존이 모두 위 표에 정의되어 있는지
grep -oE '\$\{[a-z_.]+\}' ~/.claude/skills/ticket-*/SKILL.md | sort -u

# 3. project.repo 라우팅 확인
gh issue list --repo "$(yq '.project.repo' .claude/ticket-skill-configs.yaml)" --limit 1
```
