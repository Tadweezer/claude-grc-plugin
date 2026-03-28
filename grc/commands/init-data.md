---
description: "Initialize a local GRC compliance tracker file for a project"
---

# /grc:init-data

Initialize a local GRC compliance tracker file (`grc-tracker.json`) in the current directory to track control implementation status for a project.

## Usage

```
/grc:init-data [framework] [baseline] [system-name?]
```

## Arguments

- **framework**: The compliance framework. Accepts same aliases as `control-lookup`. Defaults to `fedramp` if omitted.
- **baseline**: Baseline/level. For NIST/FedRAMP: `low`, `moderate`, `high`. For CMMC: `level1`, `level2`, `level3`. For SOC 2: `security`, `availability`, `confidentiality`, `privacy`, `processing-integrity` (or comma-separated). Defaults to `moderate`.
- **system-name** (optional): Name of the system or project being tracked. If omitted, uses the current directory name.

## Examples

```
/grc:init-data fedramp moderate "My SaaS Platform"
/grc:init-data nist high
/grc:init-data soc2 security "Customer Data Service"
/grc:init-data cmmc level2 "CUI Processing System"
/grc:init-data fedramp moderate
```

## Behavior

When invoked:

1. **Check for existing tracker** — If `grc-tracker.json` already exists in the current directory, warn the user and ask whether to overwrite, merge (add missing controls), or cancel.

2. **Identify framework and baseline** from arguments. Apply the same alias normalization as `control-lookup`.

3. **For NIST/FedRAMP frameworks**, read the OSCAL family JSON files to enumerate all controls in scope for the specified baseline:
   - For `fedramp`: read each family file from `skills/grc-knowledge/oscal/fedramp-moderate-rev5/{family}.json`
   - For `nist`: read each family file from `skills/grc-knowledge/oscal/nist-800-53-rev5/{family}.json`
   - Enumerate controls by scanning `.controls[]` entries where baseline assignment matches (look for `.props[] | select(.name == "label")` and baseline indicators)
   - Include control enhancements (nested `.controls[]`) that are in the baseline
   - **Use the Read tool only** — do NOT use Bash, Python, or jq to read OSCAL files
   - Read only the families relevant to the framework; for FedRAMP Moderate the standard families are: AC, AT, AU, CA, CM, CP, IA, IR, MA, MP, PE, PL, PS, RA, SA, SC, SI, SR
   - Extract per control: `id`, `title`, `family` (parent group title), `baseline` (Low/Moderate/High indicator from props)

4. **For non-NIST frameworks** (SOC 2, ISO 27001, PCI DSS, HIPAA, CIS, COBIT, CSA CCM, GDPR), read the framework reference markdown file from `skills/grc-knowledge/frameworks/` and enumerate controls/criteria from the structured tables in that file.

5. **Build the tracker structure** and write `grc-tracker.json` to the current directory:

   ```json
   {
     "version": "1.0",
     "created": "<ISO 8601 date>",
     "updated": "<ISO 8601 date>",
     "system": {
       "name": "<system-name>",
       "framework": "<normalized framework name>",
       "baseline": "<baseline>",
       "categorization": null
     },
     "controls": [
       {
         "id": "AC-1",
         "title": "Policy and Procedures",
         "family": "Access Control",
         "status": "not-started",
         "implementation_type": null,
         "notes": "",
         "last_reviewed": null
       }
     ],
     "summary": {
       "total": 0,
       "not_started": 0,
       "in_progress": 0,
       "implemented": 0,
       "inherited": 0,
       "not_applicable": 0
     }
   }
   ```

   **Status values**:
   - `not-started` — Control not yet addressed
   - `in-progress` — Implementation underway
   - `implemented` — Control fully implemented with evidence
   - `inherited` — Control inherited from a provider/leveraged system
   - `not-applicable` — Control does not apply (requires justification in notes)

   **Implementation type values** (set when status is not `not-started`):
   - `system-specific` — Implemented by this system
   - `hybrid` — Partially inherited, partially system-specific
   - `inherited` — Fully inherited from another system/service

6. **Write the file** using the Write tool. Then display a confirmation summary.

7. **If no arguments provided**, ask the user for the framework, baseline, and system name.

## Output Format

```
## GRC Tracker Initialized

**File**: `grc-tracker.json`
**System**: [system-name]
**Framework**: [framework] — [baseline]
**Controls loaded**: [N] controls across [M] families/categories

### Control Summary by Family
| Family | Controls |
|--------|---------|
| Access Control (AC) | 25 |
| Audit and Accountability (AU) | 12 |
| ... | ... |

### Next Steps
1. Run `/grc:list-controls` to review all controls in scope
2. Update `grc-tracker.json` as you implement controls (or use `/grc:gap-analysis` for a guided assessment)
3. Run `/grc:control-lookup [framework] [id]` to look up specific control requirements
4. Run `/grc:evidence-checklist [framework] [controls]` to generate evidence checklists

**Tip**: Commit `grc-tracker.json` to version control to track compliance progress over time.
```

## Notes

- The tracker file is meant to be committed to version control alongside code, giving teams a living compliance posture that evolves with the system.
- Status values are intentionally simple — teams can augment the JSON with additional fields as needed.
- The `summary` block is recomputed each time the file is updated.
- For FedRAMP, `inherited` status is commonly used for controls satisfied by cloud infrastructure providers (IaaS/PaaS).
