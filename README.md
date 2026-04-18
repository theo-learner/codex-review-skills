# codex-review-skills

Claude Code skills for structured Codex review workflow.

## Skills

- **`/prep-review`** — 구현/계획/문서 완료 후 Codex 리뷰용 컨텍스트 파일(`REVIEW_CONTEXT.md`) 생성
- **`/codex-review`** — Codex CLI를 호출해 코드/계획/문서를 리뷰하고 `CODEX_REVIEW.md`에 저장

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
/prep-review
/codex-review
```

### 계획/문서 리뷰 (파일 직접 지정)

```
/codex-review path/to/PLAN.md
/codex-review path/to/design.md
```

## Update

```bash
claude plugin update codex-review-skills
```

## Requirements

- [Codex CLI](https://github.com/openai/codex): `npm install -g @openai/codex`
