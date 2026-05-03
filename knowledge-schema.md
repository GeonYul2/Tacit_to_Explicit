# Knowledge Schema — Property-Centered Candidate Knowledge

## 1. 설계 원칙

이 스키마는 BigQuery SQL에서 추출한 이벤트 프로퍼티 활용 맥락을 저장하기 위한 canonical YAML 구조다.

핵심 원칙:

1. **프로퍼티 중심**: primary entity는 이벤트 프로퍼티 knowledge card다.
2. **증거 기반**: 모든 후보 해석은 source reference와 연결한다.
3. **사실/추론 분리**: SQL에서 직접 관찰한 사실과 agent 추론을 분리한다.
4. **후보 우선**: 승인 전까지 공식 지식이 아니다.
5. **배치 확장성**: 단일 SQL 결과도 나중에 여러 SQL 파일 결과와 병합 가능해야 한다.

## 2. Top-level package

```yaml
schema_version: "0.1.0"
package_id: "knowledge_package_001"
generated_at: "2026-05-03T00:00:00Z"
input_scope:
  mode: "single_sql"
  source_count: 1
  batch_ready: true
sources: []
event_definitions: []
property_definitions: []
property_cards: []
analysis_contexts: []
metric_candidates: []
business_rule_candidates: []
open_questions: []
```

## 3. SourceReference

SQL 출처와 snippet을 추적한다.

```yaml
id: "src_001"
type: "bigquery_sql"
name: "query_001.sql"
path: "samples/query_001.sql"
source_hash: "sha256:..."
snippet_ref:
  start_line: 12
  end_line: 27
  excerpt: "COUNTIF(event_name = 'purchase') ..."
contains_sensitive_data: false
anonymization_status: "sample_or_anonymized"
```

### 필드

| 필드 | 필수 | 설명 |
| --- | --- | --- |
| `id` | yes | source reference ID |
| `type` | yes | `github_repository`, `event_definition_file`, `bigquery_sql`, future: `confluence_doc`, `manual_note` |
| `path` | no | 파일 경로 |
| `source_hash` | yes | 캐시/변경 감지용 hash |
| `snippet_ref` | yes | 근거 위치 |
| `contains_sensitive_data` | yes | 민감 정보 포함 여부 |
| `anonymization_status` | yes | `sample_or_anonymized`, `internal_only`, `unknown` |

## 4. EventDefinition and PropertyDefinition

이벤트 정의 레포에서 가져온 기본 정보다. SQL 추론보다 우선하는 grounding source로 사용한다.

```yaml
event_definitions:
  - id: "event_def_001"
    source_ref: "src_event_repo_001"
    event_name: "sample_event"
    description: "이벤트 정의 레포에서 가져온 설명"
    owner: "unknown"
    property_refs:
      - "property_def_001"
    confidence: "high"
    review_status: "candidate"

property_definitions:
  - id: "property_def_001"
    source_ref: "src_event_repo_001"
    event_name: "sample_event"
    property_name: "sample_property"
    canonical_key: "sample_event.sample_property"
    data_type: "string"
    description: "이벤트 정의 레포에 기록된 프로퍼티 설명"
    known_values: []
    limitations: []
    confidence: "high"
    review_status: "candidate"
```

역할:

- 이벤트/프로퍼티 기본 이해를 제공한다.
- SQL에서 발견된 property usage와 매칭되는 기준점이다.
- 정의만으로 분석 맥락이 부족하면 `PropertyKnowledgeCard`의 `analysis_context_refs`가 BigQuery SQL evidence로 보강된다.

## 4. PropertyKnowledgeCard

Primary artifact다.

```yaml
id: "property_card_001"
type: "event_property_usage_card"
event:
  name: "sample_event"
  confidence: "high"
property:
  name: "sample_property"
  path: "event_params.sample_property"
  data_type: "unknown"
canonical_key: "sample_event.sample_property"
definition_refs:
  event_definition_ref: "event_def_001"
  property_definition_ref: "property_def_001"
context_completeness: "definition_only_needs_analysis_context"
summary: "이 프로퍼티는 특정 전환/세그먼트 분석에서 필터 또는 그룹핑 기준으로 사용되는 후보 맥락이 있다."
observed_usages:
  - id: "usage_001"
    source_ref: "src_001"
    sql_role: "where_filter"
    expression: "sample_property = 'example_value'"
    clause: "WHERE"
    confidence: "high"
analysis_context_refs:
  - "analysis_context_001"
metric_candidate_refs:
  - "metric_candidate_001"
business_rule_candidate_refs:
  - "rule_candidate_001"
confidence: "medium"
review_status: "needs_human_review"
official_status: "not_official"
questions:
  - "이 프로퍼티 기준이 모든 관련 지표에 공통 적용되는가?"
```

### 필수 필드

| 필드 | 설명 |
| --- | --- |
| `id` | 카드 고유 ID |
| `event.name` | 이벤트 이름 후보 |
| `property.name` | 프로퍼티 이름 후보 |
| `canonical_key` | lookup key. 예: `event.property` |
| `observed_usages` | SQL에서 관찰된 사용처 |
| `confidence` | 후보 지식 신뢰도 |
| `review_status` | human review 상태 |
| `official_status` | 공식 지식 여부 |
| `questions` | 사람 검토 질문 |

## 6. ObservedUsage

SQL에서 직접 관찰 가능한 사용 사실이다.

```yaml
id: "usage_001"
source_ref: "src_001"
sql_role: "case_condition"
clause: "CASE WHEN"
expression: "CASE WHEN sample_property IS NOT NULL THEN 1 END"
nearby_alias: "activated_users"
line_range:
  start: 15
  end: 18
confidence: "high"
extraction_method: "sql_static_analysis"
```

### `sql_role` enum

- `select_projection`
- `where_filter`
- `join_key`
- `group_by_dimension`
- `order_by_dimension`
- `case_condition`
- `aggregation_input`
- `derived_expression`
- `unnest_or_json_extract`
- `unknown`

## 7. AnalysisContext

프로퍼티가 어떤 분석 맥락에서 쓰이는지에 대한 후보다.

```yaml
id: "analysis_context_001"
type: "analysis_context_candidate"
topic: "conversion_analysis"
purpose: "전환 여부를 조건별로 비교하는 분석 후보"
context_summary: "SQL 구조상 sample_property는 전환 세그먼트를 나누거나 필터링하는 데 사용되는 것으로 보인다."
evidence_refs:
  - "usage_001"
inference_basis:
  - "property appears in WHERE filter"
  - "query aggregates users by conversion-related alias"
confidence: "medium"
review_status: "needs_human_review"
```

## 8. MetricCandidate

공식 지표가 아니라 SQL에서 추론한 지표 후보다.

```yaml
id: "metric_candidate_001"
type: "metric_definition_candidate"
name: "sample_conversion_rate"
description: "특정 이벤트 발생 사용자 중 후속 전환 이벤트가 발생한 비율 후보"
calculation_logic:
  numerator: "COUNTIF(conversion_condition)"
  denominator: "COUNT(DISTINCT user_id)"
  filters:
    - "sample_property = 'example_value'"
property_refs:
  - "sample_event.sample_property"
evidence_refs:
  - "usage_001"
confidence: "medium"
review_status: "needs_human_review"
official_status: "not_official"
```

## 9. BusinessRuleCandidate

SQL filter/CASE/JOIN에서 추론한 후보 룰이다.

```yaml
id: "rule_candidate_001"
type: "business_rule_candidate"
name: "sample_property_filter_rule"
description: "sample_property 값이 특정 조건을 만족하는 이벤트만 분석 대상에 포함하는 후보 규칙"
rule_expression: "sample_property = 'example_value'"
applies_to:
  events:
    - "sample_event"
  properties:
    - "sample_property"
evidence_refs:
  - "usage_001"
confidence: "medium"
review_status: "needs_human_review"
questions:
  - "이 필터는 해당 SQL에만 적용되는가, 전사 공통 분석 룰인가?"
```

## 10. Confidence policy

| 값 | 의미 |
| --- | --- |
| `high` | SQL에서 직접 관찰 가능하고 근거 위치가 명확함 |
| `medium` | SQL evidence가 있으나 해석이 포함됨 |
| `low` | 근거가 약하거나 이름/alias 기반 추론 비중이 큼 |
| `unknown` | 판단 불가 |

Confidence는 공식성 또는 정답성을 의미하지 않는다.

## 11. Review status policy

| 값 | 의미 |
| --- | --- |
| `candidate` | 자동 생성 후보 |
| `needs_human_review` | 사람 검토 필요 |
| `approved` | 승인됨 |
| `rejected` | 폐기됨 |

MVP 기본값은 `needs_human_review`다.

## 12. Markdown rendering sections

YAML 카드마다 Markdown은 다음 순서로 렌더링한다.

1. Property summary
2. Related metrics/analyses
3. Observed SQL usages
4. Candidate interpretation
5. Evidence/source references
6. Confidence and review status
7. Human review questions
8. Example query answer
