---
name: suji-apt-analysis-orchestrator
description: 경기도 용인시 수지구(또는 풍덕천·죽전·상현·성복·신봉·동천) 아파트의 가격 추이를 조사하고 매매 시점을 분석할 때 반드시 사용. 데이터 수집·시장 환경·매매 전략·보고서 작성 전체 워크플로우를 자동으로 실행한다. "수지 아파트 분석", "수지 시세", "수지구 매매 타이밍", "용인 아파트 언제 사", "보고서 다시 작성", "수지 분석 업데이트", "이전 결과 보완" 등의 표현이 등장하면 활용. 단순 시세 조회 한 줄 답변이 아닌, 가격 추이 + 시장 환경 + 매매 시점 종합 분석이 요구되는 모든 요청에 적용.
---

# 수지구 아파트 매매 분석 — 오케스트레이터

## 트리거 조건

- 사용자가 경기도 용인시 수지구(또는 행정동: 풍덕천/죽전/상현/성복/신봉/동천)의 아파트 가격·매매 시점을 묻거나 분석을 요청할 때
- "수지 아파트 분석", "수지 시세 조사", "용인 아파트 언제 매매", "수지 매매 타이밍", "수지구 가격 추이" 같은 표현
- 후속 작업: "다시 실행", "재실행", "보고서 업데이트", "수지 분석 수정", "○ 시나리오 보완", "이전 결과 기반으로"
- 단순 한 줄 답변이 아니라 종합 분석/보고서 산출이 필요한 경우

## 워크플로우 개요

**실행 모드:** 에이전트 팀 (TeamCreate + SendMessage + TaskCreate)

```
Phase 0: 컨텍스트 확인 (초기/후속/부분 재실행 판별)
       ↓
Phase 1: 팀 구성 + 작업 할당
       ↓
Phase 2 (병렬): price-collector + context-analyst
       ↓
Phase 3 (직렬): timing-strategist  ← Phase 2 산출물 활용
       ↓
Phase 4 (직렬): report-writer       ← Phase 2+3 산출물 종합
       ↓
Phase 5: 검토 및 사용자 전달
       ↓
Phase 6: 피드백 수집 → 필요 시 진화
```

## Phase 0: 컨텍스트 확인

워크플로우 시작 시 반드시 다음을 확인:

1. `_workspace/` 디렉토리 존재 여부 확인
2. 존재할 경우:
   - 사용자 요청이 "부분 수정"인지 판단 (예: "보고서만 다시", "시나리오 비관 부분만 보완")
     → **부분 재실행**: 해당 에이전트만 다시 호출
   - 사용자 요청이 "새 분석"인지 판단 (예: "다른 기간으로 재조사", "○○ 단지로 재분석")
     → **새 실행**: 기존 `_workspace/`를 `_workspace_prev_YYYY-MM-DD/`로 이동 후 새 워크플로우 시작
   - 사용자 요청이 "결과 업데이트"인 경우 (예: "최신 데이터로 갱신")
     → **점진 갱신**: 각 에이전트가 기존 파일을 읽고 갱신
3. 존재하지 않을 경우:
   - **초기 실행**: 전체 워크플로우 시작

## Phase 1: 팀 구성 + 작업 할당

```
TeamCreate({
  team_name: "suji-apt-analysis",
  members: ["price-collector", "context-analyst", "timing-strategist", "report-writer"]
})
```

각 에이전트 Agent 호출 시 반드시 `model: "opus"` 명시.

각 에이전트에 TaskCreate로 작업 할당:
- Task 1: price-collector → 수지구 아파트 가격·거래량·전세가율 데이터 수집
- Task 2: context-analyst → 거시·지역 환경 분석
- Task 3: timing-strategist → 시나리오 매트릭스 작성 (blockedBy: Task 1, Task 2)
- Task 4: report-writer → 최종 보고서 작성 (blockedBy: Task 1, Task 2, Task 3)

## Phase 2: 병렬 수집·분석

- price-collector와 context-analyst를 동시에 가동
- 두 에이전트는 SendMessage로 거래량·공급량 자료를 서로 요청 가능
- 산출물:
  - `_workspace/01_price_collector_data.md`
  - `_workspace/02_context_analyst_factors.md`

**모니터링**: 30분 이상 응답 없으면 leader가 상태 점검 메시지 발송

## Phase 3: 매매 시점 전략

- timing-strategist가 두 산출물을 읽고 시나리오 매트릭스 작성
- 데이터 공백 발견 시 SendMessage로 해당 에이전트에 보완 요청 (재시도 1회까지)
- 산출물: `_workspace/03_timing_strategist_scenarios.md`

## Phase 4: 보고서 작성

- report-writer가 세 산출물을 종합하여 최종 보고서 작성
- 출력 경로: 사용자 지정이 있으면 그곳, 없으면 `report_suji_apt_YYYY-MM-DD.md`

## Phase 5: 검토 및 전달

- 리더(오케스트레이터)는 최종 보고서를 검토:
  - 단정적 표현(반드시/절대) 사용 여부
  - 시점 명시 여부
  - 시나리오 분기 일관성
  - 출처 부록 완전성
- 사용자에게 보고서 경로와 핵심 요약(3~5줄) 전달

## Phase 6: 피드백 수집

전달 직후 사용자에게:
- "결과에서 추가로 보완할 부분이 있나요?"
- "특정 단지·시나리오에 대한 심화 분석이 필요하면 알려주세요."

피드백 유형에 따라:
- 결과물 품질 → 해당 에이전트의 스킬 수정 (이후 진화 트리거)
- 누락 데이터 → price-collector 재호출
- 시나리오 보강 → timing-strategist 재호출
- 톤·구조 변경 → report-writer 재호출

## 데이터 전달 프로토콜

| 전략 | 적용 |
|------|------|
| **태스크 기반** | 진행 상태 추적, 의존 관계 관리 (TaskCreate/Update) |
| **파일 기반** | `_workspace/01~03_*` 파일로 산출물 전달 |
| **메시지 기반** | 자료 공유 요청, 모순 발견 시 즉시 알림 (SendMessage) |

## 에러 핸들링

| 에러 유형 | 대응 |
|----------|------|
| 데이터 출처 접근 실패 | 1회 재시도 → 대체 출처 사용 → 누락 명시 |
| 에이전트 응답 시간 초과 (30분+) | 상태 점검 메시지 → 재할당 검토 |
| 산출물 누락 (입력 파일 부재) | 해당 에이전트에 SendMessage로 재작업 요청 1회, 그래도 누락이면 보고서에 명시 |
| 출처 간 수치 모순 | 모두 병기 + 사유 추정 (어느 쪽도 임의 채택 금지) |
| 사용자 의도 모호 | 사용자에게 명시적 확인 요청 — 단, "스탑 없이" 모드일 때는 베이스 가정으로 진행 후 보고서에 가정 명시 |

## 테스트 시나리오

**정상 흐름 (초기 실행):**
1. 사용자: "수지구 아파트 가격 추이를 조사하고 언제 매매하면 좋을지 분석해줘"
2. Phase 0: `_workspace/` 없음 확인 → 초기 실행
3. Phase 1: 팀 구성, 4개 task 생성
4. Phase 2: price-collector + context-analyst 병렬 가동, 2개 산출물 생성
5. Phase 3: timing-strategist가 시나리오 매트릭스 작성
6. Phase 4: report-writer가 보고서 작성 → `report_suji_apt_2026-05-13.md`
7. Phase 5: 리더 검토 후 사용자에게 경로와 요약 전달
8. Phase 6: 피드백 요청

**에러 흐름 (데이터 출처 접근 실패):**
1. price-collector가 국토부 실거래가 시스템 접근 실패
2. 재시도 1회 → 실패
3. KB부동산 등 대체 출처로 전환, 일부 데이터 부재 명시
4. context-analyst의 공급량 분석에 영향 → SendMessage로 알림
5. timing-strategist는 데이터 부재를 시나리오 평가에 반영 ("낮은 확신도" 라벨)
6. report-writer는 부록에 데이터 한계 섹션 추가

## 산출물 체크리스트

워크플로우 종료 시 다음을 확인:
- [ ] `_workspace/01_price_collector_data.md` 존재 및 핵심 표 포함
- [ ] `_workspace/02_context_analyst_factors.md` 존재 및 영향 매트릭스 포함
- [ ] `_workspace/03_timing_strategist_scenarios.md` 존재 및 3개 시나리오 포함
- [ ] 최종 보고서 파일 존재 및 6개 섹션 모두 포함
- [ ] 보고서에 조회 시점·출처·한계 명시
- [ ] 단정적 표현 미사용

## 진화 트리거 (Phase 7 연동)

다음 상황에서 사용자에게 하네스 수정을 제안:
- 같은 유형의 피드백이 2회 이상 반복될 때
- 특정 에이전트가 반복적으로 실패하는 패턴
- 사용자가 오케스트레이터를 우회해서 수동으로 작업하는 패턴 발견

수정 시 CLAUDE.md의 변경 이력 테이블에 반드시 기록.
