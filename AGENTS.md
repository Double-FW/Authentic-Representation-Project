---
title: "AGENTS"
version: "1.2.0"
author: "Double-FW"
date: "2026-10-13"
license: "unspecified"
language: "en"
repository: "Double-FW/Authentic-Representation-Project"
artifact_type: "policy_document"
use_case: ["research_data_governance","agent_rules","representation_analysis","external_data_reference"]
entities:
  - actor
  - production
  - casting_event
  - disability_category
  - dataset
  - report
  - imdb_2026_dataset
  - external_source
---

# AGENTS.md

This file contains machine-readable agent rules, metadata, templates, observability hooks, a quick-start checklist, a minimal programmatic decision flow, and versioning/changelog metadata for automation to validate before trusting rules.

## Machine-readable rules and metadata (YAML)

rules:
  - id: R001
    title: "Preserve original data"
    priority: high
    trigger: "file.added | file.updated"
    condition: "path matches '**/*.xlsx' or '**/*.docx'"
    action:
      - "store_raw_copy: true"
      - "add_metadata: true"
    rationale: "Raw sources must be preserved for provenance and audit."

  - id: R002
    title: "Framing for disability fields"
    priority: high
    trigger: "dataset.accessed"
    condition: "dataset.schema contains 'disability'"
    action:
      - "label_field: disclosure_status"
      - "enforce_distinction: [apparent, self_disclosed, administrative]"
      - "annotate: definitions and caveats"
    rationale: "Distinguish apparent vs verified status in outputs."

  - id: R003
    title: "No PII leakage in public outputs"
    priority: critical
    trigger: "report.published | artifact.exported"
    condition: "output.visibility == public"
    action:
      - "redact_fields: [name, contact, national_identifiers]"
      - "anonymize_roles: true"
    rationale: "Protect personal privacy."

  - id: R004
    title: "IMDb 2026 external source handling"
    priority: high
    trigger: "external_source.accessed | source_imported"
    condition: "source.name == 'IMDb 2026' or source.source_type == 'imdb'"
    action:
      - "attach_source_provenance: true"
      - "mark_secondary_context: true"
      - "do_not_treat_as_verified_disability_status: true"
      - "record_collection_date: '2026'"
      - "label_as_external_context: true"
    rationale: "IMDb 2026 data can provide contextual metadata for productions and casting visibility but must not be used as direct evidence of actual disability status or personal disclosure without independent verification."

metadata_schema:
  version: "1.0"
  required_fields:
    - title
    - description
    - owner
    - source
    - collection_date
    - reporting_period
    - unit_of_analysis
    - category_scheme
    - disclosure_status
    - sensitivity
    - source_type
    - provenance

# JSON Schema (simplified) for validation

metadata_json_schema: |
  {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "required": ["title","owner","source","collection_date"],
    "properties": {
      "title": {"type":"string"},
      "description": {"type":"string"},
      "owner": {"type":"string"},
      "source": {"type":"string"},
      "collection_date": {"type":"string","format":"date"},
      "reporting_period": {"type":"string"},
      "unit_of_analysis": {"type":"string"},
      "category_scheme": {"type":"string"},
      "disclosure_status": {"type":"string","enum":["self_disclosed","apparent","verified","unknown"]},
      "source_type": {"type":"string","enum":["primary","secondary","external_context","archive","mixed"]},
      "provenance": {"type":"string"}
    }
  }

## IMDb 2026 data handling

source_registry:
  - name: "IMDb 2026"
    source_type: "external_context"
    purpose: "Contextual metadata for productions, cast, credits, and title-level reference in support of broader representation analysis."
    scope: "industry and production-level metadata; not a primary disability dataset"
    allowed_use:
      - "cross-check production metadata"
      - "support title and cast tracking"
      - "compare broad production context"
    prohibited_use:
      - "use as evidence of verified disability status"
      - "use as proof of self-disclosure"
      - "treat as a person-level disability register"
    metadata_requirements:
      - "record_source: 'IMDb 2026'"
      - "mark_secondary_context: true"
      - "note_provenance: 'External public metadata; not independently validated by project researchers'"
      - "record_date_accessed: required"
      - "document_limitations: required"

external_data_policy:
  - rule: "Any IMDb-derived dataset must be stored separately from primary project datasets."
  - rule: "Any title or cast-level match from IMDb must be validated against project source material before publication."
  - rule: "If IMDb data is used to enrich a report, the output must state that the dataset is contextual, not authoritative for disability classification."
  - rule: "When an IMDB field overlaps with actor identity, disability, or personal disclosure, it must be treated as unverified unless a primary evidence record exists."

## Templated prompt/response examples (short)

templates:
  - name: "DatasetSummaryRequest"
    prompt: |
      "You are an assistant summarizing a dataset about casting activity. Provide: dataset title, reporting_period, unit_of_analysis, top-3 findings, and limitations. Use neutral, person-centered language."
    example_response: |
      "Title: 2024 Casting Activity Master\nReporting period: 2021-2024\nUnit: production-role\nTop findings: ...\nLimitations: apparent vs self-disclosed distinctions apply."

  - name: "PolicyCheckBeforePublish"
    prompt: |
      "Before publishing a public report, check for PII, confirm metadata, list redactions, and provide an audit summary. Respond with JSON: {publish: bool, issues:[], audit: {redacted_fields:[], retained_raw_paths:[]}}"
    example_response: |
      "{\n  \"publish\": false,\n  \"issues\": [\"contains actor full names\"],\n  \"audit\": {\"redacted_fields\": [\"name\"], \"retained_raw_paths\": [\"/Actor Data Sets/…\"]}\n}"

  - name: "IMDbContextReference"
    prompt: |
      "Use IMDb 2026 as contextual metadata only. State whether it is being used for title, cast, or production context. Do not claim verified disability status. Output a concise note with provenance and limitations."
    example_response: |
      "Contextual note: IMDb 2026 data used to confirm production title and cast listing context only. It is a secondary external source and does not establish verified disability status or self-disclosure. Provenance: external metadata; date accessed: 2026-10-13; limitations: not a primary disability evidence source."

prompt_templates_notes: "Agents should select the smallest applicable template and always include metadata validation step before returning a response that references datasets or external sources such as IMDb 2026."

## Observability hooks and logging requirements

audit_event_schema:
  - field: event_id
    type: uuid
  - field: timestamp
    type: iso8601
  - field: actor
    type: string
  - field: action
    type: string
  - field: target
    type: path
  - field: outcome
    type: string
  - field: redacted_fields
    type: [string]
  - field: notes
    type: string
  - field: external_source
    type: string

logging_requirements:
  - record: all escalations, publications, metadata changes, and IMDb-derived source imports
  - retention_days: 1825 # 5 years
  - storage: "encrypted-blob-store:projects/Double-FW/Authentic-Representation-Project/audit/"
  - access_controls: ["project_admin","data_steward"]
  - redact_policy:
      - redact_when: "output.visibility == public"
        fields: ["actor.name","actor.contact","national_identifiers"]
      - redact_when: "user_request.redaction == true"
        fields: ["*personal_identifiers*"]
  - escalation_notice:
      - on: "rule.R003 triggered"
        notify: ["data_steward@doublefw.com"]
        attach: ["audit_event_id","affected_paths","suggested_redactions"]
      - on: "rule.R004 triggered"
        notify: ["data_steward@doublefw.com"]
        attach: ["source_name","provenance","validation_status"]

observability_endpoints:
  - metrics: "/internal/metrics/agents"
  - audit_log: "/internal/logs/audit"
  - traces: "/internal/traces/agents"

# Quick-start checklist (minimal)

quick_start:
  - step: "Read README.md and AGENTS.md"
  - step: "Validate dataset metadata against metadata_json_schema"
  - step: "If IMDb 2026 data is used, add provenance and record it as external_context"
  - step: "If producing public outputs, run PolicyCheckBeforePublish template"
  - step: "Preserve raw sources and create a derived copy with version"
  - step: "Record audit_event with redactions and retention policy"

## Minimal programmatic decision flow (JSON)

flow:
  start: check_metadata
  nodes:
    - id: check_metadata
      type: decision
      condition: "metadata.valid == true"
      on_true: allow_analysis
      on_false: request_metadata
    - id: request_metadata
      type: action
      action: "return: 'metadata_missing', fields_needed"
      on_true: end
    - id: allow_analysis
      type: decision
      condition: "output_visibility == public"
      on_true: run_policy_check
      on_false: produce_internal_report
    - id: run_policy_check
      type: action
      action: "invoke: PolicyCheckBeforePublish"
      on_true: review_and_publish
    - id: review_and_publish
      type: decision
      condition: "policy_check.publish == true"
      on_true: publish_and_audit
      on_false: return_issues
    - id: publish_and_audit
      type: action
      action: "store: audit_event; push: public_output"
      on_true: end
    - id: produce_internal_report
      type: action
      action: "store: internal_output; store: audit_event"
      on_true: end

# Decision flow as compact JSON for programmatic consumption

flow_json: |
  {
    "start": "check_metadata",
    "nodes": {
      "check_metadata": {"type":"decision","condition":"metadata.valid","true":"allow_analysis","false":"request_metadata"},
      "request_metadata": {"type":"action","action":"return metadata_missing","next":null},
      "allow_analysis": {"type":"decision","condition":"output_visibility=='public'","true":"run_policy_check","false":"produce_internal_report"},
      "run_policy_check": {"type":"action","action":"invoke PolicyCheckBeforePublish","next":"review_and_publish"},
      "review_and_publish": {"type":"decision","condition":"policy_check.publish==true","true":"publish_and_audit","false":"return_issues"},
      "publish_and_audit": {"type":"action","action":"publish;create_audit_event","next":null},
      "produce_internal_report": {"type":"action","action":"store_internal;create_audit_event","next":null}
    }
  }

## Version, changelog, and review metadata

policy_meta:
  current_version: "1.2.0"
  latest_release_date: "2026-10-13"
  changelog:
    - version: "1.0.0"
      date: "2026-10-08"
      changes: "Initial AGENTS.md with principles and metadata requirements"
    - version: "1.1.0"
      date: "2026-10-13"
      changes: "Added machine-readable rules, templates, observability hooks, flow_json, and policy_meta"
    - version: "1.2.0"
      date: "2026-10-13"
      changes: "Added IMDb 2026 external source provenance rules and contextual metadata handling"
  reviewers:
    - name: "Double FW"
      role: "owner"
      reviewed_at: "2026-10-13"
  next_review_due: "2027-10-13"

review_requirements:
  - must_have: ["policy_meta.current_version","rules","metadata_json_schema","audit_event_schema"]
  - auto_check_script: "/scripts/validate_agents_policy.sh"

## How automation should validate before trusting rules

validation_steps:
  - step: "fetch AGENTS.md and parse YAML front matter"
  - step: "confirm policy_meta.current_version matches referenced version"
  - step: "validate rules against allowed schema and check for unresolved TODOs"
  - step: "ensure audit endpoints and retention are present"
  - step: "verify reviewer signature or reviewer entry present"
  - step: "if IMDb 2026 is used, confirm source_type is external_context and provenance is specified"

## Redaction and escalation (programmatic)

redaction_rules:
  - id: REDACT_PUBLIC_PIIs
    when: "output.visibility == public"
    fields: ["actor.name","actor.contact","national_identifiers","address"]
    method: "masking or removal"

escalation_rules:
  - id: ESCALATE_SENSITIVE_PUBLISH
    trigger: "rule.R003 and output.contains_unredacted_pii"
    action:
      - "create_audit_event"
      - "notify: data_steward@doublefw.com"
      - "hold_publish: true"

  - id: ESCALATE_IMDB_CONTEXT
    trigger: "rule.R004 and source.validation_status != 'verified'"
    action:
      - "create_audit_event"
      - "attach: source_name, provenance, limitations"
      - "require_method_note: true"
      - "notify: data_steward@doublefw.com"

## Maintenance notes

- Keep the YAML front matter up to date (automation reads this first).
- Do not change rule IDs (R001...) without updating changelog and policy_meta.
- For major changes, increment major version and add an explicit review sign-off entry.
- IMDb 2026 data should be treated as contextual metadata only unless validated by a primary project evidence source.

---

Maintained for: Double-FW/Authentic-Representation-Project
