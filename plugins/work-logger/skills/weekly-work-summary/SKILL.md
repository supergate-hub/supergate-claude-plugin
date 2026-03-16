---
name: weekly-work-summary
description: |
  daily-work-logger가 생성한 Daily Note 1주일치(전주 월~일)를 분석하여 주간 업무 요약 문서 생성.
  서브 에이전트 기반 병렬 처리로 메인 컨텍스트 절약.
  "주간 업무 요약", "weekly summary", "이번 주 정리", "주간 요약" 등의 요청 시 자동 적용.
---

# Weekly Work Summary Skill

## 개요

daily-work-logger가 생성한 Daily Note(`dailies/YYYY-MM-DD.md`)를 **전주 월요일~일요일** 범위로 수집하여 주간 업무 요약 문서를 생성하는 skill.

## 핵심 아키텍처

> **서브 에이전트 기반 병렬 처리**로 메인 에이전트의 컨텍스트를 최소화합니다.

```
┌─────────────────────────────────────────────────────────────┐
│                    Main Agent (Orchestrator)                 │
│  - 날짜 범위 계산 (Phase 1)                                    │
│  - 서브 에이전트 병렬 실행 (Phase 2)                            │
│  - 결과 통합 및 주간 요약 작성 (Phase 3)                        │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ SubAgent 1  │  │ SubAgent 2  │  │ SubAgent 3  │
    │ 업무 요약    │  │ 학습/기술    │  │ 이슈/리스크  │
    │ Extractor   │  │ Extractor   │  │ Extractor   │
    └─────────────┘  └─────────────┘  └─────────────┘
         │               │               │
         └───────────────┼───────────────┘
                         ▼
              ┌─────────────────┐
              │ 주간 요약 작성    │
              │ (Main Agent)    │
              └─────────────────┘
```

## 인수 (Arguments)

| 인수 | 설명 | 기본값 |
|------|------|--------|
| 날짜 또는 주차 | 분석할 주 (YYYY-MM-DD 또는 YYYY-WXX) | 전주 |

**사용 예시**:
- `/weekly-work-summary` - 전주(월~일) 자동 분석
- `/weekly-work-summary 2026-W11` - 2026년 11주차 분석
- `/weekly-work-summary 2026-03-10` - 해당 날짜가 속한 주 분석

## 주차 정의

> **중요**: 이 스킬에서 주(week)는 **월요일~일요일** 기준입니다.

## 실행 시점

- **실행**: 매주 월요일 아침 (또는 필요 시)
- **대상 기간**: 전주 월요일 ~ 일요일 (7일간)
- **출력**: `work-log/weekly/YYYY-MM-WN-summary.md` (예: `2026-03-W2-summary.md` = 3월 2주차)

## 경로 정보

| 항목 | 경로 |
|------|------|
| vault | `~/Documents/Obsidian Vault/` |
| daily | `~/Documents/Obsidian Vault/notes/work-log/daily/` |
| 출력 | `~/Documents/Obsidian Vault/notes/work-log/weekly/` |

## 입력 소스

| 소스 | 경로 | 파일 패턴 |
|------|------|----------|
| Daily Notes | `notes/work-log/daily/` | `YYYY-MM-DD.md` |

---

## 실행 절차

### Phase 1: 초기화 (메인 에이전트 - 순차)

1. **주차 결정 및 날짜 범위 계산**

```bash
# 인수 없으면 전주 계산
if [ -z "$1" ]; then
  # 전주 월요일 = 이번 주 월요일 - 7일
  THIS_MONDAY=$(date -v-$(($(date +%u) - 1))d +%Y-%m-%d)
  MONDAY=$(date -j -v-7d -f "%Y-%m-%d" "$THIS_MONDAY" +%Y-%m-%d)
else
  # 인수가 YYYY-WXX 형식인 경우
  # ISO 주차의 월요일 구하기
  MONDAY=$(date -j -f "%G-W%V-%u" "${1}-1" +%Y-%m-%d 2>/dev/null)
  if [ -z "$MONDAY" ]; then
    # 인수가 YYYY-MM-DD 형식인 경우 해당 주 월요일 구하기
    DOW=$(date -j -f "%Y-%m-%d" "$1" +%u)
    MONDAY=$(date -j -v-$((DOW - 1))d -f "%Y-%m-%d" "$1" +%Y-%m-%d)
  fi
fi

# 일요일 = 월요일 + 6일
SUNDAY=$(date -j -v+6d -f "%Y-%m-%d" "$MONDAY" +%Y-%m-%d)

# 월 기준 주차 계산 (YYYY-MM-WN 형식)
MONTH=$(date -j -f "%Y-%m-%d" "$MONDAY" +%m)
YEAR=$(date -j -f "%Y-%m-%d" "$MONDAY" +%Y)
MONTH_NAME=$(date -j -f "%Y-%m-%d" "$MONDAY" +%-m)
# 해당 월의 첫 번째 월요일 기준 주차
FIRST_DAY=$(date -j -f "%Y-%m-%d" "${YEAR}-${MONTH}-01" +%u)
if [ "$FIRST_DAY" -le 1 ]; then
  FIRST_MONDAY="${YEAR}-${MONTH}-01"
else
  OFFSET=$((8 - FIRST_DAY))
  FIRST_MONDAY=$(date -j -v+${OFFSET}d -f "%Y-%m-%d" "${YEAR}-${MONTH}-01" +%Y-%m-%d)
fi
DAY_OF_MONTH=$(date -j -f "%Y-%m-%d" "$MONDAY" +%-d)
FIRST_MONDAY_DAY=$(date -j -f "%Y-%m-%d" "$FIRST_MONDAY" +%-d)
WEEK_OF_MONTH=$(( (DAY_OF_MONTH - FIRST_MONDAY_DAY) / 7 + 1 ))
WEEK_NUM="${YEAR}-${MONTH}-W${WEEK_OF_MONTH}"

echo "주차: ${MONTH_NAME}월 ${WEEK_OF_MONTH}주차 (${WEEK_NUM})"
echo "대상 기간: $MONDAY (월) ~ $SUNDAY (일)"
```

2. **Daily Note 파일 목록 수집**

```bash
DAILIES_DIR="$HOME/Documents/Obsidian Vault/notes/work-log/daily"

# 월~일 범위의 Daily Note 파일 찾기
for i in $(seq 0 6); do
  DATE=$(date -j -v+${i}d -f "%Y-%m-%d" "$MONDAY" +%Y-%m-%d)
  FILE="$DAILIES_DIR/$DATE.md"
  if [ -f "$FILE" ]; then
    echo "$FILE"
  fi
done
```

3. **출력 경로 확인**
```bash
OUTPUT_DIR="$HOME/Documents/Obsidian Vault/notes/work-log/weekly"
mkdir -p "$OUTPUT_DIR"
OUTPUT_FILE="$OUTPUT_DIR/${WEEK_NUM}-summary.md"
# 예: work-log/weekly/2026-03-W2-summary.md
```

---

### Phase 2: 서브 에이전트 병렬 실행 ★

> **중요**: 아래 3개의 Task를 **단일 메시지에서 동시에 호출**하여 병렬 실행합니다.
> 각 서브 에이전트는 분석 결과를 **마크다운 형식의 텍스트**로 반환합니다.
> 비용/속도 최적화를 위해 **haiku 모델**을 사용합니다.

---

#### SubAgent 1: 업무 요약 Extractor

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "주간 업무 요약 추출" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({MONDAY}, {SUNDAY}, {FILE_LIST} 치환 필요):**

```
당신은 업무 요약 분석 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONDAY} (월) ~ {SUNDAY} (일) 기간의 Daily Notes를 읽고 주간 업무 요약을 추출합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. 각 일자별로 "작업 내역" 섹션에서 핵심 업무를 추출

## 추출 기준
- 프로젝트/기능 단위로 그룹핑
- 반복되는 작업은 하나로 병합 (예: "매일 코드리뷰" → "코드리뷰 (5회)")
- 완료된 작업 vs 진행 중 작업 구분
- 주요 성과/마일스톤 강조

## 출력 형식 (마크다운으로 반환)

### 주요 성과
- 성과 1 (관련 날짜)
- 성과 2 (관련 날짜)

### 프로젝트별 업무

#### [프로젝트/영역명]
- 완료: 작업 내용
- 진행 중: 작업 내용

### 일별 요약
| 날짜 | 핵심 업무 |
|------|----------|
| 월 | ... |
| 화 | ... |
| ... | ... |

(Daily Notes가 없으면 "해당 주에 Daily Notes 없음" 반환)
```

---

#### SubAgent 2: 학습/기술 Extractor

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "주간 학습/기술 추출" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({MONDAY}, {SUNDAY}, {FILE_LIST} 치환 필요):**

```
당신은 기술 학습 분석 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONDAY} (월) ~ {SUNDAY} (일) 기간의 Daily Notes에서 학습/기술 관련 내용을 추출합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. "학습 기록", "Claude Code 작업", "Vault 문서 작업" 섹션에서 기술 내용 추출

## 추출 대상
- 새로 배운 기술/도구/라이브러리
- 해결한 기술 문제와 해결 방법
- 작성/수정한 기술 문서 목록
- 코드 작성/리팩토링 내역
- 아키텍처 결정사항

## 출력 형식 (마크다운으로 반환)

### 기술 학습
| 기술/도구 | 내용 | 날짜 |
|----------|------|------|
| ... | ... | ... |

### 해결한 기술 문제
- **[문제]**: 해결 방법 (날짜)

### 작성/수정 문서
- [[문서명]]: 내용 요약

### 사용 기술 스택 요약
- 언어/프레임워크: ...
- 도구: ...

(학습 내용이 없으면 "해당 주에 학습 기록 없음" 반환)
```

---

#### SubAgent 3: 이슈/리스크/Next Action Extractor

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "주간 이슈/다음 작업 추출" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({MONDAY}, {SUNDAY}, {FILE_LIST} 치환 필요):**

```
당신은 프로젝트 관리 분석 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONDAY} (월) ~ {SUNDAY} (일) 기간의 Daily Notes에서 이슈, 블로커, 다음 주 할 일을 추출합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. 전체 내용에서 이슈/리스크/다음 작업 패턴을 탐지

## 추출 대상
- 미완료/보류된 작업
- 반복적으로 언급되지만 해결되지 않은 이슈
- 블로커/의존성
- 미팅에서 나온 Action Items
- "다음에", "나중에", "TODO", "해야 할" 등의 키워드

## 출력 형식 (마크다운으로 반환)

### 미완료 작업 (Carry-over)
- [ ] 작업 내용 (최초 언급 날짜)

### 이슈/블로커
- **[이슈]**: 상태 및 영향

### 미팅 Action Items
- [ ] Action Item (미팅명, 날짜)

### 다음 주 제안 (Next Actions)
- [ ] 추천 작업 1 (근거: ...)
- [ ] 추천 작업 2 (근거: ...)

(이슈가 없으면 "해당 주에 특이 이슈 없음" 반환)
```

---

### Phase 3: 결과 통합 및 주간 요약 작성 (메인 에이전트)

1. **3개 서브 에이전트 결과 수집**

2. **주간 요약 문서 작성**

Write 도구를 사용하여 `work-log/weekly/{WEEK_NUM}-summary.md` 생성:

```markdown
---
id: {WEEK_NUM}-summary
aliases:
  - {YEAR}년 {MONTH_NAME}월 {WEEK_OF_MONTH}주차 업무 요약
tags:
  - work-log/weekly
  - work-log/summary
created_at: {TODAY}
period: {MONDAY} ~ {SUNDAY}
related: []
---

# 주간 업무 요약 — {MONTH_NAME}월 {WEEK_OF_MONTH}주차

> **기간**: {MONDAY} (월) ~ {SUNDAY} (일)

---

## 주요 성과

{SubAgent 1 결과 - 주요 성과 부분}

---

## 프로젝트별 업무

{SubAgent 1 결과 - 프로젝트별 업무 부분}

---

## 일별 요약

{SubAgent 1 결과 - 일별 요약 테이블}

---

## 기술 & 학습

{SubAgent 2 결과}

---

## 이슈 & 다음 주 계획

{SubAgent 3 결과}

---

## Daily Notes

- [[{MONDAY}]]
- [[{TUESDAY}]]
- [[{WEDNESDAY}]]
- [[{THURSDAY}]]
- [[{FRIDAY}]]
- [[{SATURDAY}]]
- [[{SUNDAY}]]
```

3. **완료 메시지 출력**
```
{MONTH_NAME}월 {WEEK_OF_MONTH}주차 업무 요약이 생성되었습니다: work-log/weekly/{WEEK_NUM}-summary.md
분석된 Daily Notes: N개 / 7일
```

---

## 병렬 실행 핵심 원칙

1. **단일 응답에서 3개 Task 동시 호출**: 메인 에이전트는 Phase 2에서 하나의 응답에 3개의 Task 도구 호출을 포함해야 합니다.

2. **haiku 모델 사용**: 비용과 속도 최적화를 위해 서브 에이전트는 haiku 모델을 사용합니다.

3. **결과만 반환**: 각 서브 에이전트는 마크다운 형식의 분석 결과 텍스트만 반환합니다.

4. **메인 에이전트 역할 최소화**:
   - Phase 1: 날짜 계산 + 파일 목록 수집만 수행
   - Phase 2: Task 호출만 수행 (분석 로직 없음)
   - Phase 3: 결과 조합 및 문서 작성만 수행

---

## 에러 처리

| 상황 | 처리 방식 |
|------|----------|
| 서브 에이전트 실패 | 해당 섹션을 "분석 실패"로 표시, 나머지 정상 반영 |
| Daily Notes 0개 | "해당 주에 Daily Notes가 없습니다. 먼저 /daily-work-logger를 실행해주세요." 안내 |
| Daily Notes 일부만 존재 | 존재하는 파일만 분석, "N/7일 분석됨" 표시 |
| weekly 폴더 없음 | 자동 생성 (`mkdir -p`) |
| 이미 존재하는 요약 | "이미 {WEEK_NUM} 요약이 존재합니다. 덮어쓸까요?" 확인 |

---

## 의존 관계

```
daily-work-logger (매일)
        ↓
    work-log/daily/YYYY-MM-DD.md
        ↓
weekly-work-summary (월요일) ← 이 skill
        ↓
    work-log/weekly/YYYY-WXX-summary.md
        ↓
weekly-newsletter (토요일, 선택)
        ↓
    newsletters/YYYY-WXX-newsletter.md
```

---

## 관련 Skill

- `daily-work-logger`: 일일 작업 로그 (이 skill의 **필수 입력**)
- `weekly-newsletter`: 주간 뉴스레터 (이 skill의 출력을 활용 가능)
- `weekly-claude-analytics`: Claude 사용 통계 (보완적 데이터)
