---
name: prep-review
description: Use when implementation, plan, or document is complete and you want to
  prepare context for Codex review. Triggers: "prep review", "prepare for codex",
  "리뷰 준비해줘", "코덱스에 보낼 컨텍스트 만들어", "구현 끝났으니 리뷰 준비",
  "이 문서 리뷰 준비해줘", "이 계획 리뷰 준비해줘",
  after subagent-driven-development completes, after gsd:execute-phase completes,
  after writing a plan or spec. Flags: --doc <path> for plan/document review.
---

# Prep Review

## Overview

구현 완료 후 Codex 리뷰를 위한 컨텍스트 파일을 생성한다.
현재 대화에서 작업 배경과 설계 결정을 추출해 `REVIEW_CONTEXT.md`로 구조화한다.

## What Claude Does

현재 대화 컨텍스트에서 다음을 추출해 `REVIEW_CONTEXT.md`를 repo root에 생성:

1. **Task Background** — 무엇을 왜 만들었는가
2. **Design Decisions** — 왜 이 방식을 택했는가 (고려한 대안 포함)
3. **Changed Files** — `git diff --name-only` 실행 결과
4. **Review Focus** — 불확실하거나 특히 확인받고 싶은 부분
5. **GSD PLAN.md** — GSD 프로젝트인 경우 자동 포함
6. **Document to Review** — `--doc <path>` 플래그 사용 시 해당 파일 내용 포함

## GSD 프로젝트 감지

```bash
ls .planning/phases/ 2>/dev/null && echo "gsd" || echo "general"
```

GSD 프로젝트면 가장 최근 phase의 `*-PLAN.md`를 자동으로 포함:

```bash
ls -t .planning/phases/*/*.md 2>/dev/null | head -5
```

## `--doc <path>` 플래그 (계획/문서 리뷰)

`$ARGUMENTS`에 `--doc <path>` 형식으로 전달하면:
- git diff 대신 해당 파일 내용을 `## Document to Review` 섹션에 포함
- 기존 Task Background / Design Decisions 섹션은 유지

```bash
# 플래그 파싱
if echo "$ARGUMENTS" | grep -q "\-\-doc"; then
  DOC_PATH=$(echo "$ARGUMENTS" | sed 's/--doc //')
  DOC_CONTENT=$(cat "$DOC_PATH" 2>/dev/null || echo "ERROR: file not found: $DOC_PATH")
fi
```

## Output Format

생성되는 `REVIEW_CONTEXT.md` 구조:

```markdown
## Task Background
{작업 목적 — 무엇을 왜 만들었나}

## Design Decisions
{핵심 설계 결정과 근거. 택하지 않은 대안도 포함}

## Changed Files
{git diff --name-only 결과}

## Review Focus
{특히 확인받고 싶은 부분, 확신이 없는 부분}

## Source: GSD PLAN.md
{GSD 프로젝트인 경우만 포함}

## Document to Review
{--doc 플래그 사용 시만 포함 — 파일 전체 내용}
```

## 완료 후 안내

```
REVIEW_CONTEXT.md 생성 완료.

다음 단계:
  /codex-review              ← 코드 리뷰 (git diff 기반)
  /codex-review path/to/file ← 파일 직접 지정 리뷰
```

## 워크플로우 통합

| 워크플로우 | 언제 실행 |
|---|---|
| `subagent-driven-development` | 모든 task 완료 후 자동 권장 |
| `gsd:execute-phase` | phase 실행 완료 후 |
| 일반 구현 | 구현 완료 후 수동 호출 |
| 계획/문서 리뷰 | `--doc path/to/file.md` 플래그로 파일 지정 |
