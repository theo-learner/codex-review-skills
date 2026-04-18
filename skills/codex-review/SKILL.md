---
name: codex-review
description: Use when you want Codex to independently review code, a plan, or a document.
  Triggers: "codex로 리뷰해줘", "코덱스 리뷰", "codex review",
  "이 계획 리뷰해줘", "이 문서 리뷰해줘", "plan 검토해줘", "spec 검토해줘",
  "get codex to review this", "independent review", "second opinion from codex".
  With file argument: /codex-review path/to/file.md reviews that file directly.
  Without argument: reviews git diff (code review mode).
---

# Codex Review

## Overview

코드(git diff), 계획서(PLAN.md), 또는 문서(spec, design doc)를 Codex CLI로 리뷰하고
결과를 `CODEX_REVIEW.md`에 저장한 뒤 HIGH 이슈를 요약한다.

## Step 1: Prerequisites 확인

```bash
# Codex CLI 설치 확인
command -v codex >/dev/null 2>&1 || echo "ERROR: codex not installed. Run: npm install -g @openai/codex"
```

파일 인수 없이 실행 시 추가 확인:
```bash
# REVIEW_CONTEXT.md 존재 확인 (인수 없는 코드 리뷰 모드)
test -f REVIEW_CONTEXT.md || echo "ERROR: REVIEW_CONTEXT.md not found. Run /prep-review first, or pass a file path: /codex-review path/to/file.md"
```

오류가 있으면 해당 메시지를 출력하고 중단한다.

## Step 2: 인수 파싱 및 타입 감지

`$ARGUMENTS`로 전달된 파일/디렉토리 경로를 파싱한다.

**인수 없음** → 코드 리뷰 모드 (REVIEW_CONTEXT.md + git diff)

**인수 있음** → 파일명 패턴으로 타입 자동 감지:

| 패턴 | 타입 | 프롬프트 포커스 |
|------|------|----------------|
| `*-PLAN.md`, `*PLAN*.md` | plan | 완결성, 의존성, 실현가능성 |
| `*-design.md`, `*spec*`, `*SPEC*` | doc | 정확성, 일관성, 누락 섹션 |
| `*ROADMAP*`, `*REQUIREMENTS*` | doc | 커버리지, 우선순위, 범위 |
| 그 외 `.md` | doc | 일반 문서 리뷰 |

디렉토리가 지정된 경우 해당 디렉토리의 모든 `.md` 파일을 수집한다:
```bash
# 디렉토리인 경우
find "$ARGUMENTS" -name "*.md" | sort | xargs cat
# 파일인 경우
cat "$ARGUMENTS"
```

## Step 3: Codex 호출

**코드 리뷰 모드 (인수 없음):**

```bash
DIFF=$(git diff HEAD 2>/dev/null || git diff 2>/dev/null)
CONTEXT=$(cat REVIEW_CONTEXT.md)

codex exec --skip-git-repo-check "
${CONTEXT}

## Actual Changes (git diff)
\`\`\`diff
${DIFF}
\`\`\`

Review this implementation. Provide structured feedback:

1. **Summary** — One paragraph assessment of the overall implementation
2. **Issues**
   - HIGH: bugs, security issues, data loss risks, broken logic
   - MEDIUM: edge cases, missing error handling, performance concerns
   - LOW: style, naming, minor improvements
3. **Suggestions** — Specific actionable improvements with code examples where helpful
4. **Overall Risk** — LOW / MEDIUM / HIGH with one-sentence justification

Output structured markdown.
" > /tmp/codex-review-output.md 2>&1
```

**계획 리뷰 모드 (파일 인수, plan 타입):**

```bash
CONTENT=$(cat "$ARGUMENTS")  # 또는 디렉토리 collect

codex exec --skip-git-repo-check "
${CONTENT}

Review this implementation plan. Provide structured feedback:

1. **Completeness** — Are all requirements covered? Missing steps?
2. **Feasibility** — Are the steps realistic? Any hidden complexity?
3. **Dependencies** — Are task dependencies correct? Anything missing?
4. **Risks** — What could go wrong? HIGH/MEDIUM/LOW
5. **Suggestions** — Specific improvements with concrete examples

Output structured markdown.
" > /tmp/codex-review-output.md 2>&1
```

**문서 리뷰 모드 (파일 인수, doc 타입):**

```bash
CONTENT=$(cat "$ARGUMENTS")  # 또는 디렉토리 collect

codex exec --skip-git-repo-check "
${CONTENT}

Review this document. Provide structured feedback:

1. **Accuracy** — Any factually incorrect or outdated information?
2. **Clarity** — Ambiguous or confusing sections?
3. **Completeness** — Missing sections or important omissions?
4. **Consistency** — Internal contradictions?
5. **Suggestions** — Specific improvements

Output structured markdown.
" > /tmp/codex-review-output.md 2>&1
```

## Step 4: CODEX_REVIEW.md 저장

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
{
  echo "# Codex Review — ${TIMESTAMP}"
  echo ""
  cat /tmp/codex-review-output.md
} > CODEX_REVIEW.md
rm /tmp/codex-review-output.md
```

## Step 5: 완료 후 요약

CODEX_REVIEW.md를 읽고 다음 형식으로 요약:

```
CODEX_REVIEW.md 저장 완료.

HIGH 이슈:
- {이슈 1}
- {이슈 2}
(없으면 "HIGH 이슈 없음")

전체 리뷰: CODEX_REVIEW.md
```

## 오류 처리

| 상황 | 대응 |
|------|------|
| `codex` 미설치 | `npm install -g @openai/codex` 안내 후 중단 |
| `REVIEW_CONTEXT.md` 없음 (인수 없음) | `/prep-review` 먼저 실행 안내 후 중단 |
| 지정 파일 없음 | 파일 경로 확인 요청 |
| `git diff` 비어있음 | 경고 출력 후 계속 (커밋된 변경사항일 수 있음) |
| Codex 호출 실패 | `/tmp/codex-review-output.md` 내용 확인 안내 |
