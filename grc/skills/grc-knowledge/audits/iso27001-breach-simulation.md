# ISO 27001 Data Breach Simulation & Remediation Reference

Structured scenario for tabletop exercises and post-incident review. Maps each breach phase to the ISO 27001:2022 Annex A controls that failed, what evidence the auditor would expect, and specific remediation actions required.

---

## Scenario: Healthcare SaaS Provider — Credential Compromise & Data Exfiltration

**Organization Profile**: Mid-size healthcare SaaS provider processing patient records for regional clinics. ISO 27001:2022 certified. ~50 employees, cloud-hosted infrastructure.

**Classification**: Confidentiality breach (PII/PHI)
**Scope**: 62,000 patient records — names, dates of birth, diagnosis codes, insurance IDs
**Discovery**: 74 hours after initial compromise
**Attack Vector**: Spear phishing → MFA bypass → lateral movement → staged exfiltration

---

## Attack Timeline

### Phase 1 — Initial Access (Hour 0)

**What happened**: A customer support representative received a spear-phishing email impersonating an internal IT helpdesk notification. The email contained a link to a spoofed login portal that harvested the user's credentials and intercepted their SMS OTP code in real time (real-time phishing proxy).

**Controls that failed**:

| Control ID | Title | How it failed |
|------------|-------|---------------|
| A.6.3 | Information security awareness, education and training | Employee had not received phishing simulation training in 18 months. Training records showed the last company-wide exercise was completed only at onboarding. |
| A.5.17 | Authentication information | SMS-based OTP was the only MFA factor. SMS OTP is susceptible to real-time phishing proxies; phishing-resistant MFA (FIDO2/WebAuthn) was not required. |
| A.8.23 | Web filtering | The phishing domain was 4 days old. Web filtering was deployed but DNS-layer blocking was not active for the user's remote working environment. |
| A.6.7 | Remote working | Remote working policy did not mandate VPN-split-tunnel enforcement or endpoint security verification before access to production portals. |

---

### Phase 2 — Privilege Escalation (Hours 1–6)

**What happened**: Using the compromised support account, the attacker explored the internal admin portal and discovered that the account had been over-provisioned six months earlier during a product migration. It retained a legacy "data-export" role that allowed bulk CSV export of patient records without requiring a secondary approval workflow.

**Controls that failed**:

| Control ID | Title | How it failed |
|------------|-------|---------------|
| A.5.15 | Access control | The access control policy did not enforce least-privilege reviews on a defined schedule. Access rights assigned during the migration were never reviewed for continued necessity. |
| A.5.18 | Access rights | There was no formal access review process. The quarterly access recertification campaign had lapsed — the last completed review was 9 months prior. |
| A.8.2 | Privileged access rights | The "data-export" role was classified as a standard application role rather than a privileged role. Privileged access management controls (just-in-time, dual-approval) therefore did not apply to it. |
| A.5.3 | Segregation of duties | A single account combining customer support and bulk data export functions violated segregation of duties. No compensating detective control (e.g., export volume alerting) was in place. |

---

### Phase 3 — Discovery & Lateral Movement (Hours 6–18)

**What happened**: The attacker queried the patient record API to map the data schema, then moved laterally to a secondary microservice that held insurance claim data. The lateral move was possible because inter-service authentication used long-lived static API keys stored in a shared secrets manager with overly broad access policies.

**Controls that failed**:

| Control ID | Title | How it failed |
|------------|-------|---------------|
| A.8.3 | Information access restriction | The API did not enforce record-level authorization. Any valid session token could query all patient records regardless of the account's assigned clinic scope. |
| A.8.9 | Configuration management | Static API keys had not been rotated per the defined 90-day rotation policy. Keys were 8 months old. Configuration baseline review had not flagged this drift. |
| A.5.19 | Information security in supplier relationships | The secrets manager was a third-party SaaS tool. The supplier relationship review had not evaluated whether the tool's access policies aligned with the organization's least-privilege requirements. |
| A.8.22 | Segregation of networks | Patient records service and insurance claims service shared the same internal network segment. There was no micro-segmentation preventing lateral movement between services after initial authentication. |

---

### Phase 4 — Exfiltration (Hours 18–72)

**What happened**: Over 54 hours, the attacker staged 62,000 records into a temporary storage bucket using the data-export role, then exfiltrated them in 12 increments of ~5,000 records each to an external endpoint disguised as a legitimate analytics integration. Each increment was below the DLP volume threshold.

**Controls that failed**:

| Control ID | Title | How it failed |
|------------|-------|---------------|
| A.8.12 | Data leakage prevention | DLP rules were volume-based only (alerting on transfers >10,000 records). The attacker evaded detection by staying below the threshold per transfer. Content-aware DLP (detecting PHI field patterns) was not implemented. |
| A.8.15 | Logging | API export logs were retained for only 14 days in the hot tier. Log shipping to the SIEM was delayed by 6 hours due to a misconfigured pipeline. The anomalous export pattern was not captured in near-real-time. |
| A.8.16 | Monitoring activities | SIEM correlation rules existed for single large exports but not for repeated medium-volume exports from the same account over time. Behavioral baseline alerting was not configured. |
| A.5.7 | Threat intelligence | The external destination IP had been flagged in a commercial threat intelligence feed 11 days earlier as a known data staging endpoint. The threat intelligence feed was subscribed to but not integrated into firewall block lists or SIEM enrichment. |

---

### Phase 5 — Discovery & Response (Hour 74)

**What happened**: A routine data integrity check by an internal analyst noticed an unexplained spike in the API export audit log. The analyst escalated to the security team, which confirmed the breach 2 hours later. The incident response plan was invoked, but the plan did not include a specific runbook for credential compromise scenarios, causing a 4-hour delay in containment.

**Controls that failed**:

| Control ID | Title | How it failed |
|------------|-------|---------------|
| A.5.24 | Information security incident management planning and preparation | The incident response plan existed but lacked scenario-specific playbooks. The plan referenced generic steps without covering credential-based exfiltration, leading to decision paralysis during the first hours. |
| A.5.25 | Assessment and decision on information security events | The security team had no documented triage criteria for distinguishing security events from incidents. The initial escalation sat in the ticket queue for 90 minutes before being classified as a potential incident. |
| A.5.28 | Collection of evidence | Forensic evidence collection procedures were not followed at the outset. The compromised account's active sessions were terminated before a forensic snapshot was taken, destroying volatile evidence needed to determine the full scope. |
| A.8.17 | Clock synchronization | Timestamps across the API server, SIEM, and storage service were not synchronized (drifts of up to 90 seconds). This complicated timeline reconstruction during forensic analysis. |

---

## Remediation Plan — ISO 27001:2022 Control Mapping

### Immediate Actions (0–72 hours post-discovery)

| Action | ISO 27001 Control | Owner |
|--------|-------------------|-------|
| Revoke compromised account credentials and all active sessions | A.5.16, A.5.17 | IAM / Security Operations |
| Rotate all static API keys for affected services | A.8.9 | Platform Engineering |
| Isolate affected network segments pending forensic review | A.8.22 | Network / Cloud Ops |
| Preserve all logs and snapshots before any remediation | A.5.28 | Security Operations |
| Notify affected individuals and regulatory authorities per legal obligations | A.5.31, A.5.34 | Legal / DPO / CISO |
| Activate incident response team and assign Incident Commander | A.5.24, A.5.26 | CISO |

---

### Short-Term Remediation (1–30 days)

#### A.5.17 — Authentication Information
**Finding**: SMS OTP susceptible to real-time phishing proxy attacks.
**Remediation**:
- Mandate phishing-resistant MFA (FIDO2 hardware tokens or passkeys) for all accounts with access to production systems and PII.
- Remove SMS OTP as an accepted factor for privileged and production access.
- Document revised authentication requirements in the Access Control Policy.
- Update the SoA to reflect enhanced implementation status.

#### A.5.15 / A.5.18 — Access Control & Access Rights
**Finding**: Quarterly access recertification had lapsed 9 months. Over-provisioned account retained legacy export role.
**Remediation**:
- Conduct an emergency full access review for all accounts with data-export or bulk-query roles.
- Implement an automated access recertification workflow with escalation on non-response.
- Add access review completion as a KPI in the ISMS monitoring dashboard (Clause 9.1).
- Enforce automatic de-provisioning of access not recertified within the review window.

#### A.8.2 — Privileged Access Rights
**Finding**: Data-export role not classified as privileged; no just-in-time controls applied.
**Remediation**:
- Re-classify all roles permitting bulk data access as privileged.
- Implement just-in-time access with dual-approval workflow for bulk export operations.
- Log all privileged role activations to an immutable audit trail.

#### A.5.3 — Segregation of Duties
**Finding**: Customer support and bulk data export capabilities combined in one role.
**Remediation**:
- Redesign role matrix to separate customer support, read-access, and export functions.
- Enforce the separation in the identity provider and application role assignments.
- Add compensating control: alert on any single account performing >500 record reads in a 24-hour window.

#### A.8.12 — Data Leakage Prevention
**Finding**: Volume-only DLP rules evaded by staged incremental exports.
**Remediation**:
- Deploy content-aware DLP capable of detecting PHI field patterns (e.g., ICD codes, insurance ID formats) regardless of transfer volume.
- Implement behavioral DLP: alert when an account's export frequency over a rolling 48-hour period exceeds its historical 90th percentile.
- Add egress controls blocking data transfers to uncategorized or newly registered domains from production environments.

#### A.8.16 — Monitoring Activities
**Finding**: SIEM lacked behavioral correlation rules for incremental exfiltration patterns.
**Remediation**:
- Build SIEM correlation rule: alert when the same account performs >3 data exports within a 24-hour period totaling >10,000 records cumulatively.
- Integrate the threat intelligence feed into SIEM enrichment and automated firewall block list updates.
- Establish a monitoring effectiveness review cadence (quarterly) as a Clause 9.1 measurement activity.

---

### Medium-Term Remediation (30–90 days)

#### A.6.3 — Information Security Awareness, Education and Training
**Finding**: Phishing simulation training not repeated post-onboarding; employee had not been tested in 18 months.
**Remediation**:
- Institute a quarterly phishing simulation program with mandatory remedial training for staff who click simulated phishing links.
- Add annual awareness training completion and phishing simulation pass rate as ISMS KPIs.
- Update the training curriculum to include real-time phishing proxy attack scenarios.
- Retain training completion records as documented evidence per Clause 7.2.

#### A.8.3 — Information Access Restriction
**Finding**: API permitted cross-clinic record access; no record-level authorization.
**Remediation**:
- Implement record-level authorization in the patient records API: every query must be scoped to the requesting account's assigned clinic.
- Add integration tests to the CI/CD pipeline that verify cross-clinic data isolation.
- Include API authorization logic in the annual security testing scope (A.8.29).

#### A.8.9 — Configuration Management
**Finding**: API key rotation policy (90 days) not enforced; keys were 8 months old.
**Remediation**:
- Integrate secrets manager with automated rotation enforcement: keys automatically rotated on schedule; applications must support zero-downtime rotation.
- Add configuration drift detection to the conmon tooling; generate alerts when secrets exceed the rotation threshold.
- Update the Configuration Management Policy with enforced rotation timelines and automated alerting thresholds.

#### A.8.22 — Segregation of Networks
**Finding**: Patient records and insurance claims services shared a network segment.
**Remediation**:
- Implement micro-segmentation: each production service in its own network segment with explicit allow-list rules between services.
- Enforce mutual TLS for all inter-service communication.
- Document the updated network architecture in the system boundary documentation.

#### A.5.24 / A.5.25 — Incident Management Planning and Event Assessment
**Finding**: Incident plan lacked scenario playbooks; triage criteria not documented.
**Remediation**:
- Develop scenario-specific playbooks for: credential compromise, data exfiltration, ransomware, and supply chain compromise.
- Define and document event triage criteria with a severity classification matrix (P1–P4) and maximum time-to-classify SLAs.
- Conduct a tabletop exercise using this breach scenario within 60 days; document results as evidence for A.5.24.

#### A.5.7 — Threat Intelligence
**Finding**: Threat feed subscribed but not operationalized; known-malicious IPs not blocked.
**Remediation**:
- Automate threat intelligence feed ingestion into firewall block lists and SIEM enrichment pipelines.
- Define a threat intelligence handling procedure: how indicators are validated, ingested, and retired.
- Assign a Threat Intelligence Analyst role (even if part-time) with documented responsibilities.

---

### Long-Term Remediation (90+ days)

#### A.5.27 — Learning from Information Security Incidents
**Action**: Conduct a formal post-incident review within 30 days of containment. Document root cause analysis, contributing factors, timeline, and corrective actions. Present findings to management as input to the next management review (Clause 9.3). Update the risk register to reflect new threat scenarios identified.

#### A.5.34 / A.5.31 — Privacy, PII, and Legal/Regulatory Requirements
**Action**: Review breach notification obligations across all applicable jurisdictions (GDPR, HIPAA, state breach notification laws). Document the notification timeline and actions taken. Engage DPO (if applicable) in root cause review. Update data protection impact assessments (DPIAs) for affected data processing activities.

#### A.5.35 — Independent Review of Information Security
**Action**: Commission an independent penetration test targeting the authentication, access control, and API authorization weaknesses identified in this breach. Use findings to update the risk register and SoA. Schedule findings review at the next management review.

#### A.6.5 — Responsibilities After Termination or Change of Employment
**Finding (latent)**: The over-provisioned account had accumulated its excessive rights during a role change — the offboarding/role-change process did not trigger an access review.
**Remediation**: Update the joiner-mover-leaver process to require access review and right-sizing any time an employee changes roles, not only upon departure.

#### Clause 10.2 — Nonconformity and Corrective Action
**ISMS Management Action**: For each control failure above, open a formal nonconformity in the corrective action register. Each nonconformity must document: description, root cause, corrective action, owner, target date, and effectiveness verification method. Present corrective action status at the next management review. Update the risk assessment and SoA to reflect post-remediation residual risk.

---

## Auditor Evidence Requirements Post-Breach

If the organization undergoes a surveillance or recertification audit following this incident, auditors will expect the following evidence:

| Control | Expected Evidence |
|---------|-------------------|
| A.5.24 | Updated incident response plan with scenario playbooks; tabletop exercise report |
| A.5.26 | Incident timeline, containment actions log, communication records |
| A.5.27 | Post-incident review report; corrective action register entries; management review minutes |
| A.5.28 | Forensic evidence collection records; chain-of-custody documentation |
| A.5.17 | Updated authentication policy; MFA enrollment records showing phishing-resistant factor adoption |
| A.5.18 | Access recertification completion records; recertification workflow screenshots |
| A.8.12 | DLP rule configuration; test results showing PHI detection; behavioral alert evidence |
| A.8.15 | Log retention policy; evidence of SIEM pipeline validation; log completeness checks |
| A.8.16 | Updated SIEM correlation rules; threat intelligence integration evidence; monitoring KPIs |
| A.6.3 | Phishing simulation results; training completion records; remedial training evidence |
| Clause 10.2 | Corrective action register; evidence of effectiveness verification for each NC |

---

## Risk Register Entries — New Risks Identified

Add the following entries to the ISMS risk register following this incident:

| Risk ID | Threat | Vulnerability | Existing Controls | Residual Risk (pre-remediation) | Treatment |
|---------|--------|---------------|-------------------|---------------------------------|-----------|
| R-NEW-01 | External attacker using real-time phishing proxy | SMS OTP susceptibility | MFA enforced | High | Migrate to FIDO2 MFA (A.5.17) |
| R-NEW-02 | Privilege abuse via stale over-provisioned roles | No scheduled access recertification | Access control policy | High | Automated quarterly recertification (A.5.18) |
| R-NEW-03 | Incremental data exfiltration below DLP thresholds | Volume-only DLP | DLP tool | High | Content-aware + behavioral DLP (A.8.12) |
| R-NEW-04 | Lateral movement via flat inter-service networking | No micro-segmentation | Network firewall | Medium | Micro-segmentation + mTLS (A.8.22) |
| R-NEW-05 | Threat intelligence not operationalized | Manual feed review | Threat intel subscription | Medium | Automated ingestion into SIEM + block lists (A.5.7) |

---

## Exercise Debrief Questions

Use these for post-exercise discussion or management review:

1. At which phase could the breach have been detected earliest with the controls we have today? After remediation?
2. Were breach notification timelines met? What would need to change for the next incident?
3. Which ISO 27001 control failure had the highest business impact? What does that tell us about risk prioritization?
4. Does our current SoA accurately reflect implementation status for A.8.12, A.8.16, and A.5.17?
5. Which corrective actions require a change to the risk treatment plan vs. a process improvement only?
6. How will we verify the effectiveness of each corrective action before closing the nonconformity?
