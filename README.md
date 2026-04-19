# 🧠 Claude Skills

Claude Code(터미널)에서 바로 쓸 수 있는 스킬 플러그인 모음입니다.

---

## 설치 방법

### 1. Claude Code 설치 (처음이라면)
```bash
npm install -g @anthropic-ai/claude-code
```

### 2. 스킬 다운로드
[Releases](./releases) 폴더에서 원하는 `.skill` 파일을 다운로드합니다.

### 3. 스킬 등록
```bash
claude skill install code-review.skill
```

### 4. 사용
```bash
# 프로젝트 폴더에서
claude "src/app.py 리뷰해줘"
claude "src/ 전체 리뷰해줘"
```

---

## 스킬 목록

| 스킬 | 설명 | 다운로드 |
|------|------|----------|
| 🔍 code-review | 로컬 코드 품질 분석 — 심각도별 이슈 분류 및 수정 코드 제안 | [code-review.skill](./releases/code-review.skill) |

---

## 기여 방법

스킬을 직접 만들어 PR을 보내주세요!

**폴더 구조:**
```
skills/
└── your-skill-name/
    ├── SKILL.md           # 필수
    └── references/        # 선택 (참고 문서)
        └── *.md
```

**SKILL.md 형식:**
```yaml
---
name: your-skill-name
description: >
  스킬 설명. Claude가 이 텍스트를 보고 언제 스킬을 쓸지 결정합니다.
  트리거 조건을 구체적으로 적을수록 잘 동작합니다.
---

# 스킬 이름

스킬 본문...
```

**패키징:**
```bash
claude skill package skills/your-skill-name
```

---

## 라이선스

MIT
