---
name: update-claude-code-docs
description: Claude Code 공식 What's New 주차 다이제스트를 확인해, 아직 문서화되지 않은 주차만 한국어 md로 members/junho/docs/claude-code-updates/ 에 추가한다
argument-hint: "[시작 주차/날짜 (기본: 2026-w21)]"
disable-model-invocation: true
allowed-tools: WebFetch(domain:code.claude.com) Read Write Edit Glob Bash(date +%F)
---

Claude Code 신규 기능·업데이트를 공식문서에서 수집해 한국어 md로 축적한다.

- 출력 디렉토리: `${CLAUDE_PROJECT_DIR}/members/junho/docs/claude-code-updates/`
- 기본 기준일: **2026-05-21** (= Week 21, 5/18~22 부터)
- 인자: `$ARGUMENTS` — 주차(`2026-w25`) 또는 날짜(`2026-06-01`). 있으면 그 시점 이후만 수집한다.
- 오늘 날짜: !`date +%F`

## 절차

1. **기존 문서 확인** — 출력 디렉토리에서 `2026-w*.md` 를 Glob 으로 조회한다. 디렉토리가 없으면 이번에 생성한다.
2. **주차 목록 확보** — `https://code.claude.com/docs/en/whats-new/index.md` 를 fetch 해 주차 label, 기간, 릴리스 버전 범위를 얻는다. 주차 번호나 최신 주차를 추측하지 말고 항상 이 index 를 근거로 삼는다.
3. **대상 선정** — 기준 시점 이후 주차 중 **아직 파일이 없는 것만** 고른다.
4. **수집** — 대상 주차마다 `https://code.claude.com/docs/en/whats-new/2026-wNN.md` 를 fetch 한다. 여러 주차는 한 메시지에서 병렬로 fetch 한다.
5. **작성** — 아래 템플릿으로 `2026-wNN.md` 를 만든다. 원문 직역이 아니라 *무엇이 달라졌고 왜 유용한지*가 드러나게 쓰되, 명령·설정키·frontmatter·환경변수 이름은 원문 표기를 그대로 유지한다.
6. **인덱스 갱신** — 같은 디렉토리의 `README.md` 표를 최신 주차가 위로 오도록 갱신한다.
7. **보고** — 추가한 파일, 건너뛴 주차, 사용한 출처 URL을 요약한다.

## 주차 파일 템플릿

```markdown
# Week 21 · 2026-05-18 ~ 05-22

- 릴리스: v2.1.143 → v2.1.149
- 출처: https://code.claude.com/docs/en/whats-new/2026-w21
- 수집일: 2026-07-30

## 주요 기능 — <제목>

<무엇이 달라졌는지, 어떤 상황에서 유용한지 3~5줄>

<사용 예시 — 명령/설정 코드블록>

관련 문서: <링크>

## 기타 변경

- `<기능>` — <한 줄 설명> ([문서](<링크>))
```

## README.md 인덱스 형식

```markdown
# Claude Code 업데이트 기록

`/update-claude-code-docs` 로 공식 [What's New](https://code.claude.com/docs/en/whats-new) 를 수집한 기록. 기준일 2026-05-21 이후.

| 주차 | 기간 | 릴리스 | 주요 기능 |
|------|------|--------|-----------|
| [Week 21](2026-w21.md) | 2026-05-18 ~ 05-22 | v2.1.143 → v2.1.149 | Auto mode on the Pro plan |
```

## 규칙

- **이미 있는 주차 파일은 덮어쓰지 않는다.** 단 기존 파일의 "기타 변경" 항목이 upstream 보다 적으면(주중 부분 발행 등) 빠진 항목만 append 한다. 사람이 손으로 추가한 메모·코멘트는 지우지 않는다.
- 신규 주차가 없으면 파일을 만들지 않고 "변경 없음"만 보고한다.
- 문서·주석은 한국어로 쓴다.
- 공식문서 URL은 `code.claude.com/docs/en/*` 를 쓴다. 구 주소 `docs.claude.com/en/docs/claude-code/*` 는 301 리다이렉트라 fetch 하면 리다이렉트 안내만 돌아온다.
- 페이지 본문에 상대 링크(`/docs/en/...`)가 있으면 `https://code.claude.com/docs/en/...` 절대 URL로 바꿔 기록한다.
- 버그 픽스 단위까지 옮기지 말고, 필요하면 해당 주차의 [changelog](https://code.claude.com/docs/en/changelog) 앵커 링크만 남긴다.
