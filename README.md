# Intrinsic Code Standards Corpus — Schema Spec v0.1.4

**Status:** v0.1.4 patch integrated. Locked candidate. For founder ratification.
**Author:** Claude Opus 4.7
**Founder:** Myriah Knittle
**Repo:** `intrinsic-code-standards` (public, submoduled into `intrinsic-code` main repo)
**Supersedes:** v0.1.3 rev4 (v0.1.4 patch — Constitution trace ID validation — integrated into this doc)

---

## Changelog v0.1.3 → v0.1.4

Additive patch on the v0.1.3 rev4 base. No existing field shape modified.

### Additions in v0.1.4

1. **LOAD-15 Constitution trace ID validation** (§9). Catches `intrinsic_mapping.constitutional_trace_ids[]` entries that don't resolve to a real `trace_id` in the authoritative `mappings/constitution_to_external_standards.json` mapping file. Mirrors LOAD-4 (auditor consumer validation) — same failure mode, same fix.
2. **`constitution_mapping_file_locator` block** added to `corpus_version.json` (§7). Tells the corpus loader where to find the mapping file relative to the main factory repo; documents the external-auditor access path when the main repo isn't accessible.
3. **Soft-fail mode for LOAD-15** when the mapping file isn't reachable (external-auditor context without main repo access). Honest disclosure rather than silent degradation; loader emits `mapping_file_unavailable_for_load_15_check` warning.

Precedent: MASVS-STORAGE-1 transcription drift (May 19, 2026) — chat-side Claude drafted `SEC-PILLAR-5-SECURE-STORAGE` from intuition; real trace is `SEC-PILLAR-3-SECURE-STORAGE`. GPT cross-check caught it. LOAD-15 is the structural backstop.

---

## Changelog v0.1.2 → v0.1.3 rev4

Single canonical document. Integrates the v0.1.3 patch into the locked rev3 base. No structural redesign.

### Additions in v0.1.3

1. **`standard_claim_policy` block added to per-standard `manifest.json`** (§4). Standard-wide claim posture lives at manifest level; requirements inherit and may add (not un-forbid).
2. **Three-tier inheritance for claim enforcement** (§5): corpus baseline → standard baseline → requirement-specific. Aggregator computes effective claim policy as union of all three.
3. **`certification_disclaimer_present` boolean** in `standard_claim_policy` enables deterministic LOAD-14 enforcement without implicit conditions.
4. **LOAD-14 standard claim policy consistency check** (§9). Catches missing disclaimer text when disclaimer is declared present; catches endorsement claims for standards whose body doesn't certify.

All other content from rev3 unchanged. No existing field shape modified.

---

## 1. Purpose

This corpus is the source of truth the Intrinsic Code factory audits against. Every external standard the factory checks compliance with — OWASP MASVS, NIST 800-63, PCI-DSS, HIPAA, COPPA, GDPR, and others added over time — is transcribed into machine-readable form here.

The corpus exists so that:

1. **Compliance claims are verifiable.** Every signed Validation Receipt cites specific standard requirements from this corpus. External auditors and certification bodies can reproduce audit conditions by reading the same corpus version the factory used.
2. **The factory is model-agnostic.** No LLM has to recall standards from training. Every audit reads requirement text directly from the corpus.
3. **Compliance is proactive, not reactive.** Curated and signed off before audits run.
4. **Claim boundaries are enforced as data, deterministically.** Python enforces; models cannot bypass via paraphrase. Three-tier inheritance (corpus → standard → requirement) keeps standard-wide posture out of per-requirement boilerplate.
5. **Trust claims cannot be self-anointed.** Standards body endorsement requires evidence; no boolean stands alone. Standards whose bodies don't certify can't have requirements claiming endorsement.
6. **Copyright safety is enforced at corpus load.** License terms are data; the loader refuses to load restricted-license content into public-archive standards.

The corpus is a moat. Its enforcement properties are load-bearing.

---

## 2. Repo layout

```
intrinsic-code-standards/
├── README.md                          # This document
├── corpus_version.json                # Pinned corpus version + blueprint schema version + standards status
├── claim_policy.json                  # Corpus-level forbidden claim enforcement + regex test cases
├── auditor_registry.json              # Canonical list of registered auditors
├── standards/
│   ├── owasp-masvs/
│   │   ├── v2.1.0/
│   │   │   ├── manifest.json          # Standard-level metadata + standard_claim_policy
│   │   │   └── requirements/
│   │   │       ├── MASVS-STORAGE-1.json
│   │   │       ├── MASVS-CRYPTO-1.json
│   │   │       └── ...
│   │   └── archive/
│   ├── nist-800-63/
│   ├── pci-dss/
│   ├── hipaa/
│   └── ...
├── source-archives/
│   └── public/                        # Permissively-licensed sources only
│       └── owasp-masvs-v2.1.0.pdf
└── (Restricted-license sources stored privately, referenced by hash)
```

---

## 3. Requirement schema

Every requirement file is a single JSON object matching this schema. All fields required unless marked optional.

```json
{
  "standard_id": "string — uniquely identifies the requirement within the corpus",

  "normative_source": {
    "standard_name": "string",
    "standard_version": "string",
    "standard_version_date": "string — ISO 8601",
    "source_section": "string",
    "source_locator": "string — stable URL or page reference",
    "requirement_title": "string",
    "normative_excerpt": "string | null — short licensed excerpt; null if license forbids verbatim",
    "normative_summary": "string — Intrinsic Code's internal summary, always present",
    "verbatim_allowed": "boolean",
    "external_references": ["array of stable URLs"]
  },

  "source_rights": {
    "license": "string",
    "license_url": "string | null",
    "redistribution_allowed": "boolean",
    "public_archive_allowed": "boolean",
    "verbatim_excerpt_allowed": "boolean",
    "attribution_required": "boolean",
    "license_notes": "string | null"
  },

  "source_integrity": {
    "source_artifact_sha256": "string | null",
    "transcription_input_sha256": "string",
    "requirement_file_sha256": "string"
  },

  "applicability": {
    "tier_floor": "integer — 1-5",
    "conditions_logic": "string — 'all' (AND) or 'any' (OR)",
    "conditions": [
      {
        "field": "string — dotted path validated against blueprint_schema_version",
        "operator": "string — '==' | '!=' | '>' | '>=' | '<' | '<=' | 'in' | 'not_in' | 'exists' | 'not_exists'",
        "value": "any"
      }
    ],
    "human_note": "string — for humans and model context; NEVER parsed by code"
  },

  "intrinsic_interpretation": {
    "interpretation_version": "string",
    "mapping_strength": "string — 'direct' | 'partial' | 'supporting' | 'informational'",
    "verification_guidance": {
      "evidence_locator_types": [
        "array — allowed values: 'file_pattern', 'symbol_pattern', 'config_pattern', 'manifest_entry', 'network_call', 'string_literal', 'dependency_declaration', 'header_pragma', 'build_setting', 'code_comment', 'test_assertion', 'build_phase_script', 'entitlement_declaration'"
      ],
      "expected_evidence_patterns": ["array of strings"],
      "failure_indicators": ["array of strings"],
      "false_positive_traps": ["array of strings"]
    },
    "interpretation_review": {
      "interpretation_authority": "string — defaults to 'intrinsic_code_internal_review'",
      "review_status": "string — 'pre_audit' | 'externally_reviewed' | 'disputed' | 'deprecated'",
      "external_review_type": "string — 'none' | 'security_auditor_review' | 'compliance_consultant_review' | 'customer_security_team_review'",
      "external_reviewer": "string | null",
      "external_review_date": "string | null — ISO 8601",
      "external_review_scope": "string | null"
    }
  },

  "standards_body_endorsement": {
    "endorsed": "boolean — defaults to false",
    "endorsement_authority": "string | null",
    "endorsement_evidence_locator": "string | null",
    "endorsement_evidence_sha256": "string | null",
    "endorsement_evidence_public_accessibility": "boolean — defaults false; when false, external auditors must request evidence from founder",
    "endorsement_date": "string | null",
    "endorsement_scope": "string | null"
  },

  "claim_boundaries": {
    "allowed_claims": ["array of strings — REQUIREMENT-SPECIFIC additions to standard + corpus baselines"],
    "softened_claims": ["array of strings — REQUIREMENT-SPECIFIC additions"],
    "forbidden_claims": ["array of strings — REQUIREMENT-SPECIFIC additions to standard + corpus baselines"],
    "requires_external_certification": "boolean",
    "claim_boundary_notes": "string | null"
  },

  "intrinsic_mapping": {
    "constitution_versions": ["array of strings"],
    "constitutional_trace_ids": ["array of strings"],
    "receipt_fields": ["array of strings"],
    "auditor_consumers": ["array of strings — validated against auditor_registry.json"]
  },

  "lifecycle": {
    "status": "string — 'draft' | 'active' | 'deprecated' | 'superseded' | 'withdrawn'",
    "supersedes": ["array of standard_ids"],
    "superseded_by": ["array of standard_ids — bidirectional with supersedes"],
    "effective_date": "string — ISO 8601",
    "withdrawn_date": "string | null"
  },

  "transcription_metadata": {
    "corpus_version": "string",
    "transcribed_date": "string — ISO 8601"
  }
}
```

### Key field rules

**`standards_body_endorsement`** — load-bearing trust field, separate from any review fields.

Rule: When `endorsed == true`:
- `endorsement_authority` REQUIRED
- `endorsement_evidence_locator` REQUIRED
- `endorsement_evidence_sha256` REQUIRED
- `endorsement_evidence_public_accessibility` REQUIRED
- `endorsement_date` REQUIRED
- `endorsement_scope` REQUIRED

Corpus load FAILS if any active requirement has `endorsed: true` without all six evidence fields populated. Default is `endorsed: false` with all other fields null and `endorsement_evidence_public_accessibility: false`.

Additional rule (new in v0.1.3, enforced by LOAD-14): if the parent standard's `manifest.json.standard_claim_policy.standards_body_certification_available: false`, then no requirement under that standard may have `endorsed: true`.

**`interpretation_review.external_review_type`** — uses `"none"` when `review_status: "pre_audit"`. Replaces ambiguous null. `"standards_body_endorsement"` is NOT in this enum.

**`claim_boundaries`** — per-requirement claim language rules. **Adds to** standard-level (`manifest.json.standard_claim_policy.*`) and corpus-level (`claim_policy.json`) baselines via three-tier inheritance. Cannot un-forbid claims forbidden at standard or corpus level.

**`applicability.conditions[]`** — every `field` is a dotted path validated against the blueprint JSON schema. Corpus load FAILS on unresolvable paths.

**`intrinsic_mapping.auditor_consumers`** — every entry validated against `auditor_registry.json`. Corpus load FAILS on unknown entries.

**`lifecycle.supersedes` / `superseded_by`** — bidirectional validation. Orphan supersession → corpus load fails.

**`normative_excerpt` vs `source_rights.verbatim_excerpt_allowed`** — LOAD-13 enforces. If verbatim not allowed but excerpt is non-null → load fails.

---

## 4. Per-standard `manifest.json`

Every standard version directory has a `manifest.json` summarizing what's inside.

```json
{
  "standard_id_prefix": "MASVS",
  "standard_name": "OWASP Mobile Application Security Verification Standard",
  "standard_version": "2.1.0",
  "standard_version_date": "2024-01-18",
  "source_archive_public": "../../../source-archives/public/owasp-masvs-v2.1.0.pdf",
  "source_archive_private_hash": null,
  "requirement_count": 24,
  "requirement_ids": [
    "MASVS-STORAGE-1", "MASVS-STORAGE-2",
    "MASVS-CRYPTO-1", "MASVS-CRYPTO-2",
    "MASVS-AUTH-1", "MASVS-AUTH-2", "MASVS-AUTH-3",
    "..."
  ],
  "superseded_versions": [],
  "current": true,
  "added_to_corpus_version": "v0.1.0",
  "last_reviewed_corpus_version": "v0.1.0",

  "standard_claim_policy": {
    "standards_body_certification_available": "boolean — true only if the source standards body issues formal certifications",
    "certification_disclaimer_present": "boolean — true if the source standard or standards body publishes an explicit certification disclaimer",
    "certification_disclaimer": "string | null — verbatim or paraphrased disclaimer text; REQUIRED when certification_disclaimer_present: true",
    "certification_disclaimer_source_section": "string | null — section/clause of source where disclaimer appears; REQUIRED when certification_disclaimer_present: true",
    "standard_baseline_forbidden_claims": [
      "array of strings — additions to corpus-level baseline_forbidden_claims that apply to every requirement in this standard"
    ],
    "standard_baseline_allowed_claims": [
      "array of strings — preferred claim language for requirements in this standard"
    ],
    "standard_baseline_softened_claims": [
      "array of strings — softened claim language specific to this standard"
    ],
    "standard_policy_notes": "string | null — additional clarifications on standard-wide claim posture"
  }
}
```

For restricted-license standards: `source_archive_public` is null, `source_archive_private_hash` set. External auditors acquire source from standards body and verify against hash.

### `standard_claim_policy` semantics

**`standards_body_certification_available`** — set `true` only if the source standards body itself issues formal certifications (e.g., PCI Council issues PCI-DSS QSA certifications). Set `false` for bodies that publish standards without certifying compliance with them (e.g., OWASP, NIST in most cases).

**`certification_disclaimer_present`** — set `true` if the source standard or standards body publishes an explicit certification disclaimer. Independent of `standards_body_certification_available` because a body might offer no certification without ever publishing a disclaimer. When `true`, `certification_disclaimer` and `certification_disclaimer_source_section` are required (LOAD-14).

**`standard_baseline_forbidden_claims`** — additions to the corpus-level `baseline_forbidden_claims`. Applied to every requirement under this standard automatically. Example: "OWASP-certified" goes here once, not in 24 requirement files.

**`standard_baseline_allowed_claims`** — preferred language for this standard. Example for MASVS: "evaluated against OWASP MASVS controls."

**`standard_baseline_softened_claims`** — softened language additions. Example: "controls described by MASVS."

### Inheritance semantics

At factory preprocessor time and at aggregator enforcement time, the effective claim policy for any requirement is computed as:

**Effective forbidden_claims** = `claim_policy.json.baseline_forbidden_claims` ∪ `manifest.json.standard_claim_policy.standard_baseline_forbidden_claims` ∪ `requirement.claim_boundaries.forbidden_claims`

**Effective allowed_claims** = `claim_policy.json.baseline_allowed_claims` (if any) ∪ `manifest.json.standard_claim_policy.standard_baseline_allowed_claims` ∪ `requirement.claim_boundaries.allowed_claims`

**Effective softened_claims** = `claim_policy.json.baseline_softened_claims` ∪ `manifest.json.standard_claim_policy.standard_baseline_softened_claims` ∪ `requirement.claim_boundaries.softened_claims`

Three tiers: corpus → standard → requirement. Each tier adds; nothing overrides downward. A forbidden claim at corpus or standard level stays forbidden everywhere; per-requirement entries cannot un-forbid.

---

## 5. `claim_policy.json`

Top-level corpus file. Read by Python aggregator at signing time. Curated by founder.

```json
{
  "policy_version": "v0.1.0",
  "baseline_forbidden_claims": [
    "certified by", "certified to", "is certified",
    "compliant with", "fully compliant",
    "approved by", "authorized under", "authorized by",
    "endorsed by", "officially endorsed", "blessed by",
    "OWASP-approved", "OWASP approved",
    "NIST-approved", "NIST approved",
    "PCI-certified", "PCI certified", "PCI compliant",
    "HIPAA compliant", "HIPAA certified",
    "FedRAMP authorized", "FedRAMP-approved",
    "ISO certified",
    "vulnerability-free", "audit-proof",
    "zero vulnerabilities", "zero defects",
    "perfectly secure"
  ],
  "baseline_forbidden_patterns": [
    "\\b\\w+-approved\\b",
    "\\b\\w+-certified\\b",
    "\\bapproved by \\w+\\b",
    "\\bcertified (by|to) \\w+\\b",
    "\\bendorsed by \\w+\\b",
    "\\bauthorized (by|under) \\w+\\b"
  ],
  "baseline_softened_claims": [
    "addresses", "aligned with", "mapped to",
    "evaluated against", "supports evidence for",
    "implements controls described by",
    "demonstrates coverage of", "references"
  ],
  "test_cases": [
    {
      "pattern": "\\b\\w+-approved\\b",
      "must_match": ["OWASP-approved", "NIST-approved", "FedRAMP-approved"],
      "must_not_match": ["pre-approved by code review", "preapproved internally", "approved language"]
    },
    {
      "pattern": "\\b\\w+-certified\\b",
      "must_match": ["PCI-certified", "ISO-certified"],
      "must_not_match": ["self-certified internal review", "certified mail"]
    },
    {
      "pattern": "\\bapproved by \\w+\\b",
      "must_match": ["approved by OWASP", "approved by NIST"],
      "must_not_match": ["approved", "approved language"]
    },
    {
      "pattern": "\\bcertified (by|to) \\w+\\b",
      "must_match": ["certified by ISO", "certified to PCI-DSS"],
      "must_not_match": ["certified", "certified statement"]
    },
    {
      "pattern": "\\bendorsed by \\w+\\b",
      "must_match": ["endorsed by OWASP", "endorsed by NIST"],
      "must_not_match": ["endorsement letter", "endorsed"]
    },
    {
      "pattern": "\\bauthorized (by|under) \\w+\\b",
      "must_match": ["authorized under FedRAMP", "authorized by HHS"],
      "must_not_match": ["authorized signatory", "authorized"]
    }
  ],
  "enforcement_scope": [
    "auditor_summary",
    "verdict_reasoning",
    "finding.description",
    "finding.attack_scenario",
    "bundle_audit_review.*",
    "external_standards_review.*"
  ],
  "policy_notes": "Python aggregator enforces deterministically via string match (corpus baseline_forbidden_claims + every applicable standard's standard_baseline_forbidden_claims + every cited requirement's claim_boundaries.forbidden_claims) AND regex match (baseline_forbidden_patterns) on every text field listed in enforcement_scope. Model-internal thinking and logs are exempt. Three-tier inheritance: corpus → standard → requirement. Models may flag advisory concerns; only Python blocks signing. Corpus loader runs every entry in test_cases at load time; mismatch fails load."
}
```

**Enforcement specification:**

At signing time, the Python aggregator:

1. Collects every text field listed in `enforcement_scope` from the bundle audit output that will appear in the Receipt.
2. Computes effective forbidden list = corpus baseline ∪ all applicable standards' baselines ∪ all cited requirements' forbidden_claims (three-tier union).
3. Runs case-insensitive substring match against the effective forbidden list.
4. Runs regex match against `baseline_forbidden_patterns`.
5. ANY match → verdict `fail` with reason `claim_boundary_violation`; Receipt is NOT signed at production grade.

Model-internal thinking, run logs, debug output, and any field NOT in `enforcement_scope` are exempt. No semantic interpretation. Boring deterministic matching.

---

## 6. `auditor_registry.json`

Top-level corpus file. Canonical list of every registered auditor that may appear in `intrinsic_mapping.auditor_consumers`.

```json
{
  "registry_version": "v0.1.0",
  "registry_updated_date": "2026-05-19",
  "registered_auditors": [
    {
      "auditor_id": "file_standards_auditor",
      "prompt_file": "prompts/file-standards-auditor-prompt.md",
      "domain": "per-file constitutional trace + forbidden primitive scan",
      "status": "active",
      "promoted_to_active_date": "2026-05-09"
    },
    {
      "auditor_id": "file_implementation_auditor",
      "prompt_file": "prompts/file-implementation-auditor-prompt.md",
      "domain": "per-file TODO completeness + NIAP attestation structure (Tier 5)",
      "status": "active",
      "promoted_to_active_date": "2026-05-09"
    },
    {
      "auditor_id": "reaper-auth",
      "prompt_file": "prompts/reaper-auth-prompt.md",
      "domain": "adversarial audit of authentication and authorization",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-privacy",
      "prompt_file": "prompts/reaper-privacy-prompt.md",
      "domain": "adversarial audit of privacy, data handling, and PII",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-secrets",
      "prompt_file": "prompts/reaper-secrets-prompt.md",
      "domain": "adversarial audit of secret handling and key management",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-mobile",
      "prompt_file": "prompts/reaper-mobile-prompt.md",
      "domain": "adversarial audit of mobile platform attack surface",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-input",
      "prompt_file": "prompts/reaper-input-prompt.md",
      "domain": "adversarial audit of input validation and untrusted data",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-dependencies",
      "prompt_file": "prompts/reaper-dependencies-prompt.md",
      "domain": "adversarial audit of third-party dependencies",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-crypto",
      "prompt_file": "prompts/reaper-crypto-prompt.md",
      "domain": "adversarial audit of cryptographic primitives",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "reaper-compliance",
      "prompt_file": "prompts/reaper-compliance-prompt.md",
      "domain": "adversarial audit of regulatory compliance posture (Tier 4-5)",
      "status": "active",
      "promoted_to_active_date": "2026-05-10"
    },
    {
      "auditor_id": "bundle_coherence_auditor",
      "prompt_file": "prompts/bundle-coherence-auditor-v0-1-prompt.md",
      "domain": "post-Reaper bundle coherence + Receipt section integrity",
      "status": "planned",
      "planned_since_date": "2026-05-19"
    },
    {
      "auditor_id": "build_integrity_auditor",
      "prompt_file": "prompts/build-integrity-auditor-v0-1-prompt.md",
      "domain": "post-Reaper build system completeness + dead code",
      "status": "planned",
      "planned_since_date": "2026-05-19"
    },
    {
      "auditor_id": "external_standards_mapping_auditor",
      "prompt_file": "prompts/external-standards-mapping-auditor-v0-1-prompt.md",
      "domain": "post-Reaper external standards mapping verification",
      "status": "planned",
      "planned_since_date": "2026-05-19"
    },
    {
      "auditor_id": "cross_file_consistency_auditor",
      "prompt_file": "prompts/cross-file-consistency-auditor-v0-1-prompt.md",
      "domain": "post-Reaper cross-file contradiction detection",
      "status": "planned",
      "planned_since_date": "2026-05-19"
    }
  ]
}
```

**Status values:**
- `active` — auditor exists in production prompts directory. Includes `promoted_to_active_date`.
- `planned` — design locked, implementation not shipped. Includes `planned_since_date`. LOAD-12 flags referencing requirements if `planned_since_date` exceeds 90 days.
- `deprecated` — auditor readable for historical Receipt verification only.

Founder owns the registry; new auditor shipped → registry updated in same commit as the prompt file.

---

## 7. `corpus_version.json`

Root-level file tracking the whole corpus state.

```json
{
  "corpus_version": "v0.1.0",
  "corpus_version_date": "2026-05-19",
  "blueprint_schema_version": "1.0",
  "blueprint_schema_path": "orchestrator/schemas/blueprint.schema.json",
  "constitution_mapping_file_locator": {
    "path_in_main_repo": "mappings/constitution_to_external_standards.json",
    "expected_mapping_file_version": "1.1",
    "expected_constitution_version": "0.4",
    "main_repo_visibility": "private",
    "external_auditor_access_note": "Mapping file lives in the private factory repo. External auditors verifying a Receipt may request the mapping file at the corpus commit hash from the founder via support@beforrealmedia.com; verify against the corpus version's expected_mapping_file_version field."
  },
  "standards": {
    "MASVS": {
      "status": "not_transcribed",
      "current_version": null,
      "verified_at": null,
      "reason": "Pending Round 1 corpus authoring"
    },
    "NIST-800-63": {
      "status": "not_transcribed",
      "current_version": null,
      "verified_at": null,
      "reason": "Pending Round 1 corpus authoring; founder must verify current published revision"
    },
    "PCI-DSS": {
      "status": "not_transcribed",
      "current_version": null,
      "verified_at": null,
      "reason": "Tier 4 dependency"
    },
    "HIPAA": {
      "status": "not_transcribed",
      "current_version": null,
      "verified_at": null,
      "reason": "Tier 4 dependency"
    }
  },
  "review_cadence_days": 90,
  "next_review_due": "2026-08-17"
}
```

**`blueprint_schema_version` rule:** at corpus load, every requirement's `applicability.conditions[].field` is validated against the schema at `blueprint_schema_path` for the pinned version. Invalid path → load fails.

**Standards status rule:** no standard moves from `not_transcribed` to `active` without `verified_at` timestamp.

---

## 8. Reading the corpus

### Factory (preprocessor)

1. Read `corpus_version.json`. Validate `blueprint_schema_version` matches orchestrator's current blueprint schema. Mismatch → orchestrator init fails.
2. Read `claim_policy.json`. Cache baseline forbidden lists, regex patterns, enforcement scope, test cases.
3. Read `auditor_registry.json`. Cache registered auditor IDs and statuses.
4. Run corpus load validation (§9). Any failure → factory init fails.
5. For each entry in bundle's `external_standards_mapping[]`, parse `standard_id` prefix.
6. Look up current version from `standards[prefix].current_version`. If `status != "active"`, preprocessor refuses → factory emits `audit_incomplete`.
7. Load standard's `manifest.json` and cache `standard_claim_policy` for the standard.
8. Load requirement files: `standards/{prefix-lowercased}/v{version}/requirements/{standard_id}.json`.
9. Evaluate `applicability.conditions[]` deterministically against the bundle blueprint. Inapplicable requirements skipped with `not_applicable`.
10. For applicable requirements, extract relevant fields and feed into External Standards Mapping Auditor's input.
11. After auditor runs, Python aggregator computes effective forbidden_claims via three-tier inheritance and enforces deterministically on scoped fields per §5.
12. Record corpus_version, policy_version, and standard_claim_policy versions in Receipt's standards manifest preprocessor output.

### External auditor

1. Pull `intrinsic-code-standards` at commit hash in Receipt's `corpus_version`.
2. Pull `claim_policy.json` and `auditor_registry.json` at the same commit.
3. Read requirement files used by the Receipt + their parent standard `manifest.json` files.
4. For restricted-license sources, acquire from standards body, verify against `source_integrity.source_artifact_sha256`.
5. For private endorsement evidence (`endorsement_evidence_public_accessibility: false`), request from founder, verify against `endorsement_evidence_sha256`.
6. Reproduce audit independently.

Submodule pin + per-file hashes + corpus version = forensic-grade reproducibility.

---

## 9. Corpus load validation rules

At every corpus load (factory init OR external auditor verification), the loader runs all checks below. ANY failure → corpus load fails with specific error citing the requirement and rule violated. No partial loads.

**LOAD-1: Schema conformance.** Every requirement file conforms to §3 schema; every manifest conforms to §4 schema (including `standard_claim_policy`). Missing required fields, wrong types, unknown enum values → fail.

**LOAD-2: Blueprint schema validation.** Every `applicability.conditions[].field` resolves against the blueprint schema at `corpus_version.json.blueprint_schema_version`. Unresolvable path → fail with `invalid_condition_field`.

**LOAD-3: Operator validity.** Every `applicability.conditions[].operator` is in the allowed list. Unknown operator → fail.

**LOAD-4: Auditor registry validation.** Every `intrinsic_mapping.auditor_consumers[]` entry resolves to an auditor in `auditor_registry.json`. Unknown auditor → fail.

**LOAD-5: Standards body endorsement evidence.** If `standards_body_endorsement.endorsed == true`, all six evidence fields MUST be non-null. Missing evidence → fail.

**LOAD-6: Lifecycle bidirectional supersedes.** For every requirement A listing B in `lifecycle.supersedes`, requirement B must list A in `lifecycle.superseded_by`. Orphan supersession → fail.

**LOAD-7: Lifecycle status consistency.** Status `superseded` requires non-empty `superseded_by`. Status `active` requires empty `superseded_by`. Status `withdrawn` requires non-null `withdrawn_date`. Inconsistent → fail.

**LOAD-8: Source integrity hashes.** `source_integrity.requirement_file_sha256` must match actual file content hash (excluding the hash field itself). Mismatch → fail.

**LOAD-9: Manifest consistency.** Every `manifest.json` `requirement_ids[]` entry must correspond to an actual file in `requirements/`. Phantom IDs → fail. Files in `requirements/` not listed in manifest → fail.

**LOAD-10: Claim policy load.** `claim_policy.json` MUST parse successfully and contain non-empty `baseline_forbidden_claims`, `baseline_forbidden_patterns`, and `enforcement_scope`. Empty or missing → fail.

**LOAD-11: Regex test cases.** Every entry in `claim_policy.json.test_cases` is run. For each, every string in `must_match` MUST match the pattern (case-insensitive) and every string in `must_not_match` MUST NOT match. Mismatch → corpus load fails with citation of the failing pattern and failed test string.

**LOAD-12: Planned auditor zombie detection.** For every auditor in `auditor_registry.json` with `status: planned`, compute `days_since_planned = today - planned_since_date`. If `> 90`, flag all requirements referencing this auditor as `requires_quarterly_review`. Generate flag report (NOT a load failure); flag report consumed by quarterly review process.

**LOAD-13: License safety on normative_excerpt.** For every requirement, if `source_rights.verbatim_excerpt_allowed == false` AND `normative_source.normative_excerpt != null` → fail with `license_violation`.

**LOAD-14: Standard claim policy consistency.** Two checks per manifest:
- If `certification_disclaimer_present: true`, then `certification_disclaimer` and `certification_disclaimer_source_section` MUST be non-null. Missing → fail with `missing_certification_disclaimer`.
- If `standards_body_certification_available: false`, then no active requirement under this standard may have `standards_body_endorsement.endorsed: true`. Violation → fail with `certification_unavailable_but_endorsement_claimed` citing the offending requirement.

**LOAD-15: Constitution trace ID validation.** For every requirement file, every entry in `intrinsic_mapping.constitutional_trace_ids[]` MUST resolve to a `trace_id` value in the authoritative Constitution-to-external-standards mapping file (`mappings/constitution_to_external_standards.json` at the version specified in `intrinsic_mapping.constitution_versions[]` and located via `corpus_version.json.constitution_mapping_file_locator.path_in_main_repo`).

Unknown trace ID → fail with `unknown_constitutional_trace_id` citing the offending trace ID, the requirement file, and the resolved mapping file path. Declared constitution version with no available mapping file → fail with `mapping_file_version_unavailable`.

**LOAD-15 soft-fail mode:** when the loader runs in external-auditor context without main-repo access (mapping file unreachable), LOAD-15 emits a warning (`mapping_file_unavailable_for_load_15_check`) rather than a hard failure. Honest disclosure rather than silent degradation; external auditors then know to request the mapping file from the founder at the corpus commit hash to complete their audit.

The loader stops at the first failure (LOAD-1 through LOAD-10, LOAD-13, LOAD-14, LOAD-15 in factory mode) and emits the full set of failing rules + offending IDs. LOAD-11 runs all test cases before failing (so founder sees the full set of broken patterns). LOAD-12 is non-failing — produces flag report consumed at quarterly review. LOAD-15 in soft-fail mode is also non-failing.

---

## 10. Corpus governance

### 10.1 Roles

| Role | Owner | Responsibility |
|---|---|---|
| Standard selection | Founder | What's in the corpus, in what priority order |
| Source acquisition + version verification | Founder | Acquire authoritative source, verify version, archive public or private per license, record hash |
| First-pass transcription | Claude Opus 4.7 | Produce requirement files + manifest matching schema |
| Cross-check pass | GPT | Verify against source; flag missing, drift, ID mismatches, license issues |
| Red-team pass | Grok | Attack transcription; surface escape hatches, missed nuance, claim-boundary leaks |
| Final sign-off | Founder (with Matthew advising on security) | Manual review, commit, version stamp |

### 10.2 Process per standard version

1. Source acquisition + version verification (founder)
2. License determination + archive placement (founder)
3. First-pass transcription including manifest with `standard_claim_policy` (Claude)
4. Cross-check (GPT) → delta report
5. Red-team (Grok) → attack report
6. Resolution (founder) — disagreements resolved, transcription updated, resolutions live in commit messages not document
7. Sign-off (founder commits, corpus version bumps, standard status flips to `active`)
8. Main repo submodule bump (CC pulls new commit)

### 10.3 Quarterly review

Every 90 days:

1. Founder checks each active standard for new revisions
2. Revised standards → full re-transcription per §10.2; old version → `deprecated` status
3. Unchanged standards → founder reviews `intrinsic_interpretation.verification_guidance` for currency; updates `transcribed_date` if changed
4. Consume LOAD-12 flag report: for every requirement flagged `requires_quarterly_review`, founder decides — extend planned timeline, deprecate the reference, or escalate to ship the auditor. Decision logged in commit message.
5. Update `corpus_version.json.next_review_due`

### 10.4 Auditor registry maintenance

Every new auditor shipped → `auditor_registry.json` updated in same commit as the prompt file. Status `planned` only for auditors whose design is locked but implementation hasn't shipped. Once shipped, status → `active` with `promoted_to_active_date` set.

Deprecated auditors stay in registry with `deprecated` status for historical Receipt verification.

### 10.5 Source correction policy

When a standards body issues errata or corrections:

1. **Never mutate existing requirement files.** Corpus commits referenced by signed Receipts must remain hash-stable forever.
2. Create a new corpus version with the corrected source incorporated.
3. Move the old version's standard to `deprecated` status.
4. Add the new version as `active`.
5. Update `manifest.json` in the old version's directory to reflect deprecation.
6. External auditors of older Receipts reference historical corpus commit + old requirement state — correct behavior.
7. Disclose the correction publicly in changelog.

Corpus is append-only at commit level. Errata = version forward, never edit back.

---

## 11. What this corpus does NOT contain

- Factory implementation guidance for code generation (lives in factory prompts and Constitution)
- AI-generated commentary beyond schema fields
- Vendor-specific advisory content (only normative requirements; advisory lives in `external_references`)
- Forbidden claim language presented as allowed (three-tier inheritance enforced at signing)
- Endorsement claims without evidence (LOAD-5 + LOAD-14 enforced)
- Restricted normative text in the public repo (LOAD-13 enforced)
- Per-requirement duplication of standard-wide claim posture (manifest `standard_claim_policy` inheritance enforced)

---

## 12. Failure modes & guardrails

| Failure mode | Mitigation |
|---|---|
| Requirement drifts from source | GPT cross-check |
| `verification_guidance` is gameable | Grok red-team |
| Bundle claims compliance with standard not in corpus | Preprocessor refuses; factory emits `audit_incomplete` |
| Stale standard version claimed | Preprocessor flags version mismatch |
| Standards body publishes revision; corpus out of date | Quarterly review surfaces |
| AIs disagree during transcription | Founder resolves; resolution in commit history |
| Source becomes unavailable | Source archive (public or private) preserves; hash enables external verification |
| Factory emits forbidden claim language | Aggregator deterministic enforcement on scoped fields with three-tier inheritance blocks signing |
| Endorsement claimed without evidence | LOAD-5 failure |
| Endorsement claimed for non-certifying body | LOAD-14 failure |
| Disclaimer declared present but text missing | LOAD-14 failure |
| Applicability condition references nonexistent blueprint field | LOAD-2 failure |
| Auditor consumer references unknown auditor | LOAD-4 failure |
| Orphan supersession | LOAD-6 failure |
| Restricted-license source can't be redistributed | Stored privately; corpus references via locator + hash |
| Standards body issues errata on older release | §10.5 — new version, old deprecated, never mutate |
| Regex pattern broken by founder edit | LOAD-11 test cases fail load |
| Planned auditor never ships | LOAD-12 flags referencing requirements for quarterly review |
| Restricted normative text accidentally pasted into public requirement | LOAD-13 load failure |
| Standard-wide forbidden claims missed by per-requirement files | Manifest inheritance applies standard baseline to all requirements automatically |
| Requirement references nonexistent Constitution trace ID | LOAD-15 corpus load failure (factory mode); soft-fail warning (external auditor mode without mapping file access) |
| Constitution mapping file version mismatch with corpus `constitution_versions` declaration | LOAD-15 corpus load failure with `mapping_file_version_unavailable` |
| External auditor cannot resolve trace IDs without main-repo mapping file | LOAD-15 soft-fail mode emits explicit warning; corpus loader documents the gap; auditor requests mapping file from founder |

---

## 13. Versioning

Semver:
- **Major (1.0 → 2.0):** breaking schema change
- **Minor (0.1 → 0.2):** new standard added or existing standard bumped to new version
- **Patch (0.1.2 → 0.1.3):** additive schema enhancements (e.g., `standard_claim_policy`) or edits to existing requirements

Every Receipt records corpus version used. Pulling at that commit reproduces audit conditions.

---

## 14. Operating principle

> The corpus governs what the factory CAN claim. Forbidden language is data, enforced via three-tier inheritance. Endorsement defaults to false and requires evidence. Source is license-aware. Interpretation is versioned independently from source. Audit conditions are commit-pinned. Trust claims cannot be self-anointed. Copyright is enforced at corpus load. Standard-wide claim posture lives once at the manifest level, not 24 times across requirement files.

This is the moat.

---

*v0.1.4 — additive Constitution trace ID validation. Lock candidate. Awaiting founder ratification.*
