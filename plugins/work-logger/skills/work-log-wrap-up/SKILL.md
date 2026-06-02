---
name: work-log-wrap-up
description: |
  Claude/Codex 세션, Obsidian 문서, 미팅 노트를 모아 daily/weekly/monthly work log를 생성하고, 필요하면 Apple Reminders에 오늘 할 일을 생성할 때 사용.
  daily-work-logger, weekly-work-logger, monthly-work-logger의 공통 수집 로직과 출력 규칙을 제공한다.
user_invocable: false
---

# Work Log Wrap Up

이 skill은 사용자에게 직접 노출하기보다 `daily-work-logger`, `weekly-work-logger`,
`monthly-work-logger`, 그리고 호환용 `weekly-work-summary`/`monthly-work-summary`가
공유하는 규칙과 helper script를 제공한다.

공통 원칙:

- Vault는 `$VAULT_ROOT`가 있으면 우선 사용하고, 없으면 `~/Documents/Obsidian Vault`를 사용한다.
- 출력 위치는 기존 Claude Code 루틴과 동일하게 `notes/work-log/daily`, `notes/work-log/weekly`, `notes/work-log/monthly`를 사용한다.
- 세션 소스는 Claude와 Codex를 모두 지원한다. 기본값은 `WORK_LOG_SESSION_SOURCES=claude,codex`다.
- Claude는 `~/.claude/projects/**/*.jsonl`에서 top-level 세션을 수집한다. `subagents/` 로그는 기본 수집에서 제외한다.
- Codex는 `~/.codex/sessions`, `~/.codex/archived_sessions`, `~/.codex/session_index.jsonl`에서 수집한다.
- Things는 사용하지 않는다.
- Apple Reminders 자동 생성은 `scripts/apple_reminders.py`가 담당한다. 인증이 필요 없고, macOS Reminders 앱의 `Daily Focus` 리스트를 사용한다.
- helper script는 이 skill 디렉터리 기준 `scripts/` 아래에 있다. 다른 skill에서 호출할 때는 해당 skill의 `SKILL.md` 위치 기준 `../work-log-wrap-up/scripts/...` 경로를 사용한다.
- Codex Desktop Automations로 실행할 때는 automation의 writable workspace/cwds에 Obsidian Vault 경로도 포함해야 한다. Vault가 빠지면 work-log 파일 생성이 `Operation not permitted`로 실패한다.

공통 수집 스크립트:

```bash
python3 ./scripts/collect_work_log_context.py daily 2026-06-01
python3 ./scripts/collect_work_log_context.py weekly 2026-06-01
python3 ./scripts/collect_work_log_context.py monthly 2026-05
```

스크립트는 raw context를 Markdown으로 출력한다. 에이전트는 이 결과를 읽고 중복을 제거한 뒤, 기존 work log 포맷에 맞게 한국어로 요약한다.

Apple Reminders:

```bash
python3 ./scripts/apple_reminders.py suggest-from-worklog ~/Documents/Obsidian\ Vault/notes/work-log/daily/2026-06-01.md
python3 ./scripts/apple_reminders.py create-from-worklog ~/Documents/Obsidian\ Vault/notes/work-log/daily/2026-06-01.md --due 2026-06-02 --list "Daily Focus"
```

`create-from-worklog`는 각 active reminder의 notes에 work log 링크를 넣고, 같은 리스트의 completed 항목에 `Daily Work Log - YYYY-MM-DD` reminder를 생성/갱신한다. 이 completed reminder는 생성된 문서를 다시 열기 위한 링크 보관용이다.

Microsoft To Do는 Azure/Graph 인증이 필요하므로 자동화 기본 대상으로 사용하지 않는다. 개인 Microsoft 계정에서 tenant mismatch/AADSTS90072가 나면 더 진행하지 말고 Apple Reminders를 사용한다. 아래 helper는 사용자가 명시적으로 Microsoft To Do를 다시 원할 때만 보조 옵션으로 쓴다:

```bash
python3 ./scripts/microsoft_todo.py status
python3 ./scripts/microsoft_todo.py suggest-from-worklog ~/Documents/Obsidian\ Vault/notes/work-log/daily/2026-06-01.md
python3 ./scripts/microsoft_todo.py create-from-worklog ~/Documents/Obsidian\ Vault/notes/work-log/daily/2026-06-01.md --due 2026-06-02 --list "Daily Focus"
```

인증은 `MS_TODO_ACCESS_TOKEN` 또는 `MS_TODO_CLIENT_ID` 기반 device-code OAuth token cache를 사용한다. 장기 사용 시에는 올바른 tenant에서 만든 App Registration client ID가 필요하다.

```bash
python3 ./scripts/microsoft_todo.py login
python3 ./scripts/microsoft_todo.py status
```

수동 실행이나 automation에서 환경변수가 필요하면 `VAULT_ROOT`, `CLAUDE_HOME`, `CODEX_HOME`, `WORK_LOG_SESSION_SOURCES`, `WORK_LOG_TZ`, `REMINDERS_LIST_NAME`을 실행 환경에 맞게 설정한다.
