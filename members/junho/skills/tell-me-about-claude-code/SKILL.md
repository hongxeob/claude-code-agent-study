---
name: tell-me-about-claude-code
description: 입력한 Claude Code 키워드·기능을 공식문서(code.claude.com/docs) 근거로 한국어로 설명한다
argument-hint: "[키워드 (예: remote control, hooks, 서브에이전트)]"
disable-model-invocation: true
allowed-tools: WebFetch(domain:code.claude.com) WebSearch
---

키워드: `$ARGUMENTS`

## 절차

1. 키워드가 비어 있으면 무엇을 알고 싶은지 되묻고 끝낸다.
2. `https://code.claude.com/docs/llms.txt` 를 fetch 해 전체 페이지 색인(제목 + URL)을 얻고, 키워드에 맞는 페이지를 1~3개 고른다. 한국어 키워드는 영어 문서 용어로 매핑한다 — 훅→hooks, 원격 제어→remote-control, 서브에이전트→sub-agents, 예약 작업→scheduled-tasks, 권한→permissions, 설정→settings, 기억/메모리→memory.
3. 고른 페이지의 `.md` URL 을 한 메시지에서 병렬 fetch 한다. 이해에 꼭 필요하면 연관 페이지 1개까지 추가로 읽는다.
4. 아래 순서로 한국어로 답한다.
   - **한 줄 정의** — 이게 뭔지
   - **언제/왜 쓰나** — 실제 사용 맥락
   - **핵심 동작과 설정** — 실제 명령, 설정키, frontmatter, 환경변수를 원문 표기 그대로 예시와 함께
   - **제약·주의점** — 최소 버전 요구, 플랜·플랫폼 제한, 알려진 함정
   - **관련 문서** — 더 볼 페이지
5. 마지막에 참고한 출처 URL을 모두 나열한다.
6. 색인에서 못 찾으면 `WebSearch`(allowed_domains: `code.claude.com`)로 보강한다.

## 규칙

- **공식문서에 근거가 없는 내용은 쓰지 않는다.** 확인되지 않으면 "공식문서에서 확인되지 않음"이라고 밝히고 추측으로 채우지 않는다.
- 공식문서 URL은 `code.claude.com/docs/en/*` 를 쓴다. 구 주소 `docs.claude.com/en/docs/claude-code/*` 는 301 리다이렉트라 fetch 하면 리다이렉트 안내만 돌아온다.
- 상대 링크(`/docs/en/...`)는 `https://code.claude.com/docs/en/...` 절대 URL로 바꿔 제시한다.
- 문서 원문을 통째로 옮기지 말고, 이 레포에서 바로 써먹을 수 있는 수준으로 압축한다.
- 파일은 만들지 않는다. 대화로만 답한다.
