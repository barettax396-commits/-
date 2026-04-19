---
name: code-review
description: >
  Reviews local code files from the terminal with severity-graded feedback and
  concrete fixes. Use this skill whenever the user mentions a local file path or
  directory and asks for review, feedback, or improvement — even phrased casually
  like "이 코드 봐줘", "뭐가 문제야?", "리뷰해줘", "check this", "any issues here?",
  or "roast my code". Designed for Claude Code (terminal) usage where files are
  read directly from the local filesystem. Produces a structured report: summary,
  severity-graded issues with file+line references, and copy-pasteable fixed code.
---

# Code Review Skill (Claude Code / Terminal)

Review local source code directly from the filesystem. Read files with bash_tool,
produce clear actionable feedback, and show fixes the user can paste in directly.

## Handling Input

**Single file** (`claude "src/app.py 리뷰해줘"`):
Use bash_tool to `cat` the file, then review it.

**Multiple files** (`claude "src/ 리뷰해줘"`):
Use bash_tool to list the directory (`find <dir> -type f`), read each relevant
source file, and cross-reference issues that span files.

**Whole project** (`claude "전체 리뷰해줘"`):
Start by reading the directory tree, identify entry points and key files,
then review top-down. Lead with an architecture-level comment before
diving into individual file issues.

---

## Review Structure

Always follow this exact structure:

### 1. 📋 Summary
2–4 sentences. What does this code do? What's the overall quality?
Mention the language/framework detected. Flag any immediate red flags.

### 2. 🔴 Critical Issues  *(skip section if none)*
Bugs, security vulnerabilities, data loss risks, crashes. Must be fixed.
```
**[파일명:줄번호]** 제목
문제: 무엇이 왜 잘못됐는지
수정:
\`\`\`언어
// fixed code here
\`\`\`
```

### 3. 🟡 Warnings  *(skip section if none)*
Performance issues, bad practices, unclear logic, missing error handling.
Should be fixed. Same format as Critical.

### 4. 🟢 Suggestions  *(skip section if none)*
Style, naming, readability, nice-to-haves. Worth considering.
Bullet points with brief explanation + one-line fix OK.

### 5. ✅ Positives
1–3 things done well. Be genuine — acknowledge good patterns, clever
solutions, or clean structure.

---

## Review Checklist (run through these mentally)

**Correctness**
- Off-by-one errors, null/undefined derefs, type mismatches
- Edge cases: empty input, zero, negative, max values
- Race conditions / concurrency issues
- API/contract violations

**Security**
- SQL injection, XSS, command injection
- Hardcoded secrets or credentials
- Unvalidated user input
- Improper auth/authz

**Performance**
- N+1 queries
- Unnecessary re-renders or recomputation
- Missing indexes (if schema is visible)
- Memory leaks

**Maintainability**
- Function/variable names that lie or confuse
- Functions doing too many things (> ~30 lines is a smell)
- Magic numbers/strings without constants
- Copy-paste code that should be extracted
- Missing or misleading comments on complex logic

**Error Handling**
- Swallowed exceptions
- Generic catch-all error messages
- Missing cleanup (finally, defer, RAII)

---

## Tone & Language

- Match the user's language (한국어로 질문하면 한국어로 리뷰, English question → English review)
- Direct but not harsh. "This will crash when X is null" not "this is terrible"
- Show actual fixed code, not just descriptions of fixes
- If the code is genuinely good, say so briefly and focus on polish

## Context Gathering

If the code references external modules or env variables that aren't visible,
note what can't be assessed but still review what's readable.
Ask at most one clarifying question if truly critical —
e.g., "이게 FastAPI 서버야, 아니면 스크립트야?" Only if it materially changes the review.

## Language-Specific Notes

Read the appropriate section from `references/language-notes.md` when reviewing:
- **Python**: focus on exception handling, mutability traps, type hints
- **JavaScript/TypeScript**: async/await pitfalls, null coalescing, TS strictness
- **Go**: error ignoring, goroutine leaks, defer usage
- **SQL**: injection, index usage, N+1 patterns
- For other languages: apply general checklist, note language-specific idioms where relevant
