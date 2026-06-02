# Work Logger

Claude Code와 Codex 세션, Obsidian 문서, 미팅 노트를 모아 `notes/work-log` 아래에 daily/weekly/monthly 업무 로그를 생성하는 Supergate 팀용 플러그인입니다. Daily 실행은 생성된 work-log의 다음 할 일 후보를 Apple Reminders `Daily Focus` 리스트에도 넣을 수 있습니다.

## Skills

- `daily-work-logger`: 어제 또는 지정일의 work log 생성
- `weekly-work-logger`: 전주 월요일~일요일 daily work log를 주간 요약으로 집계
- `monthly-work-logger`: 전월 weekly work log를 월간 요약으로 집계
- `weekly-work-summary`: `weekly-work-logger` 호환 이름
- `monthly-work-summary`: `monthly-work-logger` 호환 이름
- `work-log-wrap-up`: 공통 수집/Reminders helper skill
- `learning-tracker`: 학습 내용을 TIL 문서로 정리

## Sources

Daily routine은 기본적으로 아래 소스를 읽습니다.

- Claude sessions: `~/.claude/projects/**/*.jsonl`
- Codex sessions: `~/.codex/sessions`, `~/.codex/archived_sessions`, `~/.codex/session_index.jsonl`
- Obsidian vault: `$VAULT_ROOT` 또는 `~/Documents/Obsidian Vault`
- Meeting notes: vault 안의 `notes/dailies/YYYY-MM-DD*.md`와 날짜가 포함된 markdown 문서

Things는 더 이상 기본 소스로 사용하지 않습니다.

## Environment

필수는 아니지만 팀원별 경로 차이가 있으면 아래 환경변수를 설정하세요.

```bash
export VAULT_ROOT="$HOME/Documents/Obsidian Vault"
export CLAUDE_HOME="$HOME/.claude"
export CODEX_HOME="$HOME/.codex"
export WORK_LOG_SESSION_SOURCES="claude,codex"
export WORK_LOG_TZ="Asia/Seoul"
export REMINDERS_LIST_NAME="Daily Focus"
```

출력 경로는 다음과 같습니다.

```text
$VAULT_ROOT/notes/work-log/daily/YYYY-MM-DD.md
$VAULT_ROOT/notes/work-log/weekly/YYYY-MM-WN.md
$VAULT_ROOT/notes/work-log/monthly/YYYY-MM.md
```

## Manual Usage

```bash
/daily-work-logger
/daily-work-logger 2026-06-01
/weekly-work-logger
/monthly-work-logger 2026-05
```

helper script를 직접 smoke test할 때는 이 디렉터리 기준으로 실행할 수 있습니다.

```bash
python3 skills/work-log-wrap-up/scripts/collect_work_log_context.py daily 2026-06-01 --max-items 5
python3 skills/work-log-wrap-up/scripts/apple_reminders.py suggest-from-worklog "$VAULT_ROOT/notes/work-log/daily/2026-06-01.md"
```

## Automations

팀원이 자동 실행을 원하면 각자 Codex Desktop Automations, launchd, 또는 Claude/Codex CLI runner에 아래 세 작업을 등록합니다. Codex Desktop Automations를 쓰는 경우 workspace/cwds 설정에 특히 주의하세요.

| Name | Schedule | Prompt |
| --- | --- | --- |
| Daily Work Logger | 매일 09:00 | `Run the daily work-log routine for yesterday in Asia/Seoul. Use the daily-work-logger skill. Collect Claude and Codex sessions, Obsidian documents, and meeting notes; write or update the daily work log in the Obsidian vault; then create 3 to 5 Apple Reminders in the Daily Focus list for today. Ensure each reminder notes field includes the generated work-log Obsidian link, file URL, and path, and create or update a completed Daily Work Log link reminder in the same list.` |
| Weekly Work Logger | 매주 월요일 09:00 | `Run the weekly work-log routine for the previous Monday through Sunday in Asia/Seoul. Use the weekly-work-logger skill and write or update the weekly summary in the Obsidian vault.` |
| Monthly Work Logger | 매월 1일 09:00 | `Run the monthly work-log routine for the previous month in Asia/Seoul. Use the monthly-work-logger skill and write or update the monthly summary in the Obsidian vault.` |

Automation workspace/cwds에는 최소한 아래 경로를 포함해야 합니다.

- Obsidian vault path, 예: `~/Documents/Obsidian Vault`
- 선택적으로 dotfiles 또는 이 플러그인 저장소

Vault 경로가 writable workspace에 없으면 daily/weekly/monthly 파일 생성이 macOS sandbox에서 `Operation not permitted`로 실패할 수 있습니다.

## Apple Reminders

Daily routine은 `## 다음 할 일`, `Action Items`, `미완료 작업` 등에서 3~5개를 추출해 `Daily Focus` 리스트에 오늘 due date로 생성합니다. 각 reminder notes에는 다음 링크가 들어갑니다.

- Obsidian URI
- `file://` URL
- filesystem path

또한 completed reminder `Daily Work Log - YYYY-MM-DD`를 생성 또는 갱신해 완료 목록에서도 work-log 문서를 찾을 수 있게 합니다.

macOS에서 처음 실행할 때 Reminders 자동화 권한 요청이 뜰 수 있습니다.

## Microsoft To Do

Microsoft To Do는 기본값이 아닙니다. Graph API OAuth와 Azure App Registration이 필요하고 tenant mismatch가 잦기 때문에, 팀 공용 기본은 Apple Reminders입니다. 꼭 필요한 경우에만 `skills/work-log-wrap-up/scripts/microsoft_todo.py`를 사용하세요.
