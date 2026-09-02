# ch5 — 출력 인터페이스

> 그 주 챕터 결과물. 개인 정리본(홍섭·재윤·준호)을 4개 고정 렌즈로 종합했다.
> 개인 정리 원문: [홍섭](./hongseob/정리본.md) · [재윤](./jaeyoon/정리본.md) · [준호](./junho/정리본.md)

---

## 참여자별 (원문 보존)

### 홍섭 — 출력 스타일 vs CLAUDE.md, 그리고 '플러그인 경로'로 말투 성형 (실측)

- **내장 스타일은 3개가 아니라 5개** — draft엔 default/explanatory/learning 3개로 적었지만 공식 문서엔 proactive·concise 포함 5개.
- **출력 스타일 vs CLAUDE.md**: 출력 스타일 = 시스템 프롬프트 **직접 수정**(행동 교체), CLAUDE.md = 유저 메시지로 **append**(지식 추가). 한 줄 — "어떻게 응답하나"를 바꾼다 vs "무엇을 알아야 하나"를 얹는다.
- **빠른 접두사**: `#`(입력을 CLAUDE.md에 저장), `!`(bash 모드). 이 세션에서 실제로 `!`로 `gsw`·`gl`·`git pull`을 돌려 결과를 대화에 넣음(실측 일치).
- **핵심 실측·생각**: 내 `~/.claude/settings.json`엔 `outputStyle` 키가 없어 **default로 돌고 있음.** 대신 `enabledPlugins`에 **`caveman`·`i-have-adhd`·`ponytail` 3개**로 출력 성형을 **플러그인(SessionStart 훅으로 규칙 주입) 경로**로 하는 중 — 출력 스타일 기능(output-styles)은 안 씀. 즉 지금 내 말투 성형은 CLAUDE.md도 출력 스타일도 아닌 **플러그인**이 하고 있어, 세 축의 역할 분리가 실물로 드러남.
- **도입 계획**: `~/.claude/output-styles/`에 프로젝트 성격별(코드작업 / 이력서 피드백 / 면접 롤플레잉)로 만들어 `/config`로 전환.

### 재윤 — 두 출력 표면(출력 스타일·상태 표시줄), 바이너리·settings 실측

- **중심 메시지**: 에이전트엔 출력 표면이 둘 — **무엇을 말하나(출력 스타일)** 와 **지금 어떤 상태인가(상태 표시줄).** 출력 스타일은 시스템 프롬프트를 대체, 상태 표시줄은 stdin JSON을 내 스크립트로 렌더링. 둘 다 설정 파일로 소유, 방향만 반대.
- **바이너리 실측(v2.1.258)**: `strings`로 5개 스타일 이름·설명 문자열, `keep-coding-instructions`(5회), `/output-styles` 슬래시 문자열, **플러그인 매니페스트의 `outputStyles` 키**까지 확인 — 4장 "플러그인이 서브에이전트 배포"와 같은 구조가 출력 스타일에도.
- **커스텀 출력 스타일을 '만들지 않기로' 결정** — ① 교재 예시가 하려는 일(한국어·문서 규칙)을 이 레포는 이미 `CLAUDE.md`가 함(목적 중복) ② 프로젝트 수준은 팀 전체 시스템 프롬프트를 **대체**하는 변경 → 레포 규칙상 모임 합의 사안, 혼자 커밋할 파일 아님 ③ 도입한다면 "추가가 아니라 대체가 필요한 경우"(코딩 지침을 걷어내고 설명에만 집중)에.
- **상태 표시줄이 이 장에서 유일하게 내 시스템에 실물** — settings.json 인라인 bash 원라이너 해부: stdin JSON 5필드(`jq`), `fmt_tokens` 헬퍼(awk로 k/M 축약), `git --no-optional-locks`, `IJ_ADDED`류 변수명에 **IntelliJ Dracula**가 그대로 남음.
- **프롬프트↔결과 어긋남 발견**: 프롬프트는 "`~`는 Mod, `?`는 New"(기호→라벨)인데 구현은 "Mod 3 / New 1"(라벨+개수). settings.json을 실제로 읽고서야 알아챔 — "한 번 만들고 계속 쓰는 산출물"의 함정.
- **정직하게 남긴 미확정**: "custom 시 SWE 지침 제거"의 적용 범위, Default·Proactive 설명 문자열 미발견, `/output-styles` 실동작, `statusLineHealthLatches`.

### 준호 — 시스템 프롬프트 계층 원리 + 최신 공식문서 diff + 확장 주제

- **계층도로 위치 잡기**: 시스템 프롬프트(Output Style **대체 가능**) / 대화 계층(CLAUDE.md append, Skills·Slash) / 별도 컨텍스트(Subagent). 위 둘은 "항상 켜진 배경", 아래는 "불러야 오는 것".
- **흔한 오해 2개 교정**: ① "CLAUDE.md는 대화 길어지면 희석" → **틀림**(시스템 프롬프트·CLAUDE.md 둘 다 맨 앞, 희석되는 건 훅의 prompt injection) ② "시스템 프롬프트는 세션당 1회 전송" → **틀림**(stateless라 매 턴 전체 재전송, '1회'인 건 조립). 역할 의미론 — `system`=협상 불가 vs `user`=밀어낼 수 있음.
- **컴팩션 생존**: 루트 CLAUDE.md는 **생존(재주입)**, 스코프 룰(`paths:`)·대화 중 지시·훅 주입은 소실. → "규칙을 수명으로 분류하라, `/compact` 수동 실행으로 검증".
- **핵심 함정**: 커스텀 스타일은 `keep-coding-instructions: true` 없으면 **SWE 지침을 빼버림(경고도 없음)** — 페르소나 하나 만들고 갑자기 코딩을 못 하게 됨.
- **최신 공식문서 diff(하이라이트)**: `/output-style` deprecated(v2.1.73)→제거(v2.1.91), `:new` 사라짐, 3종→**5종**, `keep-coding-instructions`·`force-for-plugin` 추가, 저장 2곳→**3곳+플러그인**, 세션 1회 로드→`/clear` 필요, 서브에이전트 미적용(fork만 예외). + Deprecation 소동 타임라인·교훈.
- **확장**: 출력 스타일 ≠ structured output(`--json-schema`는 문법 컴파일=디코딩 제약, "출력 스타일로 JSON 강제"는 안티패턴) / `claude -p`는 같은 harness, `--bare` 없는 `-p`의 CI 보안 표면 / status line(유닉스 파이프, `session_id`를 캐시키로) / 단축키·Rewind.

---

## 공통 (셋 다 짚은 것)

- **출력 스타일 = 시스템 프롬프트 '대체'(행동 교체) vs CLAUDE.md = '추가'(지식)** — 셋 다 이 대비를 축으로 삼음. "어떻게 응답하나" ↔ "무엇을 알아야 하나".
- **내장 스타일은 5종** (default·proactive·concise·explanatory·learning) — 홍섭(draft 3→5 교정), 재윤(바이너리 5개 확인), 준호(diff 3종→5종)로 모두 도달.
- **커스텀 출력 스타일의 파일 구조·위치** — `~/.claude/output-styles/`(User)·`.claude/output-styles/`(Project), YAML frontmatter + 마크다운 본문. 3·4장 스킬/서브에이전트와 같은 파일 모양.
- **`keep-coding-instructions`의 무게** — 홍섭(frontmatter 필드로 인지), 재윤(각주로 적용 범위 파고듦), 준호("가장 중요한 함정"). 셋 다 이 키를 핵심으로 봄.

## 흥미로운 것 (인상 깊었던 개념·경험)

- **[홍섭]** 출력 성형을 output-styles 기능이 아니라 **플러그인(`caveman`·`i-have-adhd`·`ponytail`) + SessionStart 훅** 경로로 하고 있다는 실측 — 교재 기능을 안 쓰고 다른 경로로 같은 목적을 달성 중.
- **[재윤]** 바이너리를 직접 `strings`로 떠서 스타일 이름·설명·`keep-coding-instructions`·플러그인 `outputStyles` 키까지 **교차검증**한 것.
- **[재윤]** 상태 표시줄 인라인 스크립트 해부 — `IJ_` 변수명에 IntelliJ Dracula가 남고, 프롬프트("`~`는 Mod")와 결과("Mod 3")가 어긋난 걸 settings.json을 실제로 읽고서야 발견.
- **[준호]** "CLAUDE.md는 길어지면 희석된다"를 **틀렸다**고 교정(둘 다 맨 앞, 희석되는 건 훅 주입) + "조립은 1회, 전송은 매 턴"으로 캐시 원리를 분리.
- **[준호]** 출력 스타일 ≠ structured output — `--json-schema`는 문법으로 컴파일해 디코딩 단계에서 제약. "출력 스타일로 JSON만 내보내게" = 안티패턴.

## 갈린 관점 (같은 걸 다르게 봄)

- **커스텀 출력 스타일을 만들까** — 홍섭은 **도입 계획 있음**(코드작업/이력서/면접 롤플레잉용, `~/.claude/output-styles/`) vs 재윤은 **만들지 않기로 결정**(CLAUDE.md와 목적 중복, 프로젝트 수준은 팀 합의 사안). 같은 기능을 한쪽은 도입, 한쪽은 유보.
- **말투를 바꾸는 실제 경로** — 홍섭은 이미 **플러그인 경로**로 출력 성형 중(기능 대신 플러그인), 재윤·준호는 **output-style 기능 자체**를 분석 축으로. 같은 "톤 제어"에 진입점이 다름.
- **"custom 시 SWE 지침 제거"의 확실성** — 준호는 "가장 중요한 함정"으로 **단정**(keep-coding-instructions로 방지) vs 재윤은 같은 지점을 **미확정(각주)**으로 유보(내장 4종에도 걸리는지 실행검증 안 함). 같은 사실을 확정 vs 유보로 다르게 처리.
- **상태 표시줄의 비중** — 재윤은 발표를 5:5(실물이 상태 표시줄뿐이라 절반을 여기에), 준호는 status line을 확장 주제로, 홍섭은 상태 표시줄을 아예 다루지 않음(출력 스타일·플러그인에 집중).

## 팀이 더 팔 것 (미해결·궁금)

- **[홍섭 §5]** 출력 토큰 절약을 **플러그인 경로 vs 커스텀 출력 스타일** 중 뭐가 유리한가 — 유지보수·토큰 관점의 열린 질문.
- **[재윤 미확정]** "custom 선택 시 SWE 지침 제거"가 내장 4종에도 걸리나 직접 만든 것만인가 / Default·Proactive 원문 설명 문자열 / `/output-styles` 슬래시 실동작 / `statusLineHealthLatches` 역할.
- **[준호 스터디 논점]** 팀 레포에 커밋할 프로젝트 출력 스타일에 어떤 **기계 검증 가능한** 규칙을 넣을까 / `keep-coding-instructions: false`가 기본인 게 합리적인가(왜?) / CI에 `--bare` + `--json-schema`를 어디 쓸까 / **output style을 `-p` 모드에서 쓸 수 있나(문서 명시 없음 — 실험)** / 컴팩션 넘어 살아남아야 할 규칙은 뭐고 지금 어디 있나(`/compact`로 검증).
- **[팀 공통]** 프로젝트 수준 출력 스타일은 팀 전체 시스템 프롬프트를 대체 → 재윤 지적대로 **모임 합의 안건.** 우리 스터디 레포에 하나 만들까 말까?

---

### 참고 출처 (정리본에서 모음)

- Output styles — https://code.claude.com/docs/en/output-styles
- Status line — https://code.claude.com/docs/en/statusline
- Headless / Agent SDK CLI — https://code.claude.com/docs/en/headless
- Context window (컴팩션 생존 범위) — https://code.claude.com/docs/en/context-window · Prompt caching — https://code.claude.com/docs/en/prompt-caching · Checkpointing — https://code.claude.com/docs/en/checkpointing

> 버전 의존 사실(스타일 5종, `keep-coding-instructions`, deprecation 타임라인 등)은 2026-09 기준 실측·공식문서 확인값이다. 미확정으로 표시된 항목은 실행 검증 전이다.
