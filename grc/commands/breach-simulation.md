# /grc:breach-simulation

Generate a data breach simulation scenario with phase-by-phase ISO 27001:2022 control failure analysis and a prioritized remediation plan.

## Usage

```
/grc:breach-simulation [scenario-type] [organization-profile]
```

**Arguments** (all optional):
- `scenario-type` — Type of breach: `credential-compromise` (default), `ransomware`, `insider-threat`, `supply-chain`, `physical`
- `organization-profile` — Brief description of the organization (industry, size, cloud/on-prem, certification status)

**Examples**:
```
/grc:breach-simulation
/grc:breach-simulation credential-compromise "SaaS provider, 200 employees, AWS, ISO 27001 certified"
/grc:breach-simulation ransomware "manufacturing company, on-prem, pursuing ISO 27001"
/grc:breach-simulation insider-threat "financial services, hybrid cloud, 500 employees"
```

---

## Behavior

When this command is invoked, generate a structured breach simulation using the reference material in `grc/skills/grc-knowledge/audits/iso27001-breach-simulation.md` as the canonical template. Adapt the scenario to the specified type and organization profile.

### Output Structure

Produce the following sections:

#### 1. Scenario Overview
- Organization profile (adapted to input or using a realistic default)
- Attack summary: vector, timeline, data exposed
- Severity classification

#### 2. Attack Timeline — Phase by Phase
For each phase (Initial Access → Privilege Escalation → Discovery/Lateral Movement → Exfiltration → Detection):
- Narrative description of what happened
- Table of ISO 27001:2022 Annex A controls that failed, with control ID, title, and specific failure description

#### 3. Remediation Plan
Organize by timeframe:
- **Immediate (0–72 hours)**: Containment and preservation actions with control references
- **Short-term (1–30 days)**: Control-specific remediations with concrete implementation steps
- **Medium-term (30–90 days)**: Systemic process and architectural fixes
- **Long-term (90+ days)**: ISMS-level improvements, Clause 10.2 corrective actions, independent review

For each remediation item include:
- ISO 27001:2022 control ID and title
- Root cause of the failure
- Specific remediation steps
- Evidence the organization should retain
- How to verify effectiveness (for corrective action closure)

#### 4. Risk Register Updates
List new risk entries to add to the ISMS risk register, with threat, vulnerability, current control gap, preliminary risk rating, and treatment option.

#### 5. Auditor Evidence Checklist
Table of controls → expected audit evidence for the next surveillance or recertification audit.

#### 6. Debrief Questions
5–8 discussion questions for post-exercise or management review.

---

## Tone and Format

- Use ISO 27001:2022 control IDs and titles exactly (e.g., "A.8.12 — Data leakage prevention")
- Reference ISMS Clauses (4-10) where management system failures are involved
- Use tables for control-failure mappings and evidence checklists
- Be specific: name concrete tools, policies, and process steps rather than generic guidance
- Note when a finding would constitute a Major vs. Minor Nonconformity under ISO 27001 audit criteria

---

## Data Sensitivity Reminder

> **Before sharing:** Replace actual organization names, system names, IP addresses, and real incident details with placeholders (e.g., `[Organization Name]`, `[System Name]`, `10.x.x.x`). This command generates structural GRC documentation — it does not assess whether your actual systems are secure.
