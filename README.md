# codex-review-skills

Claude Code skill for structured Codex review.

## Skills

- **`/codex-review`** — Codex CLI로 코드/계획/문서를 독립 리뷰하고 `CODEX_REVIEW.md`에 저장

## Install

```bash
# 1. marketplace 등록 (1회)
claude plugin marketplace add github:theo-learner/codex-review-skills

# 2. plugin 설치
claude plugin install codex-review-skills
```

## Usage

### 코드 리뷰 (구현 완료 후)

```
/codex-review
```

대화 컨텍스트를 수집해 `REVIEW_CONTEXT.md`를 생성한 뒤 `git diff`를 Codex로 리뷰합니다.

### 계획/문서 리뷰 (파일 직접 지정)

```
/codex-review path/to/PLAN.md
/codex-review path/to/design.md
/codex-review .planning/phases/07-drb/
```

### 대화 컨텍스트 + 문서 리뷰

```
/codex-review --doc path/to/file.md
```

## Update

```bash
claude plugin update codex-review-skills
```

## Requirements

- [Codex CLI](https://github.com/openai/codex): `npm install -g @openai/codex`
