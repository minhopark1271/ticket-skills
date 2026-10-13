---
name: ticket-checklist
description: ticket-review 의 CHECKLIST.md 항목만으로 코드를 점검하고 위반을 직접 수정합니다. 기능·문법 축 검토와 리뷰 의견 작성은 하지 않음. "체크리스트 점검", "체크리스트대로 고쳐줘", "ticket-checklist" 요청 시 사용. 인자 = 파일 / 디렉터리, 없으면 현재 브랜치 변경분.
---

# Ticket Checklist

[CHECKLIST.md](../ticket-review/CHECKLIST.md) (`~/.claude/skills/ticket-review/CHECKLIST.md`) 를 유일한 기준으로 점검하고 위반을 고친다. ticket-review 와 기준을 공유하므로 체크리스트는 그 파일 하나만 수정한다.

## 범위

1. **인자 있음**: 지정한 파일 / 디렉터리 전체
2. **인자 없음**: 현재 worktree 의 `git diff $(git merge-base HEAD origin/${default_branch})` + untracked. `${default_branch}` 는 `{project_root}/.claude/ticket-skill-configs.yaml` 의 `project.default_branch`, config 가 없으면 `origin/HEAD`
3. 범위가 비면 대상을 묻는다

변경분 모드에서는 변경된 hunk 와 그 hunk 가 속한 함수·클래스만 점검한다. 단 "같은 형태의 누락은 전수 확인" 같은 항목은 항목 문구가 정한 범위를 따른다.

## 실행 단계

1. **로드** — CHECKLIST.md 를 읽는다. 매 호출 새로 읽고 항목을 기억에서 재구성하지 않는다
2. **점검** — 대상 파일마다 전 항목을 확인해 위반 수집 (`파일:라인`, 항목, `→` 배정 블록)
3. **수정** — 배정 블록별로 처리
   - 결함 · 기타 수정사항 → 수정
   - 검토 필요 사항 → 수정이 하나로 정해지고 동작이 바뀌지 않으면 수정. 아니면 수정하지 않고 보고
   - 참고 → 수정하지 않고 보고
4. **검증** — 프로젝트 CLAUDE.md 의 빌드·테스트 명령 실행. 실패하면 원인이 이번 수정이면 고치고, 기존 실패면 보고
5. **보고** — 아래 양식으로 출력하고 종료

## 수정 규칙

- 체크리스트에 없는 문제는 고치지 않는다. 눈에 띄면 보고의 "범위 밖" 에 한 줄
- rename 은 Serena `rename_symbol` 로 참조까지 함께 바꾼다. 파일 rename 은 `git mv` + 빌드 타깃명 갱신
- 동작이 바뀌는 수정 (null 체크 추가의 early return 등) 은 체크리스트가 처리 방식을 정한 경우에만 그대로 적용
- commit · push 금지. 사용자가 요청할 때만

## 보고 양식

```
## 체크리스트 점검 — {범위}

### 수정
- `파일:라인` — 항목 — 변경 내용

### 미수정
- `파일:라인` — 항목 — 수정하지 않은 이유 (판단 필요 / 참고)

### 범위 밖
- `파일:라인` — 체크리스트 외 문제

### 검증
- {명령} — 통과 / 실패 (출력 요약)
```

내용 없는 블록은 제목째 생략. 위반이 없으면 "위반 없음" 한 줄. 개수를 채우지 않는다.
