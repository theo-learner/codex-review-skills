---
name: codex-review
description: Use when you want Codex to independently review code, a plan, or a document.
  Triggers: "codex로 리뷰해줘", "코덱스 리뷰", "codex review", "리뷰해줘",
  "이 계획 리뷰해줘", "이 문서 리뷰해줘", "plan 검토해줘", "spec 검토해줘",
  "get codex to review this", "independent review", "second opinion from codex",
  after subagent-driven-development completes, after gsd:execute-phase completes,
  after writing a plan or spec.
  Without argument: gathers conversation context then reviews git diff (code review).
  With file/dir argument: reviews that file directly.
  With --doc path: gathers conversation context + includes specified file.
  With --topic <name>: explicit slug override.
---

# Codex Review

## Overview

코드(git diff), 계획서, 또는 문서를 Codex CLI로 독립 리뷰하고 결과를 `docs/codex-review/<topic>/`에 저장한다.
같은 루트에서 여러 번 호출해도 topic별 디렉토리에 분리 저장되어 덮어쓰지 않는다.

| 호출 방식 | 동작 |
|-----------|------|
| `/codex-review` | 대화 컨텍스트 수집 → git diff 코드 리뷰 |
| `/codex-review path/to/PLAN.md` | 파일 직접 리뷰 (타입 자동 감지) |
| `/codex-review --doc path/to/file.md` | 대화 컨텍스트 + 지정 파일 리뷰 |
| `/codex-review --topic <slug>` | slug 명시적 지정 |

저장 경로: `docs/codex-review/<topic>/CODEX_REVIEW.md`, `docs/codex-review/<topic>/REVIEW_CONTEXT.md`

## Step 1: Slug 결정

`$ARGUMENTS`를 파싱해 slug와 파일 인수를 분리한다:

```bash
SLUG=""
FILE_ARG=""
DOC_ARG=""

# --topic 파싱
if echo "$ARGUMENTS" | grep -q -- "--topic"; then
  SLUG=$(echo "$ARGUMENTS" | sed 's/.*--topic[[:space:]]\+\([^[:space:]]*\).*/\1/' \
    | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g' | sed 's/-\+/-/g' | sed 's/^-\+//;s/-\+$//')
fi

# --doc 파싱
if echo "$ARGUMENTS" | grep -q -- "--doc"; then
  DOC_ARG=$(echo "$ARGUMENTS" | sed 's/.*--doc[[:space:]]\+\([^[:space:]]*\).*/\1/')
fi

# 나머지 파일 인수 (--topic / --doc 플래그 제거 후)
FILE_ARG=$(echo "$ARGUMENTS" \
  | sed 's/--topic[[:space:]]\+[^[:space:]]*//' \
  | sed 's/--doc[[:space:]]\+[^[:space:]]*//' \
  | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
```

Slug 우선순위:

| 순서 | 조건 | slug 결정 방법 |
|------|------|----------------|
| 1 | `--topic` 명시 | 해당 값 사용 |
| 2 | `--doc <path>` | 문서 파일 basename에서 파생 |
| 3 | `FILE_ARG` 있음 | 파일/디렉토리 이름에서 파생 |
| 4 | 인수 없음 (코드 리뷰) | **Claude가 대화 컨텍스트에서 직접 결정** |

파일 기반 slug 파생:
```bash
# DOC_ARG 또는 FILE_ARG에서 파생 (SLUG가 비어있을 때만)
if [ -z "$SLUG" ] && [ -n "$DOC_ARG" ]; then
  SLUG=$(basename "$DOC_ARG" .md | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g')
elif [ -z "$SLUG" ] && [ -n "$FILE_ARG" ]; then
  SLUG=$(basename "$FILE_ARG" .md | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g')
fi
```

**인수 없는 코드 리뷰 모드에서 Claude가 slug를 직접 결정하는 규칙:**
- 무엇을 만들었는지/고쳤는지를 핵심 단어로 표현
- 소문자, 영어, 숫자, 하이픈만 사용 (`[a-z0-9-]`)
- 2–4단어, 너무 일반적인 단어 단독 사용 금지 (`fix`, `update`, `change` 단독 X)
- 예시: `usdc-token-support`, `electron-e2e-spec`, `auth-middleware-fix`, `docker-go-mod-cache`

빈 slug 방어:
```bash
if [ -z "$SLUG" ]; then
  SLUG=$(date +%Y-%m-%d-%H%M)
fi
```

출력 디렉토리 설정:
```bash
REVIEW_DIR="docs/codex-review/${SLUG}"
mkdir -p "$REVIEW_DIR"
CONTEXT_FILE="${REVIEW_DIR}/REVIEW_CONTEXT.md"
OUTPUT_FILE="${REVIEW_DIR}/CODEX_REVIEW.md"
```

## Step 2: Prerequisites 확인

```bash
command -v codex >/dev/null 2>&1 || { echo "ERROR: codex not installed. Run: npm install -g @openai/codex"; exit 1; }
```

오류가 있으면 해당 메시지를 출력하고 중단한다.

## Step 3: 모드 결정 및 콘텐츠 준비

### 인수 없음 — 코드 리뷰 모드

현재 대화에서 컨텍스트를 추출해 `$CONTEXT_FILE`을 생성한다:

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
```

GSD 프로젝트 감지 및 PLAN.md 자동 포함:
```bash
ls .planning/phases/ 2>/dev/null && echo "gsd" || echo "general"
ls -t .planning/phases/*/*.md 2>/dev/null | head -5
```

이후 Codex에 전달할 콘텐츠:
```bash
CONTENT=$(cat "$CONTEXT_FILE")
DIFF=$(git diff HEAD 2>/dev/null || git diff 2>/dev/null)
TYPE="code"
```

### `--doc <path>` — 문서+컨텍스트 리뷰 모드

위와 동일하게 대화 컨텍스트를 수집하되, `$CONTEXT_FILE`에 `## Document to Review` 섹션을 추가한다:

```bash
DOC_CONTENT=$(cat "$DOC_ARG" 2>/dev/null || echo "ERROR: file not found: $DOC_ARG")
```

$CONTEXT_FILE 끝에 추가:
```markdown
## Document to Review
{$DOC_CONTENT}
```

이후:
```bash
CONTENT=$(cat "$CONTEXT_FILE")
TYPE="doc"
```

### 파일/디렉토리 경로 — 직접 리뷰 모드

파일명 패턴으로 타입 자동 감지:

| 패턴 | 타입 |
|------|------|
| `*-PLAN.md`, `*PLAN*.md` | plan |
| `*-design.md`, `*spec*`, `*SPEC*`, `*ROADMAP*`, `*REQUIREMENTS*` | doc |
| 그 외 `.md` | doc |

```bash
# 디렉토리인 경우
find "$FILE_ARG" -name "*.md" | sort | xargs cat > /tmp/review-input.md
# 파일인 경우
cat "$FILE_ARG" > /tmp/review-input.md

CONTENT=$(cat /tmp/review-input.md)

# 타입 자동 감지
case "$FILE_ARG" in
  *PLAN*.md|*-PLAN.md) TYPE="plan" ;;
  *-design.md|*spec*|*SPEC*|*ROADMAP*|*REQUIREMENTS*) TYPE="doc" ;;
  *) TYPE="doc" ;;
esac
```

## Step 4: Codex 호출

**code 타입:**
```bash
codex exec --skip-git-repo-check "
${CONTENT}

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

**plan 타입:**
```bash
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

**doc 타입:**
```bash
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

## Step 5: 저장

```bash
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
{
  echo "# Codex Review — ${TIMESTAMP}"
  echo ""
  cat /tmp/codex-review-output.md
} > "$OUTPUT_FILE"
rm -f /tmp/codex-review-output.md /tmp/review-input.md
```

## Step 6: 완료 후 요약

`$OUTPUT_FILE`을 읽고 다음 형식으로 요약:

```
${OUTPUT_FILE} 저장 완료.

HIGH 이슈:
- {이슈 1}
- {이슈 2}
(없으면 "HIGH 이슈 없음")

전체 리뷰: ${OUTPUT_FILE}
```

## 오류 처리

| 상황 | 대응 |
|------|------|
| `codex` 미설치 | `npm install -g @openai/codex` 안내 후 중단 |
| 지정 파일 없음 | 파일 경로 확인 요청 |
| `git diff` 비어있음 | 경고 출력 후 계속 (커밋된 변경사항일 수 있음) |
| Codex 호출 실패 | `/tmp/codex-review-output.md` 내용 확인 안내 |
