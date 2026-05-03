# Tool Contracts — Tacit-to-Explicit Knowledge Agent

## 1. 목적

이 문서는 구현 시 필요한 내부 tool/function 계약을 정의한다. MVP는 실제 BigQuery 연결 없이, 사용자가 제공한 SQL 파일/텍스트를 처리한다.

## 2. Pipeline overview

```text
event repository input
  -> load_event_repository_context
  -> build_event_property_catalog
SQL input
  -> normalize_sql_input
  -> extract_sql_facts
  -> match_sql_to_event_properties
  -> infer_property_context
  -> build_knowledge_package
  -> render_yaml
  -> render_markdown
  -> answer_property_question
```

## 3. load_event_repository_context

### 역할

사용자가 지정한 특정 이벤트 정의 레포에서 이벤트와 프로퍼티의 기본 정보를 가져온다. 이 단계가 프로젝트의 시작점이다.

### Input

```json
{
  "provider": "github",
  "owner": "ORG",
  "repo": "event-definitions",
  "ref": "main",
  "paths": ["events/"],
  "access_mode": "api_or_local_clone"
}
```

### Output

```json
{
  "repository_source": {
    "id": "src_event_repo_001",
    "type": "github_repository",
    "owner": "ORG",
    "repo": "event-definitions",
    "ref": "main",
    "commit_sha": "...",
    "source_hash": "sha256:..."
  },
  "event_definition_files": []
}
```

### Contract rules

- Scope is one explicitly configured repository, not arbitrary GitHub search.
- Prefer GitHub repository contents/tree APIs or a local clone for source retrieval.
- Copilot/GitHub Models-style inference may summarize or normalize definitions, but repository files remain the source of truth.
- Respect repository permissions and content exclusions.
- Do not store private event definitions in this public/sample repo.

## 4. build_event_property_catalog

### 역할

이벤트 정의 파일을 `EventDefinition`과 `PropertyDefinition`으로 정규화한다.

### Output

```json
{
  "event_definitions": [],
  "property_definitions": [],
  "unresolved_definitions": []
}
```

### Contract rules

- This catalog is the first lookup layer for event/property understanding.
- Missing descriptions or ambiguous properties are marked as incomplete, not hallucinated.
- `canonical_key` must be stable: `event_name.property_name`.

## 5. match_sql_to_event_properties

### 역할

SQL에서 추출한 event/property usage를 이벤트 정의 레포의 catalog와 매칭한다.

### Output

```json
{
  "matches": [
    {
      "canonical_key": "sample_event.sample_property",
      "property_definition_ref": "property_def_001",
      "usage_ids": ["usage_001"],
      "match_confidence": "high"
    }
  ],
  "unmatched_usages": []
}
```

### Contract rules

- Exact event/property matches are high confidence.
- Fuzzy matches require human review.
- Unmatched SQL properties remain candidates and must not create official definitions.

## 6. normalize_sql_input

### 역할

SQL 파일 또는 텍스트 입력을 표준 source object로 변환한다.

### Input

```json
{
  "input_type": "sql_text",
  "name": "query_001.sql",
  "path": "samples/query_001.sql",
  "content": "SELECT ...",
  "anonymization_status": "sample_or_anonymized"
}
```

### Output

```json
{
  "source": {
    "id": "src_001",
    "type": "bigquery_sql",
    "name": "query_001.sql",
    "path": "samples/query_001.sql",
    "source_hash": "sha256:...",
    "content": "SELECT ...",
    "line_count": 120,
    "contains_sensitive_data": false,
    "anonymization_status": "sample_or_anonymized"
  }
}
```

### Errors

- `empty_sql_input`
- `unsupported_input_type`
- `sensitive_data_detected` if future detector is enabled

## 7. extract_sql_facts

### 역할

SQL에서 직접 관찰 가능한 사실을 추출한다. 이 단계는 가능한 한 deterministic해야 한다.

### Input

```json
{
  "source": {
    "id": "src_001",
    "type": "bigquery_sql",
    "content": "SELECT ..."
  },
  "dialect": "bigquery"
}
```

### Output

```json
{
  "sql_facts": {
    "events": [
      {
        "name": "sample_event",
        "evidence_refs": ["snippet_001"],
        "confidence": "high"
      }
    ],
    "properties": [
      {
        "event_name": "sample_event",
        "property_name": "sample_property",
        "property_path": "event_params.sample_property",
        "usages": [
          {
            "id": "usage_001",
            "sql_role": "where_filter",
            "clause": "WHERE",
            "expression": "sample_property = 'example_value'",
            "line_range": {"start": 10, "end": 12},
            "snippet_ref": "snippet_001",
            "confidence": "high"
          }
        ]
      }
    ],
    "query_signals": {
      "aliases": ["conversion_rate"],
      "aggregations": ["COUNT", "COUNTIF"],
      "joins": [],
      "comments": []
    }
  }
}
```

### Contract rules

- Do not infer business meaning here.
- Do not mark official status.
- Use `unknown` when role cannot be determined.
- Preserve line/snippet reference whenever possible.

## 8. infer_property_context

### 역할

SQL facts와 query signals를 바탕으로 metric/analysis/business rule 후보를 생성한다.

### Input

```json
{
  "source": {"id": "src_001", "name": "query_001.sql"},
  "event_property_catalog": {},
  "sql_facts": {},
  "matched_properties": [],
  "optional_manual_context": [
    {
      "type": "manual_note",
      "content": "이 쿼리는 전환 퍼널 분석에 사용된다."
    }
  ]
}
```

### Output

```json
{
  "property_context_candidates": [
    {
      "canonical_key": "sample_event.sample_property",
      "analysis_contexts": [
        {
          "topic": "conversion_analysis",
          "purpose": "전환율 또는 퍼널 분석 후보",
          "context_summary": "이 프로퍼티는 전환 분석에서 필터 또는 세그먼트 기준으로 쓰이는 것으로 보인다.",
          "evidence_usage_ids": ["usage_001"],
          "confidence": "medium",
          "review_status": "needs_human_review",
          "questions": [
            "이 프로퍼티 기준이 해당 쿼리에만 적용되는가?"
          ]
        }
      ],
      "metric_candidates": [],
      "business_rule_candidates": []
    }
  ]
}
```

### Contract rules

- Every inference must cite at least one usage/evidence ID.
- Use candidate language: “appears to”, “후보”, “가능성”.
- If evidence is weak, confidence must be `low` and review question required.
- Never set `approved`.

## 9. build_knowledge_package

### 역할

source, facts, inferred context를 canonical YAML schema로 조립한다.

### Input

```json
{
  "source": {},
  "sql_facts": {},
  "property_context_candidates": []
}
```

### Output

```json
{
  "schema_version": "0.1.0",
  "package_id": "knowledge_package_001",
  "sources": [],
  "property_cards": [],
  "analysis_contexts": [],
  "metric_candidates": [],
  "business_rule_candidates": [],
  "open_questions": []
}
```

### Contract rules

- Generate stable IDs from source hash + canonical key when possible.
- Deduplicate repeated property usages.
- Keep source facts and inference candidates linked.

## 10. render_yaml

### 역할

Knowledge package를 YAML 파일로 출력한다.

### Input

```json
{
  "knowledge_package": {},
  "output_path": "outputs/query_001.knowledge.yaml"
}
```

### Output

```json
{
  "path": "outputs/query_001.knowledge.yaml",
  "bytes_written": 12345
}
```

## 11. render_markdown

### 역할

사람 검토용 문서를 생성한다.

### Input

```json
{
  "knowledge_package": {},
  "output_path": "outputs/query_001.review.md"
}
```

### Markdown sections

- Summary
- Property cards
- Related metrics/analyses
- SQL evidence
- Candidate interpretations
- Confidence/review status
- Human review questions

## 12. merge_property_cards — future batch extension

### 역할

여러 SQL 파일에서 생성된 property cards를 canonical key 기준으로 병합한다.

### MVP status

설계만 한다. 첫 구현 필수 아님.

### Merge rules

- Same `canonical_key` → same property card family.
- Preserve all source references.
- Merge analysis contexts by topic/purpose similarity, but do not collapse conflicting interpretations without marking conflict.
- If evidence conflicts, create `conflict_notes` and require human review.

## 13. answer_property_question

### 역할

생성된 YAML/Markdown만 사용해 특정 프로퍼티 질문에 답한다.

### Input

```json
{
  "canonical_key": "sample_event.sample_property",
  "knowledge_package_paths": ["outputs/query_001.knowledge.yaml"],
  "question": "이 프로퍼티는 어떤 분석에 쓰이나요?"
}
```

### Output

```json
{
  "answer": "이 프로퍼티는 전환 분석에서 필터 기준으로 쓰이는 후보 맥락이 있습니다...",
  "related_metrics_or_analyses": [],
  "evidence_refs": [],
  "confidence": "medium",
  "review_status": "needs_human_review",
  "limitations": ["SQL 기반 후보이며 공식 정의는 아닙니다."]
}
```

### Contract rules

- Do not use information outside provided knowledge package.
- Always disclose candidate status.
- Include source references.
