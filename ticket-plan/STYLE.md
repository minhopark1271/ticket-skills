# 산출물 작성 원칙

`ticket-plan` · `ticket-implement` · `ticket-complete` · `ticket-doc` 가 생성하는
산출물 (Issue body, Issue comment, PR body, Wiki 페이지) 의 공통 작성 원칙.

## 원칙

1. **최소 + 충분** — 정보 전달에 필요한 만큼만. 미사여구 · 강조 · 연결 표현 제거
2. **한 문서 내 중복 금지** — 같은 문서에서 같은 정보를 두 번 적지 않음.
   단, 가독성에 명확히 기여하는 경우는 예외 (예: 표 머리말 ↔ 본문, 요약 ↔ 상세).
   다른 문서 간 중복은 제한 없음
3. **실재 용어** — 코드·시스템·도메인에 실재하는 이름 (함수 · 클래스 · 필드 · 파일 ·
   모듈 · 표준 개념) 으로 지칭. 검증 불가능한 추상어 (`구조적 일관성`, `프로세스 최적화`,
   `유연성 확보`, `통합 추상화 계층` 등 LLM 풍 가짜 용어) 금지 — 무엇이 어떻게
   바뀌는지 점검할 수 없는 표현은 정보가 아님
4. **빈 섹션 생략** — 내용이 없으면 헤더 자체 삭제. placeholder (`없음`, `해당 없음`, `TBD`) 도 금지
5. **구조 우선** — 표 · bullet · 코드 블록으로 정보 정렬. 산문은 의미 부여가 필요할 때만
6. **약어 첫 등장 풀어쓰기** — 약어 · 내부 용어가 처음 등장할 때 `Open Interest (OI)` 형식으로 풀어쓰고, 이후 동일 문서에서는 약어만 사용

## 안티 패턴

| 회피 | 대체 |
|---|---|
| "다음과 같습니다", "아래와 같이", "위에서 설명한 것처럼" | 바로 본문 |
| "~할 수 있도록", "~하기 위해서" | 동사 · 명사형 |
| "구조적 일관성", "프로세스 최적화", "통합 추상화 계층" 류 가짜 용어 | 실재 이름 (예: `validate_input() 의 분기 정리`, `A·B 처리를 합친 함수`) |
| 라벨 + 본문에 동일 정보 (`priority:high` + "긴급함") | 라벨만 |
| "성공적으로 ~ 완료되었습니다" | "완료" |

## History · 세션 서명

모든 산출물에 **머신 · 세션 ID** 를 남겨 나중에 transcript 로 되짚기 위한 기록. GitHub 은 작성자 · 시각만 자동 기록한다.

### Issue body · Wiki 페이지 — `## History` 누적

계속 고쳐지는 문서라 매 편집이 기록 대상.

```markdown
## History

- `2026-08-12T14:32+09:00` · created · ticket-plan · MinhoWindowPC · `6204fbb6-3a3f-4494-8ed6-aedba0a33410`
- `2026-08-12T15:10+09:00` · edited · ticket-doc · MinhoWindowPC · `df10f8c9-7204-4eca-86aa-d8fe0a0b0fa4`
- `2026-08-12T16:04+09:00` · edited · ticket-doc · MinhoWindowPC · `df10f8c9-7204-4eca-86aa-d8fe0a0b0fa4` — 3절 표 · 엣지 재측정
```

필드 `시각 · 행위 · 스킬 · 머신 · 세션 ID`. 행위는 `created` (최초) / `edited` (이후 전부).

**세션 ID 뒤는 비워라.** 무엇을 왜 고쳤는지는 commit 메시지 · transcript 가 담당한다. 남겨야 할 때만 `— ` 뒤에 **명사구 3개 이하** (`·` 구분, 도합 40자 내외). 문장 · 판정 · 근거 · 수치 서술 금지.

| 금지 | 허용 |
|---|---|
| `— 부록 A 신설: 원본 구조도의 담당자 배정을 빼고 기능 구성만 텍스트 도식 2종으로 이관 (A-1 와이드 214열 / A-2 분할 139+75열). 폭 모호 문자를 전면 배제하고 한글 2열 폭으로 격자 정렬 검증` | `— 부록 A 신설 · 도식 2종` |
| `— 1-1절·5절 코드블록의 폭 모호 문자 38줄 치환(-> <-> * ! v 로)` | `— 코드블록 문자 치환` |

```bash
printf -- '- `%s` · %s · %s · %s · `%s`\n' \
  "$(date +%Y-%m-%dT%H:%M%:z)" edited ticket-doc "$(hostname)" "${CLAUDE_CODE_SESSION_ID:?}"
```

- **내용이 바뀌는 시점마다 한 줄** — Issue 는 `gh issue edit --body-file` 로 쓸 때, Wiki 는 파일을 고쳐 commit 하기 전
- Wiki push (발행) 는 내용 변경이 아님 — 줄 추가 안 함
- 기존 줄 보존, **아래에 append**. 수정 · 삭제 금지

### Issue comment · PR body — 한 줄 footer

1회성 발화라 누적될 이력이 없다. 필드 `스킬 · 머신 · 세션 ID` — 시각 · 작성자는 GitHub 이 붙이므로 뺀다.

```markdown
---
<sub>ticket-implement · MinhoWindowPC · `6204fbb6-3a3f-4494-8ed6-aedba0a33410`</sub>
```

```bash
printf -- '\n---\n<sub>%s · %s · `%s`</sub>\n' \
  ticket-implement "$(hostname)" "${CLAUDE_CODE_SESSION_ID:?}"
```

- PR body 는 기존 `🤖 Generated with [Claude Code]` 줄 **아래**
- 나중에 고쳐 세션이 달라지면 교체하지 말고 아래에 한 줄 추가

### 공통

- **세션 ID 는 full UUID** — `claude --resume {uuid}` 로 그 세션을 복원할 수 있는 유일한 형태. 축약 금지
- `CLAUDE_CODE_SESSION_ID` 미설정 시 **중단 · 보고**. 대체값 (`unknown` 등) 금지 — env 이름 변경을 즉시 인지해야 함
- 머신은 양쪽 셸 모두 `hostname` (`$env:COMPUTERNAME` 은 전 대문자라 어긋남). PowerShell 시각은 `Get-Date -Format "yyyy-MM-ddTHH:mmzzz"`
- Issue 브리핑 · 요약 시 History · footer 는 발췌 제외 (기계 기록)
