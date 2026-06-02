---
name: learning-tracker
description: |
  Claude/Codex 세션과 Obsidian 문서에서 새로운 기술/라이브러리/개념 학습 내용을 추출하여 TIL(Today I Learned) 형태로 정리할 때 사용.
  "학습 정리", "TIL", "오늘 배운 것", "learning" 등의 요청 시 사용.
argument-hint: "[YYYY-MM-DD]"
user_invocable: true
---

# Learning Tracker

work-log 루틴과 같은 소스를 사용해 학습 내용을 추출한다. Claude Code와 Codex 세션을 모두 기본 소스로 사용한다.

## 실행

1. 대상 날짜를 정한다. 인수가 없으면 Asia/Seoul 기준 어제다.
2. 공통 수집 스크립트를 실행한다. 아래 script 경로는 이 skill 디렉터리 기준 상대 경로다.

```bash
python3 ../work-log-wrap-up/scripts/collect_work_log_context.py daily {TARGET_DATE}
```

3. `AI Sessions (Claude/Codex)`, `Vault Documents`, `Meeting Notes`에서 학습 후보를 추출한다.
4. 중복을 제거하고 TIL 형식으로 정리한다.

## 학습 감지 기준

한국어 키워드:

- 배웠, 알게, 처음, 새로운, 이해, 몰랐
- 뭐야, 어떻게, 왜, 차이

영어 키워드:

- TIL, learned, discovered, first time
- What, How, Why

기술 지표:

- 새 라이브러리, CLI, API, 프레임워크를 처음 다룸
- 공식 문서나 외부 자료를 확인하고 사용법을 정리함
- 에러나 장애를 해결하면서 재사용 가능한 원인을 파악함
- 아키텍처, 운영, 보안, 배포 방식에 대한 새 결정을 내림

## 출력

Daily work-log에 넣을 때:

```markdown
### 학습 기록

#### 기술/도구
- **도구명**: 학습 내용

#### 개념
- **개념명**: 이해한 내용

#### 해결방법
- **문제**: 해결 방법 요약
```

독립 실행 TIL로 정리할 때:

```markdown
---
date: {TARGET_DATE}
type: til
tags: [til, learning]
---

# TIL - {TARGET_DATE}

## 오늘 배운 것

1. **주제**
   - 핵심 내용
   - 재사용 가능한 맥락

## 더 볼 것

- [ ] 후속 확인 항목
```

TIL 파일을 새로 만들 경우 기본 위치는 `$VAULT_ROOT/notes/dailies/` 또는 사용자가 지정한 TIL 디렉터리를 따른다. 기존 프로젝트에서 다른 TIL 위치가 확인되면 그 패턴을 우선한다.

## 에러 처리

- Claude/Codex 세션 없음: Vault 문서와 미팅 노트만 보고 정리한다.
- 학습 내용 없음: "해당 날짜에 특별한 학습 기록 없음"으로 반환한다.
- 수동 편집된 TIL 파일이 있으면 덮어쓰기 전에 확인한다.

## 관련 Skill

- `daily-work-logger`: daily work-log 생성
- `work-log-wrap-up`: 공통 수집 로직과 helper scripts
