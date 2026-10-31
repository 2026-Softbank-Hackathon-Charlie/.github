# agent-rules 설치 방법

이 폴더의 내용을 팀 레포 **루트**에 그대로 복사한다.

```
AGENTS.md                                   # Codex, Cursor 등이 자동으로 읽는 공통 규칙 (요약)
CLAUDE.md                                   # Claude Code 진입점 (@AGENTS.md import + skill 안내)
.claude/skills/finals-constraints/SKILL.md  # 전체 규칙 (Single Source of Truth)
.cursor/rules/finals-constraints.mdc        # Cursor 항상 적용 규칙
```

복사: `cp -r agent-rules/. <repo-root>/`
규칙을 바꿀 때는 SKILL.md를 먼저 고치고, AGENTS.md 요약을 맞춘다.
