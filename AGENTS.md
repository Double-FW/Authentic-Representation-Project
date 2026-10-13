---
title: "AGENTS"
version: "1.1.0"
author: "Double-FW"
date: "2026-10-13"
license: "unspecified"
language: "en"
repository: "Double-FW/Authentic-Representation-Project"
artifact_type: "policy_document"
use_case: ["research_data_governance","agent_rules","representation_analysis"]
entities:
  - actor
  - production
  - casting_event
  - disability_category
  - dataset
  - report
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
    rationale: "Protect personal privacy.",

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
      "disclosure_status": {"type":"string","enum":["self_disclosed","apparent","verified","unknown"]}
    }
  }

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

prompt_templates_notes: "Agents should select the smallest applicable template and always include metadata validation step before returning a response that references datasets."

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

logging_requirements:
  - record: all escalations, publications, and metadata changes
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

observability_endpoints:
  - metrics: "/internal/metrics/agents"
  - audit_log: "/internal/logs/audit"
  - traces: "/internal/traces/agents"

# Quick-start checklist (minimal)

quick_start:
  - step: "Read README.md and AGENTS.md"
  - step: "Validate dataset metadata against metadata_json_schema"
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
  current_version: "1.1.0"
  latest_release_date: "2026-10-13"
  changelog:
    - version: "1.0.0"
      date: "2026-10-08"
      changes: "Initial AGENTS.md with principles and metadata requirements"
    - version: "1.1.0"
      date: "2026-10-13"
      changes: "Added machine-readable rules, templates, observability hooks, flow_json, and policy_meta"
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

## Maintenance notes

- Keep the YAML front matter up to date (automation reads this first).
- Do not change rule IDs (R001...) without updating changelog and policy_meta.
- For major changes, increment major version and add an explicit review sign-off entry.

---

Maintained for: Double-FW/Authentic-Representation-Project
