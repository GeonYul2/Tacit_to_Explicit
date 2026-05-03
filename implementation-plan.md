# Implementation Plan — Tacit-to-Explicit Knowledge Agent

## 0. 현재 상태

이 문서는 구현 전 계획이다. 아직 구현을 시작하지 않는다. 구현은 `agent-prd.md`, `knowledge-schema.md`, `eval-spec.md`, `tool-contracts.md`, `failure-cases.md`, `cost-and-caching.md` 검토 후 시작한다.

## 1. 구현 원칙

- 첫 구현은 특정 이벤트 정의 레포 1개와 단일 BigQuery SQL 입력을 지원한다.
- 내부 회사 지식은 이 레포에 넣지 않는다.
- 샘플/익명화 SQL fixture만 사용한다.
- YAML schema validation을 먼저 만든다.
- SQL에서 관찰한 사실과 agent 추론을 분리한다.
- 자동으로 `approved`를 부여하지 않는다.
- 테스트/평가는 property question answering 성공 여부까지 포함한다.

## 2. Phase 0 — Event repository connector

목표: 특정 이벤트 정의 레포를 프로젝트의 첫 번째 grounding source로 연결한다.

예상 작업:

- 이벤트 정의 레포 접근 방식 결정: GitHub API, local clone, 또는 내부 mirror.
- owner/repo/ref/path allowlist 설정.
- 이벤트 정의 파일 조회.
- 이벤트/프로퍼티 기본 catalog 생성.
- private definition이 샘플 레포에 저장되지 않도록 output policy 적용.

완료 조건:

- 지정된 레포에서 이벤트/프로퍼티 정의를 읽을 수 있다.
- `event_name.property_name` canonical key를 만들 수 있다.
- 정의가 부족한 프로퍼티를 `needs_analysis_context`로 표시한다.

## 3. Phase 1 — Repository scaffold

목표: 구현 기반 구조 생성.

예상 작업:

- `samples/sql/` 생성.
- `outputs/` 생성.
- schema/type 정의 위치 결정.
- CLI 또는 script entrypoint 결정.
- generated/cache 파일 git ignore 정책 설정.

완료 조건:

- 샘플 SQL 2~3개가 있다.
- output path convention이 있다.
- schema validation 실행 경로가 있다.

## 4. Phase 2 — Knowledge schema validation

목표: YAML 산출물이 schema를 만족하는지 검증한다.

예상 작업:

- `KnowledgePackage`, `SourceReference`, `PropertyKnowledgeCard`, `ObservedUsage`, `AnalysisContext`, `MetricCandidate`, `BusinessRuleCandidate` type/schema 구현.
- enum validation:
  - `sql_role`
  - `confidence`
  - `review_status`
  - `official_status`
- sample YAML fixture 작성.

완료 조건:

- 올바른 sample YAML은 통과한다.
- evidence 없는 candidate는 실패한다.
- 자동 `approved` 상태는 MVP generator에서 생성되지 않는다.

## 5. Phase 3 — SQL fact extraction

목표: BigQuery SQL에서 직접 관찰 가능한 사실을 추출한다.

예상 작업:

- SQL input normalization.
- source hash 생성.
- line/snippet reference 생성.
- property/event 후보 추출.
- clause/role 추출.
- query signals 추출:
  - aliases
  - aggregations
  - comments
  - CASE expressions
  - filters

구현 옵션:

- 단순 MVP: regex + heuristic + line scanner.
- 견고한 버전: SQL parser 도입.

완료 조건:

- simple SELECT/WHERE/GROUP BY/CASE fixture에서 property usage를 추출한다.
- 추출 결과에 line/snippet reference가 있다.

## 6. Phase 4 — Context inference

목표: SQL facts를 바탕으로 property별 analysis/metric/business rule 후보를 생성한다.

예상 작업:

- inference prompt/template 작성.
- 입력은 SQL 전체가 아니라 extracted facts + minimal snippets 위주로 구성.
- 후보 해석에는 evidence ID를 강제한다.
- confidence와 review questions 생성.

완료 조건:

- 모든 candidate context가 evidence_refs를 가진다.
- 공식 지식처럼 단정하지 않는다.
- ambiguity가 큰 경우 질문을 생성한다.

## 7. Phase 5 — YAML/Markdown rendering

목표: canonical YAML과 human review Markdown을 출력한다.

예상 작업:

- YAML renderer.
- Markdown renderer.
- 파일명 convention:
  - `outputs/packages/{source_id}.knowledge.yaml`
  - `outputs/reviews/{source_id}.review.md`
- Markdown sections는 schema 문서 기준으로 구성.

완료 조건:

- 샘플 SQL에서 YAML + Markdown이 생성된다.
- Markdown만 읽어도 사람이 리뷰할 수 있다.

## 8. Phase 6 — Property question answering

목표: 특정 property 질문에 대해 generated knowledge만 사용해 답한다.

예상 작업:

- property canonical key lookup.
- related metrics/analyses 정리.
- usage/evidence 요약.
- candidate disclaimer 강제.

완료 조건:

- “이 프로퍼티는 어떤 분석에 쓰이나요?” 질문에 대해 property card 기반 답변을 생성한다.
- 답변에 evidence, confidence, review status가 포함된다.

## 9. Phase 7 — Evaluation harness

목표: MVP 성공 기준을 자동/반자동으로 확인한다.

예상 작업:

- fixture SQL + gold labels 작성.
- extraction precision 체크.
- evidence grounding 체크.
- invalid output 체크.
- human review rubric 문서화.

완료 조건:

- 최소 fixture suite가 통과한다.
- invalid candidate rules를 잡아낸다.

## 10. Phase 8 — Future batch design

MVP 이후 확장.

예상 작업:

- directory/repo batch scanner.
- source_hash 기반 incremental processing.
- property card merge.
- by_property/by_topic/by_metric index.
- conflict detection.

완료 조건:

- 여러 SQL 파일에서 같은 property를 병합할 수 있다.
- 변경된 SQL만 재처리한다.
- downstream agent가 property shard만 읽고 답변할 수 있다.

## 11. 권장 구현 순서

1. Event repository connector + event/property catalog.
2. Schema/types + validation.
3. Sample SQL fixtures.
4. Deterministic SQL fact extraction.
5. SQL usage ↔ event property catalog matching.
6. YAML package builder.
7. Markdown renderer.
8. Context inference.
9. Property question answering.
10. Evaluation harness.
11. Batch merge extension.

## 12. 구현 전 체크리스트

- [ ] 산출물 7개 검토 완료.
- [ ] 실제 내부 지식이 레포에 들어가지 않는지 확인.
- [ ] 특정 이벤트 정의 레포 접근 방식 결정.
- [ ] 샘플/익명화 SQL 준비.
- [ ] 구현 언어/런타임 결정.
- [ ] SQL parser 도입 여부 결정.
- [ ] LLM 호출 방식 결정.
- [ ] 테스트 fixture 기준 합의.
