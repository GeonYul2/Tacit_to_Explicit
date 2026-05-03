# Event Property Context Agent

`event-property-context-agent`는 더 큰 **암묵지 → 형식지(Tacit to Explicit Knowledge)** 프로젝트의 첫 번째 유스케이스입니다.

이 프로젝트의 목표는 플랫폼 이벤트 프로퍼티 주변에 흩어져 있는 **분석 활용 맥락**을 구조화된 지식으로 바꾸는 것입니다.

## 문제

이벤트 레포나 Copilot API를 활용하면 특정 이벤트와 프로퍼티가 무엇인지, 어떤 설명을 갖는지는 어느 정도 확인할 수 있습니다.

하지만 실제로 더 궁금한 것은 보통 다음입니다.

- 이 프로퍼티가 어떤 지표 계산에 쓰이는가?
- 어떤 분석 주제나 의사결정 맥락에서 활용되는가?
- SQL에서 필터, 세그먼트, 조인 키, CASE 조건, 집계 입력 중 어떤 역할을 하는가?
- 이 사용 방식이 일회성 분석 조건인지, 반복적으로 쓰이는 비즈니스 룰 후보인지?
- 어떤 내용은 확실하고, 어떤 내용은 사람 검토가 필요한가?

즉, 이벤트/프로퍼티의 “정의”는 이미 형식지에 가깝지만, **분석에서 왜/어떻게 쓰이는지에 대한 맥락은 암묵지로 남아 있는 경우가 많습니다.**

## 프로젝트 목표

이 프로젝트는 다음 흐름을 자동화/구조화합니다.

```text
특정 이벤트 정의 레포 연결
→ 이벤트 이해
→ 이벤트 프로퍼티 이해
→ 정의만으로 부족한 분석 맥락 식별
→ BigQuery SQL / 분석 코드에서 사용처 확인
→ 프로퍼티 중심 context card 생성
→ 사람 검토
→ 승인된 지식만 공식 지식으로 취급
```

반복적으로 실행하면, 기존에는 사람이나 SQL 속에 숨어 있던 분석 맥락이 문서와 지식 카드로 축적됩니다.

## MVP

MVP는 다음 입력을 전제로 합니다.

1. 특정 이벤트 정의 레포 접근
2. BigQuery SQL 파일 또는 SQL 텍스트 1개

MVP 산출물은 다음입니다.

- 이벤트 프로퍼티 중심 YAML knowledge card
- 사람 검토용 Markdown 문서
- SQL source/evidence reference
- confidence
- human review status
- 사람이 확인해야 할 질문

SQL에서 추론한 분석 맥락은 항상 **후보 지식(candidate knowledge)** 으로 취급합니다. 사람 승인 전까지 공식 지식으로 등록하지 않습니다.

## 예시 산출물 개념

```yaml
canonical_key: checkout_completed.payment_method
known_definition:
  source: event_repo_or_copilot
  description: 결제 수단을 나타내는 프로퍼티
analysis_contexts:
  - topic: conversion_analysis
    purpose: 결제 수단별 전환율 또는 성과 비교 분석 후보
    sql_usage:
      role: group_by_dimension
      evidence: query_001.sql:L20-L35
    inferred_context: 결제 수단별 성과를 비교하는 분석에 쓰이는 것으로 보임
    confidence: medium
    review_status: needs_human_review
questions:
  - 이 프로퍼티는 모든 결제 전환 지표에서 공통 dimension으로 쓰이는가?
```

## 이 프로젝트가 하는 것

- 이벤트 정의 레포에서 이벤트/프로퍼티 기본 정보를 가져오는 구조를 설계합니다.
- BigQuery SQL에서 이벤트 프로퍼티 사용처를 찾습니다.
- 프로퍼티가 어떤 지표/분석 맥락에서 쓰이는지 후보 해석을 생성합니다.
- SQL evidence와 confidence를 함께 저장합니다.
- 사람이 검토해야 할 지점을 명확히 남깁니다.
- downstream agent가 특정 프로퍼티 질문에 토큰 효율적으로 답할 수 있도록 작은 카드 단위로 지식을 쪼갭니다.

## 이 프로젝트가 하지 않는 것

- 이벤트 레포 문서를 새로 작성하는 프로젝트가 아닙니다.
- Copilot API를 대체하는 프로젝트가 아닙니다.
- SQL에서 공식 지표 정의를 자동 확정하지 않습니다.
- 승인 UI를 MVP에 포함하지 않습니다.
- 회사 내부 이벤트 정의, SQL, 대시보드, 고객 데이터는 이 레포에 저장하지 않습니다.

## 현재 레포 구성

```text
README.md                 # 프로젝트 소개
agent-prd.md              # 제품 요구사항
knowledge-schema.md        # YAML 지식 스키마
eval-spec.md              # 평가 기준
tool-contracts.md         # 내부 tool/function 계약
failure-cases.md          # 실패 케이스와 대응 정책
cost-and-caching.md       # 토큰 비용과 캐싱 전략
implementation-plan.md    # 단계별 구현 계획
```

## 현재 상태

현재는 **설계/스펙 단계**입니다.

아직 구현하지 않은 것:

- 이벤트 레포 connector
- Copilot/API 연동
- BigQuery SQL parser/extractor
- YAML/Markdown generator
- 평가 harness

## 보안/데이터 원칙

이 레포에는 실제 회사 내부 지식이나 민감 데이터를 넣지 않습니다.

- 샘플/익명화 SQL만 사용합니다.
- 실제 이벤트 정의와 분석 코드는 내부 레포에서만 다룹니다.
- SQL에서 추론한 맥락은 사람 검토 전까지 공식 지식이 아닙니다.
