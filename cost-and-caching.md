# Cost and Caching — Token-Efficient Property Context Knowledge

## 1. 목표

장기 목표는 BigQuery 코드를 전수 조사해 이벤트 프로퍼티 활용 맥락을 주제/목적별로 쪼개고, 나중에 특정 프로퍼티 질문이 들어왔을 때 전체 SQL을 다시 읽지 않고 작은 knowledge card만 검색해 답변하는 것이다.

즉, 비용 최적화의 핵심은 다음이다.

- 전체 SQL 재분석 최소화.
- event/property definition catalog 캐싱.
- property-centered card 단위 캐싱.
- source hash 기반 변경 감지.
- downstream agent가 필요한 카드만 읽도록 설계.

## 2. 비용이 발생하는 지점

| 단계 | 비용 유형 | 비용 리스크 |
| --- | --- | --- |
| SQL parsing/fact extraction | CPU/local | 낮음 |
| LLM context inference | token/API | 높음 |
| Markdown rendering | token/local | 낮음~중간 |
| batch merge | CPU/token | 중간 |
| property QA | token | 낮음, 카드 검색이 잘 되면 매우 낮음 |

## 3. 캐시 레이어

### 3.1 Event definition repository cache

키: `owner/repo/ref/path + commit_sha`

저장 내용:

- repository metadata
- event definition file manifest
- event/property canonical keys
- definition source hashes

무효화 조건:

- commit SHA 변경.
- configured path 변경.
- parser/schema version 변경.

### 3.2 Source cache

키: `source_hash`

저장 내용:

- normalized SQL content hash
- file path/name
- line map
- detected dialect
- anonymization status

무효화 조건:

- SQL 내용 변경.
- anonymization status 변경.
- parser version 변경.

### 3.3 SQL facts cache

키: `source_hash + extractor_version`

저장 내용:

- detected events
- detected properties
- observed usages
- query signals
- snippet refs

특징:

- deterministic extraction 결과.
- LLM 호출 없이 재사용 가능.

### 3.4 Inference cache

키: `source_hash + inference_prompt_version + model_id + sql_facts_hash`

저장 내용:

- analysis context candidates
- metric candidates
- business rule candidates
- confidence/review questions

무효화 조건:

- source/facts 변경.
- prompt 변경.
- model 변경.
- schema 변경.

### 3.5 Property card cache

키: `canonical_key + source_hash + schema_version`

저장 내용:

- property card shard
- evidence refs
- related contexts
- generated markdown section

장점:

- 특정 property 질문 시 필요한 카드만 로드.
- batch merge 시 property 단위 증분 업데이트 가능.

## 4. Knowledge sharding 전략

### Primary shard

```text
knowledge/
  properties/
    sample_event.sample_property.yaml
```

각 property card는 특정 `canonical_key` 기준으로 작은 파일이 된다.

### Secondary index

```text
knowledge/
  indexes/
    by_topic/conversion_analysis.yaml
    by_metric/sample_conversion_rate.yaml
    by_source/query_001.yaml
```

인덱스는 조회 최적화를 위한 파생물이다. canonical source는 property card다.

## 5. Token-efficient retrieval

특정 property 질문이 들어오면 다음만 로드한다.

1. 해당 property card.
2. 카드에 연결된 analysis context/metric candidate.
3. 필요한 source snippets.
4. conflict/open question summary.

전체 SQL 파일은 기본적으로 다시 읽지 않는다.

## 6. Batch 확장 계획

MVP는 single SQL이지만, schema는 batch-ready로 둔다.

Batch 처리 시:

1. 이벤트 정의 레포 commit SHA와 definition manifest 확인.
2. 모든 SQL 파일 hash 계산.
3. 변경된 파일만 fact extraction.
4. 변경된 파일만 context inference.
5. property card 단위로 merge.
6. affected index만 재생성.

## 7. 비용 절감 정책

- SQL facts는 가능하면 parser/static analysis로 처리한다.
- LLM은 해석이 필요한 context inference에만 사용한다.
- 긴 SQL은 CTE/query block 단위로 chunk한다.
- 각 chunk inference 결과를 source evidence와 연결한다.
- confidence가 낮은 후보에 과도한 reasoning 비용을 쓰지 않고 human review question으로 넘긴다.

## 8. 민감 정보와 저장 정책

초기 레포에는 회사 내부 지식을 넣지 않는다.

- 샘플/익명화 SQL만 저장.
- 실제 내부 레포 이동 전까지 generated knowledge도 샘플만 저장.
- 내부 데이터 값은 필요 시 마스킹한다.
- source snippets는 최소 범위만 보존한다.

## 9. 추천 디렉터리 구조

```text
samples/
  sql/
outputs/
  packages/
  properties/
  reviews/
cache/
  sources/
  facts/
  inference/
```

MVP 구현 시 `cache/`는 git ignore 대상이 될 수 있다.

## 10. 캐시 메타데이터 예시

```yaml
cache_key: "sha256:source...:extractor_v1"
source_hash: "sha256:source..."
schema_version: "0.1.0"
extractor_version: "0.1.0"
inference_prompt_version: "0.1.0"
model_id: "unknown"
created_at: "2026-05-03T00:00:00Z"
invalidated: false
```
