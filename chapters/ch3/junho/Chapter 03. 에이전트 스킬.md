# 에이전트 스킬 (Agent Skills)

> Claude Code / Claude 앱에서 스킬을 만들고, 배포하고, 운영하기 위한 정리 문서
> 기준일: 2026-08-12 · Claude Code v2.1.22x 기준 · 출처는 문서 하단 참고 링크

---

## 한눈에 보기

- **스킬은 폴더다.** `SKILL.md` 하나가 최소 구성이고, 참조 문서와 스크립트를 함께 묶을 수 있다.
- **커스텀 커맨드는 스킬로 통합됐다.** `.claude/commands/deploy.md`와 `.claude/skills/deploy/SKILL.md`는 똑같이 `/deploy`를 만든다. 신규는 무조건 스킬로.
- **"커맨드형"과 "자동 발동형"의 차이는 frontmatter 두 필드다.** `disable-model-invocation`과 `user-invocable`.
- **점진적 공개가 핵심 설계다.** 메타데이터 → 본문 → 번들 파일 순으로, 필요할 때만 컨텍스트에 올라온다.
- **만드는 것보다 측정이 어렵다.** 발동하는 걸 봤다고 잘 동작하는 게 아니다. `skill-creator`로 eval을 돌리자.

---

## 목차

1. [스킬과 커스텀 커맨드](#1-스킬과-커스텀-커맨드)
2. [누가 호출하는가](#2-누가-호출하는가)
3. [스킬은 어디에 두는가](#3-스킬은-어디에-두는가)
4. [디렉터리 구조](#4-디렉터리-구조)
5. [점진적 공개 3단계](#5-점진적-공개-3단계)
6. [작성 원칙](#6-작성-원칙)
7. [출력 신뢰성 설계](#7-출력-신뢰성-설계)
8. [Opus 5에서 달라진 것](#8-opus-5에서-달라진-것)
9. [격리 실행: context fork](#9-격리-실행-context-fork)
10. [문제 해결](#10-문제-해결)
11. [최근 업데이트와 추천 기능](#11-최근-업데이트와-추천-기능)
12. [팀 배포 시 고려사항](#12-팀-배포-시-고려사항)
13. [부록: frontmatter 레퍼런스](#부록-frontmatter-레퍼런스)

---

## 1. 스킬과 커스텀 커맨드

### 둘은 이미 통합됐다

과거에는 `.claude/commands/*.md`(커맨드)와 `.claude/skills/*/SKILL.md`(스킬)가 별개 개념이었다. 지금은 병합되어 **두 위치의 파일이 동일한 `/슬래시커맨드` 인터페이스를 만든다.** 기존 `commands/` 파일은 계속 동작하지만, 신규 작성은 스킬 쪽이 권장된다.

frontmatter도 동일하게 지원된다. 차이는 스킬만 가질 수 있는 기능들이다.

| | `.claude/commands/x.md` | `.claude/skills/x/SKILL.md` |
|---|---|---|
| `/x` 호출 | O | O |
| 딸린 파일(reference, scripts, templates) | X | O |
| 중첩 디렉터리 자동 탐색 | X | O |
| `--add-dir` 디렉터리에서 로드 | X | O |
| 이름 충돌 시 | 짐 | 이김 |
| 플러그인·마켓플레이스로 패키징 | X | O |

### 결론

- **신규는 전부 `.claude/skills/`에.** 폴더 하나 더 만드는 비용밖에 차이가 없다.
- **기존 커맨드는 굳이 옮기지 않아도 된다.** 딸린 파일이 필요해지거나 자동 발동을 붙이고 싶을 때 옮기면 된다.
- 웹·Cowork까지 커버해야 하는 공용 자산은 처음부터 스킬 디렉터리로. 커맨드 파일은 업로드·패키징 단위가 아니다.

---

## 2. 누가 호출하는가

기본값은 **나도 부를 수 있고 Claude도 부를 수 있다**. 두 필드로 제한한다.

### `disable-model-invocation: true` — 나만 호출

부작용이 있거나 실행 타이밍을 통제해야 하는 워크플로우용. `/deploy`, `/commit`, `/send-slack-message` 같은 것들. 코드가 준비된 것 같다고 Claude가 알아서 배포를 결정하면 곤란하다.

부수 효과가 하나 더 있다: **스킬 목록에서 아예 빠져서 컨텍스트 비용을 줄인다.**

### `user-invocable: false` — Claude만 호출

`legacy-system-context`처럼 "옛날 시스템은 이렇게 동작한다" 같은 배경 지식용. Claude는 알아야 하지만 사용자가 그 이름을 직접 치는 건 의미 있는 행동이 아니다.

**주의:** 이 필드는 `/` 메뉴 노출만 제어하는 UI 설정이다. Claude의 자율 실행을 막지 못한다. 막으려면 `disable-model-invocation`을 써야 한다.

### 정리

| frontmatter | 내가 호출 | Claude 자동 호출 | 컨텍스트 적재 |
|---|---|---|---|
| (기본값) | O | O | description 상시, 본문은 호출 시 |
| `disable-model-invocation: true` | O | X | description도 미적재, 본문은 내가 호출할 때 |
| `user-invocable: false` | X | O | description 상시, 본문은 호출 시 |

### 발동 판단의 근거

`name`과 `description` 둘 다 트리거 판단에 쓰이지만, 실질적 무게는 `description`에 있다. 100개가 넘는 스킬 중에서 고르는 근거이기 때문이다.

**스킬이 많아지면 description이 잘린다.** 리스팅 예산은 모델 컨텍스트 윈도우의 1%이고, 넘치면 **덜 쓰는 스킬의 description부터 통째로 사라진다**(이름만 남음). 스킬 저장소 규모가 커질수록 이게 현실적인 발동 실패 원인이 된다.

- `/doctor`로 리스팅 컨텍스트 비용과 주요 기여자 확인
- `skillListingBudgetFraction` 설정으로 예산 상향 (예: `0.02` = 2%)
- 저우선순위 스킬은 `skillOverrides`에서 `name-only`로

---

## 3. 스킬은 어디에 두는가

| 위치 | 경로 | 적용 범위 |
|---|---|---|
| Enterprise | managed settings | 조직 전체 사용자 |
| Personal | `~/.claude/skills/<name>/SKILL.md` | 내 모든 프로젝트 |
| Project | `.claude/skills/<name>/SKILL.md` | 그 프로젝트만 |
| Plugin | `<plugin>/skills/<name>/SKILL.md` | 플러그인 활성화된 곳 |

**이름 충돌 우선순위:** enterprise > personal > project. 이 셋 중 아무거나 번들 스킬과 이름이 같으면 번들을 덮어쓴다(프로젝트에 `code-review`를 두면 기본 `/code-review`가 교체됨). 플러그인 스킬은 `plugin-name:skill-name` 네임스페이스라 충돌하지 않는다.

### 개인 스킬

내 머신에만 있고 모든 프로젝트에서 쓰인다. 커맨드 이름은 **디렉터리명**에서 나온다. frontmatter의 `name`은 목록 표시용 라벨일 뿐 호출 이름을 바꾸지 않는다.

### 프로젝트 스킬

리포에 커밋해 팀과 공유하는 단위. 탐색 규칙이 두 갈래다.

- **상위 방향:** Claude Code를 시작한 디렉터리부터 리포 루트까지의 `.claude/skills/`를 모두 읽는다.
- **하위 방향:** 하위 디렉터리 스킬은 시작 시점에 안 잡히고, Claude가 그 디렉터리 안의 파일을 읽거나 수정할 때 로드된다. (모노레포 패키지별 스킬 패턴)

이름이 겹치면 중첩된 쪽은 `/apps/web:deploy`처럼 경로 한정 이름으로 남고, `/deploy`는 루트 것을 실행한다.

> ⚠️ 프로젝트 스킬의 `allowed-tools`는 workspace trust 수락 후 유효하다. 스킬이 스스로에게 넓은 도구 권한을 줄 수 있으니, 남의 리포를 신뢰하기 전에 내용을 봐야 한다.

### 플러그인 스킬

배포 단위가 다르다. 스킬만이 아니라 agents·hooks·MCP 서버까지 한 덩어리로 묶어 마켓플레이스로 배포·버전 관리한다.

- 여기서만 frontmatter `name`이 커맨드의 마지막 세그먼트를 바꾼다 (`my-plugin/skills/review/`에 `name: fancy` → `/my-plugin:fancy`)
- `${CLAUDE_PLUGIN_ROOT}`, 업데이트를 넘어 살아남는 `${CLAUDE_PLUGIN_DATA}` 변수 사용 가능
- `skillOverrides` 영향을 받지 않고 `/plugin`으로 관리

### ⚠️ Cowork·클라우드 세션 주의

**Cowork와 클라우드 세션은 로컬 `~/.claude/skills/`를 읽지 않는다.** claude.ai 계정에 활성화해둔 스킬을 세션 시작 시 동기화해서 받는다. 클라우드 세션만 추가로 클론한 리포의 `.claude/skills/`를 읽는다.

개인 스킬만 만들어두면 웹·Cowork에서는 "스킬을 찾을 수 없다"가 된다. 여러 표면을 쓰는 팀이라면 배포 경로를 처음부터 이걸 전제로 설계해야 한다.

---

## 4. 디렉터리 구조

```
my-skill/
├── SKILL.md          # 필수 — 개요와 네비게이션
├── reference.md      # 상세 문서 — 필요할 때만 로드
├── examples.md       # 사용 예시 — 필요할 때만 로드
└── scripts/
    └── helper.py     # 실행되는 것 — 로드되지 않음
```

### SKILL.md

유일한 필수 파일. frontmatter + 마크다운 본문 구조다.

**한 번 호출되면 렌더된 내용이 메시지 하나로 들어가 세션 끝까지 남는다.** Claude Code는 이후 턴에서 파일을 다시 읽지 않는다. 여기서 두 가지가 따라온다.

- 본문 한 줄 한 줄이 **반복되는 상주 비용**이다. 500줄 이하 유지.
- 작업 내내 적용돼야 할 내용은 일회성 절차가 아니라 **상시 지침**처럼 써야 한다.

### 참조 파일 (reference.md 등)

500줄 제약을 푸는 장치. 긴 API 스펙이나 상세 규격을 빼두면 필요한 순간에만 읽힌다. 단 Claude가 알아서 뒤지지 않으니 SKILL.md에서 명시해야 한다.

```markdown
## Additional resources
- 전체 API 명세는 [reference.md](reference.md) 참고
- 사용 예시는 [examples.md](examples.md) 참고
```

두 가지 규칙이 붙는다.

- **참조는 SKILL.md에서 한 단계만.** 참조 파일이 또 다른 파일을 참조하면 Claude가 `head -100` 같은 부분 읽기로 훑고 넘어가 정보가 불완전해진다.
- **100줄 넘는 참조 파일에는 목차를 단다.** 부분 읽기가 일어나도 전체 범위는 보이게.

### scripts/

읽히는 게 아니라 **실행된다.** 컨텍스트를 전혀 먹지 않고, 출력만 토큰을 소비한다. LLM이 매번 다르게 할 이유가 없는 작업(검증, 변환, 리포트 생성)은 여기로 내린다.

경로는 `${CLAUDE_SKILL_DIR}`로 참조해야 개인·프로젝트·플러그인 어디에 설치되든 안 깨진다. 같은 변수를 `allowed-tools`에도 쓰면 권한 프롬프트 없이 실행된다.

```yaml
---
name: render-chart
allowed-tools: Bash(${CLAUDE_SKILL_DIR}/scripts/render.sh *)
---
차트를 생성하려면 `${CLAUDE_SKILL_DIR}/scripts/render.sh <csv-file>` 를 실행하세요.
```

**실행인지 참조인지 반드시 명시할 것.** "`analyze_form.py`를 실행하라"와 "추출 알고리즘은 `analyze_form.py`를 보라"는 다르다. 애매하면 Claude가 스크립트를 통째로 읽어버려 토큰 이점이 사라진다.

### templates/

공식 규약이 있는 폴더는 아니고, Claude가 채워 넣을 양식이나 기대 출력 형식을 담는 관례적 위치다. 동작 원리는 참조 파일과 같다 — SKILL.md에서 가리켜야 로드된다.

---

## 5. 점진적 공개 3단계

| 단계 | 무엇이 | 언제 |
|---|---|---|
| L1 메타데이터 | name + description | 시작 시 시스템 프롬프트에 상시 |
| L2 SKILL.md 본문 | 지침 전체 | 트리거될 때 |
| L3 번들 파일 | reference.md, scripts/ 등 | 실제로 참조될 때만 |

핵심은 **읽히지 않은 파일은 토큰을 0 쓴다**는 것. 그래서 방대한 API 문서나 데이터셋을 번들해도 설치 비용이 없다. 스크립트는 한 단계 더 나가서, 실행하면 내용이 아니라 출력만 들어온다.

### 컴팩션 시 동작 (Claude Code)

자동 컴팩션이 일어나면 각 스킬의 최근 호출본을 요약 뒤에 다시 붙인다.

- 스킬당 **앞 5,000토큰**까지
- 재부착 전체 합산 **25,000토큰** 예산
- 최근 호출한 것부터 채우므로, 한 세션에서 많이 호출했다면 오래된 스킬은 통째로 떨어져 나간다

"스킬이 중간부터 안 듣는다"는 체감의 상당 부분이 이것이다. 큰 스킬이라면 컴팩션 후 재호출로 복구한다.

---

## 6. 작성 원칙

### 검증된 원칙

**1. 간결함이 최우선.** 컨텍스트 윈도우는 공공재다. 기본 가정은 "Claude는 이미 매우 똑똑하다"이고, Claude가 모르는 정보만 추가한다. PDF가 뭔지 설명하는 문단은 통째로 지워도 된다.

**2. description은 "무엇을 + 언제"를 모두.** 트리거 키워드를 포함하고, **반드시 3인칭으로 쓴다.** description은 시스템 프롬프트에 주입되므로 "I can help you..." 같은 시점이 섞이면 발견 자체가 망가진다.

```yaml
# 좋음
description: PDF에서 텍스트와 표를 추출하고 양식을 채우고 문서를 병합한다.
  PDF 파일을 다루거나 사용자가 PDF, 양식, 문서 추출을 언급할 때 사용한다.

# 나쁨
description: 문서 관련 도움
```

**3. 지원 파일을 도메인별로 분리.** 매출 질문에 재무 스키마만 읽으면 되도록.

```
bigquery-skill/
├── SKILL.md          # 개요와 네비게이션
└── reference/
    ├── finance.md
    ├── sales.md
    └── product.md
```

**4. 자유도(degrees of freedom)를 작업에 맞춘다.** 이게 자주 빠지는 개념이다.

| 자유도 | 형태 | 언제 |
|---|---|---|
| 높음 | 서술형 지침 | 여러 접근이 유효할 때 (코드 리뷰) |
| 중간 | 파라미터 있는 템플릿·의사코드 | 선호 패턴이 있고 변형은 허용될 때 |
| 낮음 | 정확한 스크립트, 플래그 추가 금지 | 순서가 깨지면 안 될 때 (DB 마이그레이션) |

비유하면 — 양옆이 절벽인 좁은 다리에서는 정확한 가드레일을, 탁 트인 들판에서는 방향만 주고 맡긴다.

**5. 입출력 예시는 조건부로.** 출력 품질이 예시를 봐야 결정되는 스킬(커밋 메시지 포맷 등)에는 2~3개 세트가 효과적이다. 다만 모든 스킬에 강제하면 높은 자유도가 맞는 작업까지 한 패턴에 가둔다.

**6. 결정론적 연산은 스크립트로.** 미리 만든 스크립트가 생성 코드보다 신뢰성 높고, 토큰·시간을 아끼고, 일관성을 보장한다. 함께 가야 할 규칙:
- 스크립트가 에러를 직접 처리한다 (Claude에게 떠넘기지 않음)
- 매직 넘버 금지 — `TIMEOUT = 47`은 왜 47인지 모르면 Claude도 모른다
- 경로는 항상 forward slash, Windows에서도

**7. 평가부터 만든다.** 가장 강한 권고인데 가장 자주 빠진다.

```
1. 스킬 없이 실제 태스크 실행 → 실패 지점 문서화
2. 그 갭을 테스트하는 시나리오 3개 작성
3. 스킬 없는 상태로 베이스라인 측정
4. 갭을 메울 최소한의 지침만 작성
5. 실행 → 비교 → 개선 반복
```

상상한 요구사항이 아니라 실제 문제를 풀게 만드는 장치다.

**8. 그 외 체크리스트 항목**
- 시간 종속 정보 배제 (만료되는 내용은 "old patterns" 섹션으로)
- 용어 일관성 — 하나 골라 끝까지 (`API endpoint`면 계속 `API endpoint`)
- 검증 루프 — validator 실행 → 수정 → 반복
- Haiku / Sonnet / Opus 전부에서 테스트
- 선택지 남발 금지 — 기본값 하나 + 예외 탈출구

### 자주 도는 잘못된 수치

커뮤니티에서 도는 숫자 중 공식 근거가 없는 것들이다.

| 흔히 도는 말 | 실제 |
|---|---|
| "SKILL.md는 200줄(한국어 150줄) 이내" | 공식 기준은 **500줄**. 그 이상이면 파일 분리 |
| "description은 100토큰 제한" | `name` 64자, `description` 1,024자. Claude Code에서는 `description` + `when_to_use` 합산 1,536자에서 잘림 |

500줄은 상한이지 목표가 아니다. 짧을수록 좋다는 방향은 맞지만, 실제 기준은 줄 수가 아니라 **토큰**이다. 코드 블록이 많은 스킬은 150줄로도 무겁다. 사내 가이드에 숫자를 못박으면 "150줄 안에 넣었으니 됐다"가 되면서 정작 중요한 판단(이 문단이 토큰값을 하는가)이 생략된다.

---

## 7. 출력 신뢰성 설계

흔히 "환각 방지"라고 부르는 영역. 공식 가이드는 세 가지 지침 계열 기법과, 스킬에서만 가능한 구조적 기법을 함께 제시한다.

### 지침 계열 (공식 "Reduce hallucinations")

**1. 불확실성 표현 허용** — "모르겠다"고 말할 권한을 명시적으로 준다. 단순하지만 잘못된 정보를 크게 줄인다.

> 주의: "확실하지 않으면 말하지 마라"로 쓰면 유용한 잠정 정보까지 사라진다.
> "확실하지 않은 부분은 그렇다고 표시하고 말하라"가 맞는 형태다.

**2. 참조 범위 명시 + 출처 구분** — 긴 문서 작업에서는 먼저 원문 인용구를 뽑아 응답을 실제 텍스트에 접지시킨다. 스킬 맥락에서는 "`reference/` 아래 파일에 없는 내용은 추측하지 말고 없다고 말할 것", 그리고 **제공된 컨텍스트에서 나온 것과 일반 지식에서 나온 것을 응답 안에서 구분**하게 만드는 쪽이 실효적이다.

**3. 출처 명시 + 사후 철회** — 출처를 달게 하는 데서 끝나면 안 된다. 공식 기법은 **응답 생성 후 각 주장을 뒷받침하는 인용구를 찾게 하고, 못 찾으면 그 주장을 철회**하게 만드는 루프까지 포함한다. 출처만 요구하면 출처 자체를 지어내는 실패 모드가 남는다.

### 구조 계열 (스킬이라서 가능한 것)

지침은 확률을 낮추고, **검증은 결정론적이다.**

**4. 검증 가능한 중간 산출물 (plan-validate-execute)**

50개 필드를 수정하는 작업이라면 바로 실행하지 말고:

```
분석 → 계획 파일 생성(changes.json) → 스크립트로 검증 → 실행 → 확인
```

"존재하지 않는 필드를 참조했다"는 환각이 검증 단계에서 죽는다. 검증 스크립트는 장황하게 만든다 — `"'signature_date' 필드 없음. 사용 가능한 필드: ..."` 식으로.

**5. 도구·패키지 실재 확인**
- MCP 도구는 항상 `ServerName:tool_name` 완전한 이름으로. 서버 접두사가 없으면 도구를 못 찾는 일이 생긴다 (여러 MCP 서버를 물린 환경에서 특히)
- 패키지가 설치돼 있다고 가정하지 말 것. 설치 명령을 명시한다

---

## 8. Opus 5에서 달라진 것

두 방향의 변화가 동시에 있어서, 구분해서 대응해야 한다.

### 불확실성 지침의 우선순위는 올라갔다

Opus 5 시스템 카드가 직접 밝히는 부분이다.

- 확신 없는 답을 자신 있게 말한 사례가 **놀랄 만큼 많이** 발견됨
- 전반적으로는 더 정확한데도 **사실 주장 환각은 Opus 4.8보다 약간 더 많음**
- 정확도와 환각률이 같이 오른 이유는 불확실할 때 더 자주 응답하기 때문이라는 해석

즉 "모르면 모른다고 하라"는 지침은 유행이 지난 게 아니라 지금 세대에서 더 필요해졌다.

### 반면 검증 지시는 빼야 한다

공식 Opus 5 프롬프팅 가이드가 명시적으로 **제거하라**고 하는 것들이다.

- "비자명한 작업에는 최종 검증 단계를 포함하라"
- "서브에이전트로 검증하라"
- "답을 다시 확인해라", "응답 전 재검증하라"

Opus 5는 시키지 않아도 스스로 검증하고 자기 실수를 잘 잡는다. 이런 지시는 **과잉 검증을 유발해 토큰만 낭비하고 품질은 개선되지 않는다.** 레거시 하네스의 별도 검증 단계도 마찬가지다.

| 유지 (출력 규범) | 제거 (절차 지시) |
|---|---|
| "모르면 모른다고 말하라" | "다시 확인하라" |
| "이 파일들 밖의 내용은 추측하지 말라" | "검증 단계를 추가하라" |
| "주장마다 출처를 달라" | "서브에이전트로 이중 확인하라" |

### 그 밖에 실무에 영향 있는 것

- **문구를 문자 그대로 따르는 경향이 강해졌다.** 코드 리뷰에서 "심각도 높은 것만 보고하라"고 쓰면 그대로 따라 덜 보고한다. 전부 보고하게 하고 필터링은 별도 단계로 분리하는 게 권장된다.
- **응답이 길어졌다.** effort는 "얼마나 생각하는지"를 조절하지 "얼마나 말하는지"를 조절하지 않는다. 길이를 줄이려면 명시적으로 지시해야 한다.
- **범위를 넓히는 경향이 있다.** 좁은 태스크는 범위를 명시적으로 제한한다.
- **서브에이전트에 더 적극적으로 위임한다.** 작은 작업에는 비용만 늘어난다. 위임 기준을 명시하거나 상한을 건다.
- **effort와 사실성.** 공식 가이드는 low·medium을 비용 조절의 주 수단으로 쓰되 이전 모델에서 가져온 기본값은 다시 스윕하라고 한다. 외부 분석들은 low effort에서 환각률이 크게 올랐다고 보고한다(3자 관측). 사실성이 중요한 스킬은 effort를 낮추기 전에 자체 eval 필수.

---

## 9. 격리 실행: context fork

```yaml
---
name: deep-research
description: 주제를 깊이 조사한다
context: fork
agent: Explore
---
$ARGUMENTS 를 철저히 조사하라:
1. Glob과 Grep으로 관련 파일 찾기
2. 코드 읽고 분석
3. 파일 참조와 함께 결과 요약
```

### 메커니즘

**SKILL.md 내용이 그대로 서브에이전트를 구동하는 프롬프트가 되고, 그 서브에이전트는 메인 대화 히스토리에 접근하지 못한다.**

일반 스킬은 호출되면 본문이 세션 끝까지 컨텍스트에 남는다. fork는 작업이 별도 컨텍스트에서 돌고 **결과 요약만** 돌아온다.

### 실행 모델

- 기본은 **백그라운드**(v2.1.218+). 계속 작업하다가 완료되면 결과가 도착한다
- `background: false`로 호출 턴에서 대기 가능
- 설정과 무관하게 대기하는 경우: 비대화형 모드(`-p`, Agent SDK), `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`, 같은 스킬의 이전 호출이 실행 중일 때, 스케줄 태스크 발화

### 함정 두 가지

- **백그라운드 포크는 좁은 도구 집합으로 돈다.** 스킬 단계가 그 밖의 도구를 필요로 하면 `background: false`로 풀 도구셋 유지
- **백그라운드 포크의 편집은 체크포인트 바깥이다.** `/rewind`로 못 되돌린다. git으로만 복구 가능

### 실행 환경

`agent` 필드로 결정한다. 내장 `Explore` / `Plan` / `general-purpose` 또는 `.claude/agents/`의 커스텀 에이전트. 생략하면 `general-purpose`.

- 시스템 프롬프트는 에이전트 타입에서, 태스크는 SKILL.md 본문에서
- CLAUDE.md도 로드되지만 **Explore와 Plan은 예외** (CLAUDE.md와 git status를 건너뛴다) → 꼭 필요한 지침은 스킬 본문에 다시 써야 한다
- `model`·`effort`를 같이 쓰면 포크된 쪽에 적용

### 언제 쓰는가

세 조건이 모두 참일 때:

1. **결론만 필요하다** — 탐색 과정이 아니라 결과만. 광범위 코드 조사, PR 리뷰, 리서치
2. **실행할 태스크가 있다** — 지침만 있고 태스크가 없으면 서브에이전트는 할 일 없이 끝난다
3. **대화 히스토리가 필요 없다** — 필요한 건 인자로 전부 넘길 수 있어야 한다

부수적 이유: 오래 걸리는 작업을 백그라운드로 돌리고 싶을 때, 읽기 전용 에이전트나 다른 모델/effort로 격리하고 싶을 때.

### 쓰면 안 되는 경우

- 레퍼런스·가이드라인형 스킬 (태스크가 없다)
- 되돌릴 여지가 필요한 편집
- 넓은 도구셋이 필요한 작업
- 다른 스킬과 스택해서 쓰는 스킬 (포크에서 확장이 끊긴다)
- 몇 번의 도구 호출로 끝나는 작은 일 (포크는 파악을 처음부터 다시 한다)
- **검증 목적** — Opus 5 가이드가 명시적으로 반대

> 애매하면 인라인으로 시작하고, "이 스킬 돌리면 컨텍스트가 지저분해진다"는 체감이 오면 `context: fork` 한 줄 추가. 되돌리기도 한 줄이다.

### 반대 방향 패턴과 구분

| | 시스템 프롬프트 | 태스크 |
|---|---|---|
| `context: fork` 스킬 | 에이전트 타입에서 | SKILL.md 본문 |
| `skills` 필드를 가진 서브에이전트 | 서브에이전트 본문 | Claude의 위임 메시지 |

참고로 `disable-model-invocation: true`인 스킬은 서브에이전트 프리로드에서도 제외된다.

---

## 10. 문제 해결

### 진단 순서

```
/skills  →  /context  →  /doctor  →  claude --debug
```

| 명령 | 보여주는 것 |
|---|---|
| `/skills` | 프로젝트·유저·플러그인 소스별 사용 가능한 스킬 |
| `/context` | 컨텍스트를 차지하는 모든 것 (스킬 리스팅 크기 포함) |
| `/doctor` | 설치 상태, 잘못된 설정 파일, 리스팅 비용 추정 |
| `/debug [이슈]` | 디버그 로깅을 켜고 Claude에게 진단 위임 |
| `claude --safe-mode` | 모든 커스터마이징을 끈 세션 (원인 범위 좁히기) |

### `--debug`가 잡아주는 것

두 가지다.

**1. frontmatter YAML 파싱 에러.** 은근히 고약한 실패 모드다. YAML이 깨져 있으면 Claude Code가 **본문은 빈 메타데이터로 로드한다.** 그래서 `/skill-name`은 멀쩡히 동작하는데 매칭할 description이 없다. 증상이 "직접 부르면 되는데 자동 발동만 안 된다"로 나타나 description 문제로 오진하기 쉽다.

**2. 리스팅 예산 초과 경고.** description이 잘려나갈 때 디버그 로그에 경고가 쓰인다.

로그 위치: `~/.claude/debug/<session-id>.txt` · 범위 지정: `claude --debug=mcp`

### 증상별 원인

| 증상 | 원인 | 조치 |
|---|---|---|
| `/skills`에 안 뜸 | 폴더가 아니라 `.claude/skills/name.md` 파일로 만듦 | `.claude/skills/name/SKILL.md` 형태로 |
| `/skills`엔 뜨는데 Claude가 안 부름 | `disable-model-invocation: true`이거나 description 불일치 | `/skills`의 배지 확인 — "user-only"면 자동 발동 안 함 |
| 자동 발동만 안 됨 | frontmatter YAML 깨짐 | `--debug`로 파싱 에러 확인 |
| 스킬이 너무 자주 발동 | description이 광범위 | 더 구체적으로, 또는 `paths`로 제한 |
| 세션 중반부터 안 들음 | 컴팩션으로 재부착 예산 초과 | 재호출로 복구, 또는 본문 축소 |

### 그 외

- **라이브 변경 감지:** `SKILL.md` 수정은 세션 재시작 없이 반영된다. 단 세션 시작 시 없던 최상위 스킬 디렉터리를 새로 만들면 재시작 필요
- 완전히 깨끗한 비교가 필요하면 `CLAUDE_CONFIG_DIR`을 빈 디렉터리로 지정하고 `.claude`도 `.mcp.json`도 없는 곳에서 실행

---

## 11. 최근 업데이트와 추천 기능

### 구조적 변화

| 변경 | 버전 |
|---|---|
| `context: fork` 스킬이 백그라운드 기본, `background: false`로 opt-out | v2.1.218 |
| frontmatter boolean이 `yes`/`no`/`on`/`off`/`1`/`0` 허용 | v2.1.218 |
| 스킬 스태킹 — 한 메시지에 여러 스킬 (첫 스킬 + 최대 5개) | v2.1.199 |
| 재호출 시 동일 내용이면 전문 재적재 안 함 | v2.1.202 |
| 중첩 스킬 디렉터리 경로 한정 이름 | v2.1.203 |
| `${CLAUDE_PROJECT_DIR}` 변수 | v2.1.196 |
| `/run`, `/verify`, `/run-skill-generator` | v2.1.145+ |
| `claude_code.skill_activated` OTEL 이벤트 | v2.1.169+ |

### 잘 안 알려진 frontmatter 필드

**`paths`** — glob 패턴으로 활성화를 제한한다. 해당 패턴의 파일을 다룰 때만 자동 로드. description을 아무리 다듬어도 오발동하는 스킬은 이걸로 잡는 게 확실하다.

```yaml
paths: apps/web/**, packages/ui/**
```

**`metadata`** — 자유 형식 YAML 맵. Claude Code는 내용에 관여하지 않고, 자체 툴링이 SKILL.md에서 읽을 값을 넣는 용도다. 스킬 저장소라면 소유 부서·리뷰 일자·승인 상태를 여기 박고 스크립트로 인덱싱할 수 있다.

**`disallowed-tools`** — 도구를 아예 풀에서 제거. 백그라운드 루프에서 `AskUserQuestion`을 빼는 식.

**`when_to_use`** — 트리거 문구 전용. 단 1,536자 캡을 description과 공유한다.

**`hooks`** — 스킬 수명주기에 붙는 훅. 지침이 아니라 결정론적 강제가 필요할 때. `allowed-tools`나 `hooks`를 쓰는 스킬은 첫 사용 전 사용자 승인이 필요하다.

**`skillOverrides` (settings)** — SKILL.md를 고치지 않고 노출을 제어한다. `/skills` 메뉴에서 스페이스로 순환, 엔터로 저장.

| 값 | Claude에게 노출 | `/` 메뉴 |
|---|---|---|
| `on` | 이름 + 설명 | O |
| `name-only` | 이름만 | O |
| `user-invocable-only` | 숨김 | O |
| `off` | 숨김 | 숨김 |

### 새 번들 스킬 — `/run`, `/verify`, `/run-skill-generator`

테스트나 타입체크로 도망가지 않고 **앱을 실제로 띄워서** 변경이 동작하는지 확인하는 세트다.

- `/run` — 앱을 띄우고 조작해서 변경 확인
- `/verify` — 빌드·실행해서 코드 변경이 의도대로인지 확인
- `/run-skill-generator` — 깨끗한 환경에서 앱을 띄워보고 성공한 설치 명령·환경변수·실행 스크립트를 `.claude/skills/run-<name>/`에 레시피로 커밋

프로젝트당 한 번 돌려두면 이후 `/run`, `/verify`, 다른 에이전트까지 매번 재발견하지 않고 레시피를 따른다. `/verify`도 레시피가 없으면 스스로 알아낸 절차를 기록하고, 이후엔 잘못 안내한 부분만 고치므로 커밋해도 세션마다 diff가 생기지 않는다.

### skill-creator 플러그인 — 만들었으면 측정하자

```
/plugin install skill-creator@claude-plugins-official
```

발동하는 걸 봤다고 잘 동작하는 게 아니다. **발동률**과 **출력 품질**은 따로 재야 하고, 둘 다 스킬을 끈 상태와의 비교로만 알 수 있다.

- **Create / Eval / Improve / Benchmark** 4모드
- 테스트 케이스를 스킬 디렉터리의 `evals/evals.json`에 저장
- 케이스마다 독립 서브에이전트로 병렬 실행 → 컨텍스트 오염 없음, 토큰·시간 기록
- with/without 벤치마크로 통과율 개선을 토큰·시간 오버헤드와 비교
- 두 버전 블라인드 A/B — 어느 쪽인지 모른 채 판정
- description 튜닝 — 발동해야 할/하지 말아야 할 프롬프트를 생성해 적중률 측정 후 수정안 제안

> Anthropic이 자체 문서 생성 스킬로 시험했을 때 공개 스킬 6개 중 5개에서 트리거가 개선됐다. 측정 인프라 없이는 보이지 않았을 개선이다.

### 관측: `skill_activated` 이벤트

OTEL로 스킬 발동을 추적할 수 있다. `invocation_trigger` 속성으로 **`user-slash` / `claude-proactive` / `nested-skill`** 을 구분한다.

여기서 나오는 지표:
- 스킬별 발동 횟수 → 폐기 후보 식별
- **자동 발동 대 수동 호출 비율** → 낮으면 description이 안 먹고 있다는 신호
- 아예 안 잡히는 스킬 → 존재 자체가 알려지지 않았거나 발동 조건이 틀림

가볍게는 `/usage`가 스킬·서브에이전트·플러그인·MCP별로 플랜 한도 소비를 분해해준다. Cowork도 세션 내 스킬·플러그인 호출을 OTel로 흘린다.

---

## 12. 팀 배포 시 고려사항

### 배포 경로별 특성

| 경로 | 방식 | 적합한 경우 |
|---|---|---|
| 프로젝트 스킬 | `.claude/skills/` 커밋 | 리포 단위 규약, 개발팀 |
| 플러그인 | 마켓플레이스 배포 | 여러 스킬 + agents/hooks/MCP 묶음 |
| 조직 프로비저닝 | Organization settings > Skills | 전사 공통, Team·Enterprise 플랜 필요 |
| claude.ai 계정 스킬 | Customize > Skills | Cowork·클라우드 세션 커버 |

### frontmatter 호환성 — 중요

Claude Code 밖으로 나가면 쓸 수 있는 필드가 제한된다.

| 배포 경로 | 사용 가능 필드 |
|---|---|
| Claude Code (플러그인 포함) | 전체 |
| claude.ai 업로드, Skills API, `package_skill.py` 패키징 | `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools` |

**스펙 밖 필드가 있으면 무시가 아니라 하드 에러로 실패한다.**

```
Unexpected key(s) in SKILL.md frontmatter: argument-hint.
Allowed properties are: allowed-tools, compatibility, description, license, metadata, name
```

동적 컨텍스트 주입 같은 본문 기능도 claude.ai 채팅이나 API에서는 동작하지 않는다.

→ **웹·Cowork까지 커버해야 하는 공용 스킬은 6개 필드로만 작성하고, 호출 제어가 필요한 것은 Claude Code 전용 세트로 분리한다.**

### 조직 기능 (Team·Enterprise)

- 조직 전체 스킬 프로비저닝: Organization settings에서 **코드 실행과 스킬을 둘 다** 켜야 한다
- 동료 개별 공유·전사 공유: 소유자가 토글을 켜야 버튼이 나타나며 **기본은 꺼져 있음**
- Enterprise 스킬 보안 스캐닝: 서드파티 스킬의 악성 콘텐츠 검사. 통과 시 정상 설치 / 위험 시 주의 배너 / 악성이면 차단. 기본 꺼짐, 신규 업로드·편집에만 적용

개인 요금제 환경이라 이 경로를 못 쓴다면, 그만큼 **리뷰 절차를 사람이 대신해야 한다.** 공식 엔터프라이즈 가이드의 벳팅 체크리스트가 기준이 된다 — SKILL.md와 참조 마크다운, 번들 스크립트를 전부 읽고 스크립트 동작이 명시된 목적과 일치하는지 확인.

### 기타 배포 옵션

- **심볼릭 링크**: `<skill-name>` 항목을 다른 경로의 디렉터리로 링크 가능. 같은 대상이 여러 경로에서 닿으면 한 번만 로드
- **skills-dir 플러그인**: 스킬 폴더에 `.claude-plugin/plugin.json`을 넣으면 `<name>@skills-dir` 플러그인으로 로드되어 agents·hooks·MCP까지 번들 가능
- **`CLAUDE_CODE_SYNC_SKILLS`**: 비대화형 모드에서 claude.ai 활성화 스킬을 `~/.claude/skills/synced/`로 내려받음 (`synced`는 예약 폴더명)
- **마켓플레이스 관리**: `strictKnownMarketplaces` / `blockedMarketplaces`에 `owner/*` 와일드카드로 GitHub org 단위 허용·차단

---

## 부록: frontmatter 레퍼런스

자주 쓰는 것 위주. 전체는 공식 문서 참고.

| 필드 | 설명 |
|---|---|
| `name` | 목록 표시 이름. 기본값은 디렉터리명. 64자, 소문자·숫자·하이픈만. `anthropic`, `claude` 예약어 금지 |
| `description` | **무엇을 + 언제.** 최대 1,024자. 3인칭으로 작성 |
| `when_to_use` | 트리거 문구 추가. description과 합산 1,536자 캡 |
| `disable-model-invocation` | `true`면 Claude 자동 호출 차단. 스킬 목록에서도 제외 |
| `user-invocable` | `false`면 `/` 메뉴에서 숨김 (자율 실행은 안 막음) |
| `paths` | glob으로 자동 활성화 제한 |
| `allowed-tools` | 호출 턴 동안 승인 없이 쓸 도구. 다음 메시지에 해제 |
| `disallowed-tools` | 스킬 활성 중 제거할 도구 |
| `argument-hint` | 자동완성 힌트. 예: `[issue-number]` |
| `arguments` | 이름 있는 위치 인자 |
| `model` / `effort` | 스킬 활성 중 모델·effort 오버라이드 |
| `context` | `fork`면 서브에이전트에서 격리 실행 |
| `agent` | `context: fork` 시 사용할 서브에이전트 타입 |
| `background` | `context: fork` 시 `false`면 결과를 기다림 |
| `hooks` | 스킬 수명주기 훅 |
| `metadata` | 자유 형식 맵. 자체 툴링용 |

### 문자열 치환

| 변수 | 설명 |
|---|---|
| `$ARGUMENTS` | 전체 인자 |
| `$ARGUMENTS[N]` / `$N` | N번째 인자 (0-based) |
| `$name` | `arguments`에 선언한 이름 있는 인자 |
| `${CLAUDE_SKILL_DIR}` | SKILL.md가 있는 디렉터리 |
| `${CLAUDE_PROJECT_DIR}` | 프로젝트 루트 |
| `${CLAUDE_PLUGIN_ROOT}` / `${CLAUDE_PLUGIN_DATA}` | 플러그인 설치 경로 / 영구 데이터 경로 |
| `${CLAUDE_SESSION_ID}` | 현재 세션 ID |
| `${CLAUDE_EFFORT}` | 현재 effort 레벨 |

### 동적 컨텍스트 주입

`` !`<command>` `` 문법은 스킬 내용이 Claude에게 전달되기 **전에** 셸 명령을 실행하고, 출력으로 자리를 대체한다.

```yaml
---
name: pr-summary
description: PR 변경사항을 요약한다
context: fork
agent: Explore
allowed-tools: Bash(gh *)
---

## PR 컨텍스트
- diff: !`gh pr diff`
- 코멘트: !`gh pr view --comments`

## 작업
위 PR을 요약하라...
```

주의점:
- 명령 하나라도 실패하면 **스킬 호출 전체가 중단된다.** 비정상 종료가 예상되면 `|| true`
- 주입 명령은 권한을 묻지 않는다. 허용되지 않으면 그대로 중단 → `allowed-tools`로 사전 승인
- `!`가 줄 시작이나 공백 뒤에 있어야 인식된다
- 여러 줄은 ` ```! ` 펜스 블록 사용

---

## 참고 링크

**공식 문서**
- [Extend Claude with skills (Claude Code)](https://code.claude.com/docs/en/skills)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Skills for enterprise](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/enterprise)
- [Debug your configuration](https://code.claude.com/docs/en/debug-your-config)
- [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations)
- [Prompting Claude Opus 5](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5)
- [Monitoring (OpenTelemetry)](https://code.claude.com/docs/en/monitoring-usage)
- [What's new (주간 다이제스트)](https://code.claude.com/docs/en/whats-new)

**도구·표준**
- [skill-creator 업데이트 발표](https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills)
- [Agent Skills 오픈 표준](https://agentskills.io)
- [anthropics/skills](https://github.com/anthropics/skills)

---

*스킬 관련 기능은 빠르게 바뀝니다. 버전에 민감한 내용은 공식 문서에서 재확인하세요.*
