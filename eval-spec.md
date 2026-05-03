# Evaluation Spec — Property Context Extraction MVP

## 1. 평가 목표

MVP 평가는 “SQL에서 추출한 후보 지식이 특정 이벤트 프로퍼티 질문에 유용하게 답할 수 있는가”를 확인한다.

성공 질문:

> 특정 프로퍼티를 물었을 때 관련 지표/분석이 어떻게 이뤄지고 있는지 제시하고, “이런 맥락인 것 같다”는 후보 해석을 evidence와 함께 보여주는가?

## 2. 평가 대상

- SQL fact extraction
- property knowledge card generation
- analysis/metric context inference
- source reference correctness
- confidence/review status assignment
- downstream property question answering

## 3. 평가 데이터

### MVP fixture set

- 샘플 또는 익명화 BigQuery SQL 5~10개.
- 각 SQL은 1개 이상의 이벤트와 프로퍼티를 포함한다.
- SQL별 기대 라벨을 사람이 작성한다.

### Gold label 예시

```yaml
query_id: "query_001"
expected_properties:
  - canonical_key: "sample_event.sample_property"
    expected_sql_roles:
      - "where_filter"
      - "group_by_dimension"
    expected_metric_or_analysis:
      - "conversion_analysis"
    expected_review_status: "needs_human_review"
```

## 4. 평가 축

### A. SQL fact extraction accuracy

SQL에서 직접 관찰 가능한 사실을 잘 추출하는지 평가한다.

- 이벤트명 추출 정확도.
- 프로퍼티명 추출 정확도.
- SQL role 추출 정확도.
- line/snippet reference 정확도.

권장 기준:

- MVP pass: 주요 property extraction precision 90% 이상.
- role extraction은 복잡도에 따라 70~80% 이상부터 허용.

### B. Evidence grounding

후보 해석이 근거와 연결되어 있는지 평가한다.

- 모든 analysis context에 evidence_refs가 있는가?
- evidence snippet이 실제 주장을 뒷받침하는가?
- source hash/path/snippet이 남아 있는가?

MVP pass:

- candidate interpretation의 100%가 최소 1개 evidence_ref를 가진다.
- evidence가 없는 해석은 생성하지 않거나 `low` confidence + question으로 처리한다.

### C. Candidate usefulness

사람이 보기에 “쓸모 있는 맥락 후보”인지 평가한다.

평가 질문:

1. 이 카드만 보고 프로퍼티가 어떤 분석에 쓰였는지 감이 오는가?
2. 관련 metric/analysis 후보가 너무 추상적이지 않은가?
3. 사람이 검토해야 할 질문이 명확한가?
4. downstream agent가 전체 SQL 없이 답변할 수 있는가?

MVP pass:

- 사람이 샘플 property 질문의 70% 이상에서 “유용함”으로 평가.

### D. Safety / overclaim prevention

가장 중요한 실패 방지는 “후보를 공식 truth처럼 말하지 않는 것”이다.

MVP pass:

- SQL 추론 결과는 항상 candidate/review-needed로 표시한다.
- 공식 지표 정의처럼 단정하지 않는다.
- 전사 공통 룰 여부를 자동 확정하지 않는다.

## 5. End-to-end property QA eval

각 property card에 대해 질문을 만든다.

예시 질문:

```text
sample_event.sample_property는 어떤 분석에 쓰이고 있나요?
```

기대 답변 구조:

1. 관련 metric/analysis 후보.
2. SQL에서의 사용 방식.
3. source reference.
4. candidate interpretation.
5. confidence.
6. human review questions.

Pass 조건:

- 답변이 generated YAML/Markdown에 있는 정보만 사용한다.
- 근거 없는 추가 주장을 하지 않는다.
- 불확실성을 표시한다.

## 6. Precision vs recall 우선순위

MVP는 recall보다 precision과 overclaim 방지를 우선한다.

- 놓친 property는 개선 가능하다.
- 잘못 확정한 비즈니스 의미는 위험하다.
- 따라서 ambiguous한 경우 `needs_human_review`와 open question을 생성한다.

## 7. Regression checks

향후 구현 시 최소 regression fixture를 둔다.

- simple SELECT/WHERE property
- CASE WHEN rule candidate
- GROUP BY dimension
- JOIN key
- nested JSON/event_params extraction
- ambiguous alias
- unsupported dynamic SQL

## 8. Human review rubric

각 카드에 대해 사람이 1~5점으로 평가한다.

| 점수 | 의미 |
| --- | --- |
| 5 | 바로 리뷰/승인 검토에 쓸 수 있음 |
| 4 | 일부 수정하면 유용함 |
| 3 | 근거는 있으나 맥락이 약함 |
| 2 | 추출은 되었지만 해석이 부정확함 |
| 1 | 거의 쓸 수 없음 |

MVP target: 평균 3.5 이상.
