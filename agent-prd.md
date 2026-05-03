# Agent PRD — 암묵지 → 형식지 변환 에이전트

## 1. 한 줄 요약

특정 이벤트 정의 레포에서 기존 이벤트/프로퍼티 정보를 먼저 연결하고, BigQuery SQL에 숨어 있는 분석 활용 맥락을 보강해, 에이전트가 재사용하기 좋은 **프로퍼티 중심 후보 지식 카드(YAML)**와 **사람 검토용 Markdown**으로 변환한다.

## 2. 문제 정의

회사 내부에는 이벤트 정의, 프로퍼티 이름, 분석 SQL, 대시보드 쿼리, 문서, 사람의 설명이 흩어져 있다. 이벤트/프로퍼티 자체는 이벤트 레포나 GitHub Copilot을 통해 어느 정도 찾을 수 있지만, 특정 프로퍼티가 실제로 **어떤 지표 계산이나 분석 맥락에서 사용되는지**는 찾기 어렵다.

이로 인해 다음 문제가 생긴다.

- 같은 이벤트 프로퍼티 사용 맥락을 여러 사람이 반복 조사한다.
- downstream agent가 특정 프로퍼티 질문에 답하려면 매번 많은 SQL/문맥을 다시 읽어야 한다.
- SQL에서 추론한 맥락과 공식 지식의 경계가 불명확해진다.
- 지표 정의, 비즈니스 룰, 예외 조건이 코드 속 암묵지로 남는다.

## 3. 목표

MVP의 목표는 특정 이벤트 정의 레포를 출발점으로 삼아 다음 질문에 답할 수 있는 후보 지식을 만드는 것이다.

> “이 이벤트 프로퍼티는 어떤 지표/분석에서, 어떤 방식으로, 어떤 근거에 의해 사용되고 있는가?”

구체적으로는:

- 특정 이벤트 정의 레포에서 이벤트와 프로퍼티의 기본 정의를 가져온다.
- BigQuery SQL 하나를 입력받아 해당 이벤트/프로퍼티가 실제 분석에서 어떻게 쓰이는지 확인한다.
- SQL에서 이벤트명, 프로퍼티명, 사용 위치, 사용 역할을 추출한다.
- 기존 정의만으로 부족한 부분은 SQL 사용처를 근거로 어떤 지표/분석 맥락에 쓰이는지 후보 해석을 만든다.
- 출처, evidence, confidence, human-review 상태를 남긴다.
- YAML을 canonical 산출물로, Markdown을 사람 검토용 산출물로 만든다.
- 스키마는 나중에 여러 SQL 파일 전수 조사와 병합에 확장 가능해야 한다.

## 4. MVP 범위

### In scope

- 특정 이벤트 정의 레포 연결.
- 이벤트/프로퍼티 기본 정보 조회.
- BigQuery SQL 파일 또는 SQL 텍스트 1개 입력.
- 이벤트명과 프로퍼티 후보 추출.
- SQL 사용 역할 추출:
  - `SELECT`
  - `WHERE`
  - `JOIN`
  - `GROUP BY`
  - `ORDER BY`
  - `CASE WHEN`
  - aggregation
  - derived expression
- 프로퍼티 중심 knowledge card 생성.
- 지표/분석 context 역참조 생성.
- source reference, snippet reference, confidence, review status 포함.
- 사람이 검토해야 할 질문 생성.
- YAML + Markdown 파일 출력.
- 샘플/익명화 데이터 기반 설계.

### Out of scope

- Slack, 회의록, 위키, Confluence 자동 ingestion.
- 프로덕션 BigQuery 직접 연결.
- 임의의 모든 GitHub 레포 전체 자동 인덱싱. 단, 사용자가 지정한 이벤트 정의 레포 1개 연결은 MVP 전제에 포함한다.
- 승인 UI 구현.
- 자동 공식 지식 등록.
- BI/dashboard 도구 연동.
- SQL에서 추론한 내용을 공식 truth로 취급.
- 모든 SQL dialect 또는 복잡한 동적 SQL 완전 지원.

단, Confluence 문서 등 보조 자료는 사용자가 직접 제공하는 형태로 이후 확장할 수 있다.

## 5. 주요 사용자

| 사용자 | 니즈 |
| --- | --- |
| 데이터 분석가 | 특정 이벤트 프로퍼티가 어떤 지표/분석에서 쓰였는지 빠르게 확인 |
| 데이터 엔지니어 | SQL 속 지표/룰 후보를 구조화하여 재사용 가능하게 정리 |
| PM/비즈니스 담당자 | 지표/분석 맥락과 미확정 질문을 사람이 검토할 수 있는 문서로 확인 |
| downstream agent | 특정 프로퍼티 질문에 대해 전체 SQL을 다시 읽지 않고 작은 지식 카드로 답변 |

## 6. 핵심 사용자 시나리오

### 시나리오 A — 이벤트 정의 레포 기반 프로퍼티 이해

1. 사용자가 특정 이벤트 정의 레포를 지정한다.
2. 에이전트가 해당 레포에서 이벤트와 프로퍼티의 기본 정의를 가져온다.
3. 에이전트가 이벤트 → 프로퍼티 기본 이해 카드를 만든다.
4. 정의만으로 분석 맥락이 부족한 프로퍼티를 표시한다.

### 시나리오 B — 단일 SQL에서 부족한 프로퍼티 사용 맥락 보강

1. 사용자가 BigQuery SQL 텍스트 또는 파일을 제공한다.
2. 에이전트가 기존 이벤트/프로퍼티 정의와 SQL 사용처를 매칭한다.
3. 에이전트가 각 프로퍼티의 SQL 사용 역할과 evidence를 정리한다.
4. 에이전트가 “이 프로퍼티는 어떤 지표/분석 맥락에 쓰이는 것 같다”는 후보 해석을 만든다.
5. 결과를 YAML + Markdown으로 출력한다.

### 시나리오 C — 특정 프로퍼티 질문에 답변

1. 사용자가 `event_name.property_name`을 묻는다.
2. 에이전트는 property card를 조회한다.
3. 관련 지표/분석 후보, 사용 방식, source reference, confidence, 미확정 질문을 제시한다.
4. 공식 지식이 아닌 경우 candidate임을 명확히 표시한다.

## 7. 산출물 원칙

- **YAML = canonical machine-readable source**
- **Markdown = human-readable review document**
- 모든 해석은 evidence와 연결한다.
- SQL에서 직접 관찰 가능한 사실과 LLM/agent가 추론한 해석을 분리한다.
- 사람 승인 전까지 공식 지식으로 등록하지 않는다.
- confidence는 정답 확률이 아니라 “evidence 기반 후보 신뢰도”로 취급한다.

## 8. Review state model

| 상태 | 의미 |
| --- | --- |
| `candidate` | 자동 생성된 후보 지식 |
| `needs_human_review` | 공식화 전에 사람 판단이 필요한 상태 |
| `approved` | 사람이 승인한 공식 지식 |
| `rejected` | 사람이 폐기한 후보 |

MVP 기본값은 `needs_human_review`이다.

## 9. 자동 확정 가능 vs 사람 검토 필요

### 자동 확정 가능

- SQL에 등장한 이벤트명.
- SQL에 등장한 프로퍼티명.
- 프로퍼티가 등장한 clause/role.
- source file/query/snippet reference.
- alias, expression, aggregation 등 syntactic evidence.

### 후보로만 제안

- 어떤 지표/분석에 쓰이는지.
- 주제/topic 또는 목적/purpose.
- 비즈니스 룰 후보.
- 예외 조건 후보.
- 의사결정 맥락 후보.

### 사람 검토 필요

- 공식 지표 정의.
- 프로퍼티의 비즈니스 의미.
- 전사 공통 룰 여부.
- 승인/폐기 결정.

## 10. 성공 기준

MVP는 특정 이벤트 프로퍼티를 물었을 때 다음을 제시할 수 있으면 성공이다.

- 관련 지표/분석 후보.
- 해당 분석이 SQL에서 어떻게 이뤄지는지.
- 프로퍼티가 쓰인 clause/role.
- source reference와 snippet.
- “이런 맥락인 것 같다”는 후보 해석.
- confidence와 review 필요 여부.
- 사람이 확인해야 할 질문.

## 11. 비기능 요구사항

- 내부 지식은 이 공개/초기 레포에 저장하지 않는다.
- 샘플/익명화 SQL만 사용한다.
- 실제 회사 데이터 값이나 민감 정보는 산출물에 남기지 않는다.
- 재실행 가능해야 한다.
- 이벤트 정의 레포의 commit SHA/source hash 기반 캐싱과 future batch merge를 고려한다.
- 결과는 작은 카드 단위로 쪼개 downstream agent가 토큰 효율적으로 검색할 수 있어야 한다.

## 12. 미결정/후속 결정

- 실제 구현 언어와 프레임워크.
- SQL parser 선택 여부.
- LLM provider/model 선택.
- 내부 레포 이동 후 batch indexing 방식.
- 승인 UI 또는 PR 기반 review flow 도입 여부.
