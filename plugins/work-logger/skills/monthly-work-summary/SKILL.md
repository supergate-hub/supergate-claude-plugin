---
name: monthly-work-summary
description: |
  weekly-work-summary가 생성한 주간 요약 문서들을 월 단위로 집계하여 월간 업무 요약 문서 생성.
  서브 에이전트 기반 병렬 처리로 메인 컨텍스트 절약.
  "월간 요약", "monthly summary", "이번 달 정리", "한 달 요약" 등의 요청 시 자동 적용.
---

# Monthly Work Summary Skill

## 개요

weekly-work-summary가 생성한 주간 요약(`work-log/weekly/YYYY-MM-WN-summary.md`)을 **전월** 범위로 수집하여 월간 업무 요약 문서를 생성하는 skill.

> **핵심**: Daily Notes를 직접 읽지 않고 **weekly summary를 입력**으로 사용. 이미 정제된 데이터를 집계하므로 컨텍스트 효율적.

## 파이프라인

```
daily-work-logger (매일)
        ↓
    work-log/daily/YYYY-MM-DD.md
        ↓
weekly-work-summary (매주)
        ↓
    work-log/weekly/YYYY-MM-WN-summary.md
        ↓
monthly-work-summary (매월) ← 이 skill
        ↓
    work-log/monthly/YYYY-MM-summary.md
```

## 핵심 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                    Main Agent (Orchestrator)                 │
│  - 대상 월 결정 및 weekly 파일 수집 (Phase 1)                   │
│  - 서브 에이전트 병렬 실행 (Phase 2)                            │
│  - 결과 통합 및 월간 요약 작성 (Phase 3)                        │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
    ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
    │ SubAgent 1  │  │ SubAgent 2  │  │ SubAgent 3  │
    │ 성과/업무    │  │ 기술 성장    │  │ 회고/계획    │
    │ Aggregator  │  │ Aggregator  │  │ Aggregator  │
    └─────────────┘  └─────────────┘  └─────────────┘
         │               │               │
         └───────────────┼───────────────┘
                         ▼
              ┌─────────────────┐
              │ 월간 요약 작성    │
              │ (Main Agent)    │
              └─────────────────┘
```

## 인수 (Arguments)

| 인수 | 설명 | 기본값 |
|------|------|--------|
| 월 | 분석할 월 (YYYY-MM 형식) | 전월 |

**사용 예시**:
- `/monthly-work-summary` — 전월 자동 분석
- `/monthly-work-summary 2026-03` — 2026년 3월 분석

## 실행 시점

- **실행**: 매월 1일 (또는 필요 시)
- **대상 기간**: 전월 1일 ~ 말일
- **출력**: `work-log/monthly/YYYY-MM-summary.md`

## 경로 정보

| 항목 | 경로 |
|------|------|
| vault | `~/Documents/Obsidian Vault/` |
| weekly | `~/Documents/Obsidian Vault/notes/work-log/weekly/` |
| 출력 | `~/Documents/Obsidian Vault/notes/work-log/monthly/` |

## 입력 소스

| 소스 | 경로 | 파일 패턴 |
|------|------|----------|
| Weekly Summaries | `notes/work-log/weekly/` | `YYYY-MM-WN-summary.md` |

---

## 실행 절차

### Phase 1: 초기화 (메인 에이전트 - 순차)

1. **대상 월 결정**

```bash
# 인수 없으면 전월 계산
if [ -z "$1" ]; then
  TARGET_MONTH=$(date -v-1m +%Y-%m)
else
  TARGET_MONTH="$1"  # 예: 2026-03
fi

YEAR=$(echo "$TARGET_MONTH" | cut -d'-' -f1)
MONTH=$(echo "$TARGET_MONTH" | cut -d'-' -f2)
MONTH_NUM=$(echo "$MONTH" | sed 's/^0//')

echo "대상: ${YEAR}년 ${MONTH_NUM}월"
```

2. **Weekly Summary 파일 수집**

```bash
WEEKLY_DIR="$HOME/Documents/Obsidian Vault/notes/work-log/weekly"

# 해당 월의 weekly summary 찾기 (YYYY-MM-WN-summary.md 패턴)
ls "$WEEKLY_DIR"/${TARGET_MONTH}-W*-summary.md 2>/dev/null
```

3. **출력 경로 확인**
```bash
OUTPUT_DIR="$HOME/Documents/Obsidian Vault/notes/work-log/monthly"
mkdir -p "$OUTPUT_DIR"
OUTPUT_FILE="$OUTPUT_DIR/${TARGET_MONTH}-summary.md"
```

---

### Phase 2: 서브 에이전트 병렬 실행 ★

> **중요**: 아래 3개의 Task를 **단일 메시지에서 동시에 호출**하여 병렬 실행합니다.
> 비용/속도 최적화를 위해 **haiku 모델**을 사용합니다.

---

#### SubAgent 1: 성과/업무 Aggregator

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "월간 성과/업무 집계" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({TARGET_MONTH}, {MONTH_NUM}, {FILE_LIST} 치환 필요):**

```
당신은 업무 성과 분석 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONTH_NUM}월의 주간 업무 요약들을 읽고 월간 성과와 업무를 집계합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. 각 주간 요약의 "주요 성과"와 "프로젝트별 업무" 섹션을 통합

## 집계 기준
- 프로젝트 단위로 월간 진행 상황 요약 (시작 → 현재 상태)
- 주간 반복 작업은 하나로 병합하고 빈도 표시
- 완료된 마일스톤과 미완료 항목 명확히 구분
- 월간 TOP 5 성과 선정

## 출력 형식 (마크다운으로 반환)

### 월간 TOP 성과
1. 성과 1
2. 성과 2
3. ...

### 프로젝트별 월간 진행

#### [프로젝트명]
- **상태**: 진행 중 / 완료 / 보류
- **주요 진척**: ...
- **남은 작업**: ...

### 월간 업무 통계
| 항목 | 수치 |
|------|------|
| 활동 주 | N주 |
| 진행 프로젝트 수 | N개 |
| 완료 마일스톤 | N건 |

(Weekly Summary가 없으면 "해당 월에 주간 요약 없음" 반환)
```

---

#### SubAgent 2: 기술 성장 Aggregator

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "월간 기술 성장 집계" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({TARGET_MONTH}, {MONTH_NUM}, {FILE_LIST} 치환 필요):**

```
당신은 기술 성장 분석 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONTH_NUM}월의 주간 요약들에서 기술 학습과 성장 내용을 월간으로 집계합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. 각 주간 요약의 "기술 & 학습" 섹션을 통합

## 집계 기준
- 학습한 기술/도구를 카테고리별로 분류 (인프라, 백엔드, 프론트엔드, AI/ML, DevOps 등)
- 월간 기술 스택 변화 추적 (새로 도입 / 심화 / 검토만)
- 해결한 기술 문제를 난이도별로 분류
- 작성한 기술 문서 전체 목록
- 오픈소스 기여 내역 정리

## 출력 형식 (마크다운으로 반환)

### 월간 기술 학습 맵

#### 인프라/클라우드
- [기술명]: 학습 수준 (입문/심화/실무 적용)

#### 백엔드/프론트엔드
- ...

#### AI/ML
- ...

### 해결한 주요 기술 문제
1. [문제]: 해결 방법 (주차)
2. ...

### 작성 기술 문서
- [[문서명]]: 주제 (주차)

### 오픈소스 기여
- [프로젝트명]: 기여 내용

### 월간 기술 스택 요약
- **새로 도입**: ...
- **심화 학습**: ...
- **검토/리서치**: ...

(학습 내용이 없으면 "해당 월에 기술 학습 기록 없음" 반환)
```

---

#### SubAgent 3: 회고/계획 Aggregator

**Task 호출 파라미터:**
| 파라미터 | 값 |
|---------|-----|
| description | "월간 회고 및 다음 달 계획" |
| subagent_type | "general-purpose" |
| model | "haiku" |

**프롬프트 ({TARGET_MONTH}, {MONTH_NUM}, {FILE_LIST} 치환 필요):**

```
당신은 프로젝트 회고 및 계획 전문가입니다. 코드를 작성하지 말고 분석만 수행하세요.

## 작업
{MONTH_NUM}월의 주간 요약들에서 이슈, 패턴, 다음 달 계획을 도출합니다.

## 대상 파일
{FILE_LIST}

## 실행 단계
1. 위 파일들을 Read 도구로 모두 읽기
2. 각 주간 요약의 "이슈 & 다음 주 계획" 섹션을 종합 분석

## 분석 기준
- 매주 carry-over된 작업 중 월말까지 미해결인 항목 추출
- 반복 등장하는 이슈/블로커 패턴 식별
- 주간 Next Action 제안 중 실제 실행된 것 vs 안 된 것 비교
- 월간 업무 패턴 분석 (집중도, 분산도, 병렬 프로젝트 수)

## 출력 형식 (마크다운으로 반환)

### 월간 회고

#### 잘한 점 (Keep)
- ...

#### 개선할 점 (Problem)
- ...

#### 시도할 것 (Try)
- ...

### 미해결 이슈 (월말 기준)
- [ ] 이슈 내용 (최초 언급 주차 → 현재 상태)

### 월간 패턴 분석
- **병렬 프로젝트 수**: 평균 N개/주
- **가장 많은 시간 투입**: [프로젝트명]
- **가장 오래 carry-over된 작업**: [작업명] (N주 연속)

### 다음 달 제안
- [ ] 우선순위 1: ... (근거: ...)
- [ ] 우선순위 2: ... (근거: ...)
- [ ] 우선순위 3: ... (근거: ...)

(이슈가 없으면 "해당 월에 특이 이슈 없음" 반환)
```

---

### Phase 3: 결과 통합 및 월간 요약 작성 (메인 에이전트)

1. **3개 서브 에이전트 결과 수집**

2. **월간 요약 문서 작성**

Write 도구를 사용하여 `work-log/monthly/{TARGET_MONTH}-summary.md` 생성:

```markdown
---
id: {TARGET_MONTH}-summary
aliases:
  - {YEAR}년 {MONTH_NUM}월 업무 요약
tags:
  - work-log/monthly
  - work-log/summary
created_at: {TODAY}
period: {TARGET_MONTH}
related: []
---

# 월간 업무 요약 — {YEAR}년 {MONTH_NUM}월

> **기간**: {TARGET_MONTH}-01 ~ {TARGET_MONTH}-{LAST_DAY} | **분석된 Weekly Summary**: {N}개

---

## 월간 TOP 성과

{SubAgent 1 결과 - TOP 성과}

---

## 프로젝트별 월간 진행

{SubAgent 1 결과 - 프로젝트별 진행}

---

## 기술 성장

{SubAgent 2 결과}

---

## 월간 회고 (KPT)

{SubAgent 3 결과 - 회고}

---

## 미해결 이슈 & 다음 달 계획

{SubAgent 3 결과 - 미해결 이슈 + 다음 달 제안}

---

## 월간 통계

{SubAgent 1 결과 - 업무 통계}

---

## Weekly Summaries

- [[{TARGET_MONTH}-W1-summary]]
- [[{TARGET_MONTH}-W2-summary]]
- [[{TARGET_MONTH}-W3-summary]]
- [[{TARGET_MONTH}-W4-summary]]
- [[{TARGET_MONTH}-W5-summary]]  (있는 경우)
```

3. **완료 메시지 출력**
```
{YEAR}년 {MONTH_NUM}월 업무 요약이 생성되었습니다: work-log/monthly/{TARGET_MONTH}-summary.md
분석된 Weekly Summary: N개
```

---

## 병렬 실행 핵심 원칙

1. **단일 응답에서 3개 Task 동시 호출**
2. **haiku 모델 사용**: 비용/속도 최적화
3. **결과만 반환**: 마크다운 형식의 분석 결과 텍스트만
4. **메인 에이전트 역할 최소화**: 날짜 계산 → Task 호출 → 결과 조합

---

## 에러 처리

| 상황 | 처리 방식 |
|------|----------|
| 서브 에이전트 실패 | 해당 섹션 "분석 실패" 표시, 나머지 정상 반영 |
| Weekly Summary 0개 | "해당 월에 주간 요약이 없습니다. 먼저 /weekly-work-summary를 실행해주세요." 안내 |
| Weekly Summary 일부만 존재 | 존재하는 파일만 분석, "N개 주간 요약 분석됨" 표시 |
| monthly 폴더 없음 | 자동 생성 (`mkdir -p`) |
| 이미 존재하는 요약 | "이미 {MONTH_NUM}월 요약이 존재합니다. 덮어쓸까요?" 확인 |

---

## 의존 관계

```
daily-work-logger (매일)
        ↓
    work-log/daily/YYYY-MM-DD.md
        ↓
weekly-work-summary (매주)
        ↓
    work-log/weekly/YYYY-MM-WN-summary.md
        ↓
monthly-work-summary (매월) ← 이 skill
        ↓
    work-log/monthly/YYYY-MM-summary.md
```

---

## 관련 Skill

- `weekly-work-summary`: 주간 업무 요약 (이 skill의 **필수 입력**)
- `daily-work-logger`: 일일 작업 로그 (파이프라인 시작점)
- `weekly-newsletter`: 주간 뉴스레터 (보완적 외부 공유 콘텐츠)
