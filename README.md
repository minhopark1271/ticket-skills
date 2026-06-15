# claude-ticket-skills

Claude Code 글로벌 스킬 모음 — GitHub Issues + Wiki 기반의 티켓 워크플로우.

## 구성 스킬

| 스킬 | 용도 |
|---|---|
| `ticket-plan` | GitHub Issue 생성, worktree 자동 생성 |
| `ticket-implement` | Issue end-to-end 구현 파이프라인 |
| `ticket-doc` | GitHub Wiki 기술 문서 작성·관리 |
| `ticket-review` | 코드 3축 (기능/문법/품질) 셀프 리뷰 |
| `ticket-complete` | 커밋 → PR (Closes #N) → 머지 → worktree 정리 → Implementation Results comment |

## 프로젝트별 설정

각 프로젝트 루트의 `.claude/ticket-skill-configs.yaml` 에서 minimal 4 키 정의:

```yaml
project:
  repo: owner/name          # GitHub repo (예: minhopark1271/asdf)
  default_branch: master
worktree:
  prefix: ../{repo-name}-   # worktree 디렉토리 prefix
wiki:
  clone_path: ../{repo}.wiki/
```

상세: `ticket-plan/CONFIG.md`

## 설치 (새 PC)

각 `ticket-*` 디렉토리를 `~/.claude/skills/` 에 링크한다. 원본은 repo 에 두고 링크만 거므로 `git pull` 하면 스킬도 자동 반영된다.

### macOS / Linux (symlink)

```bash
# 1. Clone — 두 org 에 동일 미러가 있다. 둘 중 하나 선택:
git clone git@github-numereal:NumeRealWorks/claude-ticket-skills.git ~/Projects/claude-ticket-skills
# 또는 NohLabWorks (기본 github.com 키 사용):
# git clone git@github.com:NohLabWorks/claude-ticket-skills.git ~/Projects/claude-ticket-skills

# 2. ~/.claude/skills/ 에 심볼릭 링크 생성
mkdir -p ~/.claude/skills
for d in ~/Projects/claude-ticket-skills/ticket-*; do
  ln -s "$d" ~/.claude/skills/
done

# 3. 확인
ls -la ~/.claude/skills/ | grep ticket
```

### Windows (PowerShell, junction)

Windows 의 심볼릭 링크 생성은 관리자 권한 또는 개발자 모드가 필요하다. 권한 없이 동일하게 동작하는 **디렉토리 junction** 을 사용한다 (읽기·쓰기 모두 원본에 그대로 적용됨).

```powershell
# 1. Clone — 두 org 에 동일 미러가 있다. 둘 중 하나 선택:
git clone git@github-numereal:NumeRealWorks/claude-ticket-skills.git "$HOME\Projects\claude-ticket-skills"
# 또는 NohLabWorks (기본 github.com 키 사용):
# git clone git@github.com:NohLabWorks/claude-ticket-skills.git "$HOME\Projects\claude-ticket-skills"

# 2. ~/.claude/skills/ 에 junction 생성
$src = "$HOME\Projects\claude-ticket-skills"
$dst = "$HOME\.claude\skills"
New-Item -ItemType Directory -Force $dst | Out-Null
foreach ($d in Get-ChildItem -Path $src -Directory -Filter "ticket-*") {
  $link = Join-Path $dst $d.Name
  if (Test-Path $link) { Remove-Item $link -Recurse -Force }
  New-Item -ItemType Junction -Path $link -Target $d.FullName | Out-Null
}

# 3. 확인
Get-ChildItem $dst | Select-Object Name, LinkType, Target | Format-Table -AutoSize
```

> 관리자 권한 또는 개발자 모드가 켜져 있다면 junction 대신 symlink 도 가능하다:
> `New-Item -ItemType SymbolicLink -Path $link -Target $d.FullName`

설치 후 스킬을 인식시키려면 Claude Code 를 재시작한다.

## 동기화

이 repo 는 **두 org 에 동일 미러**로 관리된다 (`NumeRealWorks`, `NohLabWorks`).
`origin` 의 push URL 이 양쪽 모두로 설정돼 있어 `git push` 한 번이면 두 곳이 동기화된다.

```bash
cd ~/Projects/claude-ticket-skills
git add . && git commit -m "update: ..."
git push            # origin → NumeRealWorks + NohLabWorks 동시 push
git push numereal   # NumeRealWorks 만
git push nohlab     # NohLabWorks 만

# 다른 PC 에서
git pull            # origin(=NumeRealWorks) 에서 fetch
```

| remote | URL | 계정 / SSH 키 |
|---|---|---|
| `origin` (fetch) | `git@github-numereal:NumeRealWorks/claude-ticket-skills.git` | minhopark-lgtm / `id_rsa_numereal` |
| `origin` (push) | NumeRealWorks + NohLabWorks 양쪽 | — |
| `numereal` | `git@github-numereal:NumeRealWorks/…` | minhopark-lgtm / `id_rsa_numereal` |
| `nohlab` | `git@github.com:NohLabWorks/…` | minhopark1271 / `id_rsa` |

> `github-numereal` 은 `~/.ssh/config` 의 Host alias 다. 새 PC 에서는 해당 alias 와 `IdentityFile ~/.ssh/id_rsa_numereal` 등록이 선행돼야 NumeRealWorks 접근이 된다.
