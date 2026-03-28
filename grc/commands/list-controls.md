---
description: "List all controls in a framework and baseline, optionally filtered by family"
---

# /grc:list-controls

List all controls in a compliance framework and baseline, with IDs, titles, families, and baseline assignments. Optionally filter by control family or category.

## Usage

```
/grc:list-controls [framework] [baseline?] [family?]
```

## Arguments

- **framework**: The compliance framework. Accepts same aliases as `control-lookup`. Required.
- **baseline** (optional): Baseline or level to filter controls. For NIST/FedRAMP: `low`, `moderate`, `high` (defaults to `moderate`). For CMMC: `level1`, `level2`, `level3`. For SOC 2: `security`, `availability`, `confidentiality`, `privacy`, `processing-integrity`. Omit to show all.
- **family** (optional): A control family ID or name to filter results (e.g., `ac`, `ia`, `au` for NIST; `CC6` for SOC 2; `A.8` for ISO 27001). Omit to show all families.

## Examples

```
/grc:list-controls fedramp moderate
/grc:list-controls fedramp moderate ac
/grc:list-controls nist high ia
/grc:list-controls nist moderate au
/grc:list-controls soc2
/grc:list-controls soc2 security CC6
/grc:list-controls iso27001
/grc:list-controls pci
/grc:list-controls cmmc level2
/grc:list-controls cis ig2
/grc:list-controls hipaa
```

## Behavior

When invoked:

1. **Identify framework, baseline, and optional family filter** from arguments.

2. **For NIST or FedRAMP**, read the OSCAL family JSON files to enumerate controls:
   - **Use the Read tool only** — do NOT use Bash, Python, or jq to read OSCAL files
   - If a `family` argument is given, read only that family file: `skills/grc-knowledge/oscal/{catalog}/{family}.json`
   - If no family filter, read all family files for the framework one at a time:
     - For `fedramp`: AC, AT, AU, CA, CM, CP, IA, IR, MA, MP, PE, PL, PS, RA, SA, SC, SI, SR
     - For `nist`: AC, AT, AU, CA, CM, CP, IA, IR, MA, MP, PE, PL, PM, PS, PT, RA, SA, SC, SI, SR
   - For each control in `.controls[]`, extract:
     - `id` (uppercase display, e.g., `AC-2`)
     - `title`
     - Parent family title (from the group)
     - Baseline assignment: look in `.controls[].props[]` for a prop with `name == "label"` and a baseline indicator — FedRAMP Moderate controls appear in the `fedramp-moderate-rev5/` files by definition; for NIST, controls are labeled Low/Moderate/High in `800-53B` props
     - Enhancement indicator: if the control is nested under a parent (`.controls[].controls[]`), it's an enhancement — display with full ID like `AC-2(1)`
   - **Baseline filtering logic**:
     - For `fedramp moderate`: all controls present in `fedramp-moderate-rev5/` files are in scope (that catalog IS the Moderate baseline)
     - For `nist low/moderate/high`: the NIST OSCAL props include baseline labels; scan for props where `name` is `"label"` in the 800-53B sense or look for included controls per SP 800-53B tables. If baseline props are not directly determinable from the file, list all controls and note the baseline filter is informational.
   - Include enhancements (nested controls) in the listing with their enhancement number.

3. **For all other frameworks**, read the appropriate framework markdown file from `skills/grc-knowledge/frameworks/` and extract the control/criteria list from the structured tables in that file. Apply any family/category filter to the table rows.

4. **Check for a local `grc-tracker.json`** in the current directory. If found, join the tracker status against each control and add a `Status` column to the output. This allows teams to see implementation progress inline.

5. **Format the output** as a structured table organized by family/category. If the total control count exceeds 50, paginate by family with a summary table first.

6. **If no arguments provided**, ask the user which framework and optional baseline/family to list.

## Output Format

### With tracker file present:
```
## [Framework] [Baseline] Controls — [N] controls

> Tracker: `grc-tracker.json` found — showing implementation status.

### [Family Name] ([Family ID]) — [N] controls

| ID | Title | Baseline | Status |
|----|-------|----------|--------|
| AC-1 | Policy and Procedures | Low/Mod/High | not-started |
| AC-2 | Account Management | Low/Mod/High | implemented |
| AC-2(1) | Automated System Account Management | Mod/High | in-progress |
| ... | ... | ... | ... |

### [Next Family] ...

---

## Summary

| Status | Count | % |
|--------|-------|---|
| Implemented | 45 | 15% |
| In Progress | 12 | 4% |
| Not Started | 235 | 78% |
| Inherited | 10 | 3% |
| N/A | 2 | <1% |
| **Total** | **304** | |
```

### Without tracker file:
```
## [Framework] [Baseline] Controls — [N] controls

### [Family Name] ([Family ID]) — [N] controls

| ID | Title | Baseline |
|----|-------|----------|
| AC-1 | Policy and Procedures | Low/Mod/High |
| AC-2 | Account Management | Low/Mod/High |
| AC-2(1) | Automated System Account Management | Mod/High |
| ... | ... | ... |

### [Next Family] ...

---

## Summary

| Family | Controls |
|--------|---------|
| Access Control (AC) | 25 |
| Audit and Accountability (AU) | 12 |
| ... | ... |
| **Total** | **304** |

---

**Tip**: Run `/grc:init-data fedramp moderate` to create a local tracker and see implementation status here.
```

## Notes

- For FedRAMP Moderate, the standard baseline contains approximately 304 controls and enhancements.
- NIST Low contains approximately 150 controls, Moderate ~304, High ~392.
- Enhancements are listed directly after their parent control, indented with their enhancement number in parentheses.
- When a `grc-tracker.json` is present, the status column reflects the tracker file — it does not evaluate actual implementation.
- Use `/grc:control-lookup [framework] [id]` to dive into full detail for any control.
- Use `/grc:evidence-checklist [framework] [family]` to generate an evidence checklist for a full family.
