# Failure Cases — Property Context Extraction MVP

## 1. 실패 케이스 원칙

이 에이전트는 SQL에서 후보 지식을 추출한다. 따라서 가장 위험한 실패는 “근거가 약한 해석을 공식 사실처럼 말하는 것”이다.

모든 실패 처리는 다음 원칙을 따른다.

- 불확실하면 `needs_human_review`로 보낸다.
- 근거 없는 해석은 생성하지 않는다.
- SQL에서 관찰한 사실과 비즈니스 해석을 분리한다.
- source reference가 없는 candidate는 무효로 취급한다.

## 2. 주요 실패 케이스

| ID | 케이스 | 예시 | 기대 동작 |
| --- | --- | --- | --- |
| F01 | 이벤트명/프로퍼티명 추출 실패 | nested `event_params` 또는 JSON path | `unknown` 또는 low confidence로 표시하고 snippet 보존 |
| F02 | alias 때문에 의미 오해 | `sp` alias가 실제 property인지 CTE 컬럼인지 불명확 | alias lineage를 추적하거나 human question 생성 |
| F03 | 동적 SQL | 문자열 concat으로 query 생성 | unsupported/dynamic SQL로 표시, 해석 최소화 |
| F04 | CASE WHEN 과잉 해석 | CASE branch를 전사 비즈니스 룰로 오해 | query-local rule candidate로만 표시 |
| F05 | WHERE filter 과잉 일반화 | 특정 분석용 filter를 공통 제외 룰로 오해 | “해당 SQL에서 관찰됨”으로 제한 |
| F06 | metric name hallucination | alias만 보고 공식 지표명 생성 | `metric_candidate`로 표시, official_status=`not_official` |
| F07 | source reference 누락 | snippet 없이 context 생성 | candidate 생성 실패 또는 invalid 처리 |
| F08 | conflicting usage | 같은 property가 서로 다른 분석에 사용 | 여러 context 유지, conflict_notes 생성 |
| F09 | property value 민감 정보 노출 | 내부 캠페인명/고객 세그먼트 값 | 값 마스킹 또는 샘플 데이터만 허용 |
| F10 | unsupported dialect | BigQuery가 아닌 SQL | dialect warning, best-effort extraction |
| F11 | 너무 긴 SQL | token limit 초과 | deterministic facts 먼저 추출, chunk 단위 처리 |
| F12 | downstream 답변 과잉 주장 | candidate를 official처럼 답변 | candidate/needs review disclaimer 강제 |

## 3. 상세 대응

### F01. Nested property extraction 실패

BigQuery 이벤트 로그는 `event_params`, JSON extraction, UNNEST 패턴을 사용할 수 있다.

대응:

- parser가 property path를 확정하지 못하면 `property.path: unknown`으로 둔다.
- expression과 snippet을 보존한다.
- confidence를 `low`로 둔다.
- 질문 생성: “이 expression이 어떤 이벤트 프로퍼티를 의미하는가?”

### F04/F05. SQL-local rule 과잉 일반화

SQL에 있는 filter나 CASE는 해당 쿼리에서만 적용되는 임시 분석 조건일 수 있다.

대응:

- business rule은 기본적으로 `business_rule_candidate`로만 생성한다.
- `scope: query_local_candidate`를 붙인다.
- 전사 공통 여부는 사람 검토 질문으로 남긴다.

### F06. 지표명 환각

SQL alias가 `conversion_rate`라고 해서 공식 지표명이 “전환율”이라고 단정하면 안 된다.

대응:

- `MetricCandidate`로만 생성한다.
- 공식 지표명 필드는 `official_name: null` 또는 생략한다.
- confidence는 alias/comment/evidence 강도에 따라 조정한다.

### F08. 상충 맥락

같은 property가 A 쿼리에서는 세그먼트 기준, B 쿼리에서는 제외 조건으로 쓰일 수 있다.

대응:

- context를 합쳐 하나로 단정하지 않는다.
- context별 evidence를 따로 둔다.
- batch merge 단계에서는 conflict note를 생성한다.

## 4. Invalid output 조건

다음 조건을 만족하면 산출물을 실패로 간주한다.

- candidate interpretation에 source/evidence가 없다.
- `approved` 상태를 자동으로 부여했다.
- 공식 지표 정의처럼 단정했다.
- 내부 민감 데이터 값이 샘플/익명화 없이 노출되었다.
- property question 답변에 YAML/Markdown에 없는 정보를 추가했다.

## 5. Human review 질문 템플릿

- 이 프로퍼티는 어떤 공식 비즈니스 의미를 갖는가?
- 이 SQL에서의 사용 방식이 다른 지표에도 공통 적용되는가?
- 이 filter/CASE는 임시 분석 조건인가, 전사 공통 룰인가?
- 이 metric candidate는 실제 공식 지표와 연결되는가?
- 같은 property에 대해 서로 다른 맥락이 있을 때 어떤 맥락이 우선인가?
