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

```bash
# 1. Clone
git clone git@github.com:minhopark1271/claude-ticket-skills.git ~/Projects/claude-ticket-skills

# 2. ~/.claude/skills/ 에 심볼릭 링크 생성
mkdir -p ~/.claude/skills
for d in ~/Projects/claude-ticket-skills/ticket-*; do
  ln -s "$d" ~/.claude/skills/
done

# 3. 확인
ls -la ~/.claude/skills/ | grep ticket
```

## 동기화

수정 후 다른 PC 에 반영:

```bash
cd ~/Projects/claude-ticket-skills
git add . && git commit -m "update: ..."
git push

# 다른 PC 에서
git pull
```
