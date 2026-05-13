# Henry Jung Project

## 하네스: 수지구 아파트 가격 추이 및 매매 시점 분석

**목표:** 경기도 용인시 수지구(풍덕천·죽전·상현·성복·신봉·동천) 아파트의 가격 추이를 신뢰 출처에서 수집·분석하고, 시나리오 기반 매매 시점 의사결정 보고서를 산출한다.

**트리거:** 수지구 또는 수지구 행정동의 아파트 가격·매매 시점 관련 요청 시 `suji-apt-analysis-orchestrator` 스킬을 사용하라. 한 줄 시세 조회는 직접 응답해도 무방하나, 분석·추천·보고서 요청은 반드시 오케스트레이터를 통해 처리한다.

**구성:**
- 에이전트 4명 (`price-collector`, `context-analyst`, `timing-strategist`, `report-writer`)
- 스킬 4개 + 오케스트레이터 1개 (`suji-price-data`, `real-estate-market-context`, `apt-buy-sell-timing`, `real-estate-report`, `suji-apt-analysis-orchestrator`)
- 실행 모드: 에이전트 팀 (팬아웃 → 파이프라인 하이브리드)
- 모든 에이전트는 `model: "opus"`

**산출물 경로:** 중간 산출물은 `_workspace/`, 최종 보고서는 사용자 지정 경로 또는 `report_suji_apt_YYYY-MM-DD.md`.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-05-13 | 초기 구성 | 전체 (agents 4개, skills 5개, CLAUDE.md) | - |
