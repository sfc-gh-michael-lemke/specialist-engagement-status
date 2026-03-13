---
name: specialist-engagement-status
description: "Show specialist engagement update status. Lists specialists with their use cases classified as Recent, Needed Now, or Needed Soon based on how stale the engagement is. Use when: specialist engagement status, who needs updates, specialist update needed, engagement staleness, stale engagements, which specialists need to update. Triggers: specialist engagement status, engagement status, who needs updates, specialist update needed, stale engagements, update status."
---

# Specialist Engagement Status

**Role:** Sales Operations Analyst

Show which specialists have recent engagement updates and which need to provide updates, based on how recently their use cases were modified.

## Data Source

- **Table:** `MDM.MDM_INTERFACES.DIM_USE_CASE`
- **Specialist names:** `USE_CASE_TEAM_NAME_LIST` (ARRAY) — flattened with parallel index into `USE_CASE_TEAM_ROLE_LIST`
- **Specialist roles:** `USE_CASE_TEAM_ROLE_LIST` (ARRAY) — only include specialist roles (see filter below)
- **Staleness proxy:** `LAST_MODIFIED_DATE` compared to `CURRENT_DATE`
- **Value field:** `USE_CASE_EACV`

### Specialist Roles Filter

Only include team members whose role matches one of these specialist roles:

```
SE - Workload FCTO
SE - Enterprise Architect
SE - Security FCTO
SE - Performance FCTO
SE - Solution Innovation Team
SE - Industry CTO
SE - Partner
SE - Activation
SE - Champion
FCTO - Industry Architect
FCTO - Platform Architect
FCTO - Security Architect
Industry Principal
Platform Specialist
Data Cloud Product Principal
```

Exclude non-specialist roles like "Solution Engineer", "Use Case Owner", "Account Hierarchy Visibility", "Secondary Account Executive", etc.

## Update Needed Status Classification

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | `LAST_MODIFIED_DATE` within last 7 days | Engagement was recently updated — no action needed |
| **Needed Soon** | `LAST_MODIFIED_DATE` 8–21 days ago | Getting stale — specialist should plan to update |
| **Needed Now** | `LAST_MODIFIED_DATE` more than 21 days ago | Overdue — specialist should update immediately |

## Workflow

### Step 1: Select Filter

Ask the user which update status to filter by. Use a single AskUserQuestion call:

1. **Update status filter** (single-select):
   - All (show everything)
   - Recent (updated in last 7 days)
   - Needed Soon (8–21 days stale)
   - Needed Now (more than 21 days stale)

### Step 2: Optional Scope Filters

Ask the user if they want to narrow by scope. Use a single AskUserQuestion call with up to 3 questions:

1. **Theater** (single-select):
   - All
   - AMSExpansion
   - AMSAcquisition
   - USMajors
   - EMEA
   - APJ

2. **Minimum EACV** (single-select):
   - No minimum
   - $100K+
   - $500K+
   - $1M+

### Step 3: Query Specialist Engagements

Run the following query, applying the selected filters:

```sql
WITH specialist_engagements AS (
    SELECT
        u.USE_CASE_ID,
        u.USE_CASE_NUMBER,
        u.USE_CASE_NAME,
        u.ACCOUNT_NAME,
        u.USE_CASE_EACV,
        u.USE_CASE_STAGE,
        u.THEATER_NAME,
        u.REGION_NAME,
        u.LAST_MODIFIED_DATE,
        u.SPECIALIST_COMMENTS,
        TRIM(name.value::VARCHAR, '"') AS SPECIALIST_NAME,
        TRIM(role.value::VARCHAR, '"') AS SPECIALIST_ROLE,
        DATEDIFF('day', u.LAST_MODIFIED_DATE, CURRENT_DATE) AS DAYS_SINCE_UPDATE,
        CASE
            WHEN DATEDIFF('day', u.LAST_MODIFIED_DATE, CURRENT_DATE) <= 7 THEN 'Recent'
            WHEN DATEDIFF('day', u.LAST_MODIFIED_DATE, CURRENT_DATE) <= 21 THEN 'Needed Soon'
            ELSE 'Needed Now'
        END AS UPDATE_STATUS
    FROM MDM.MDM_INTERFACES.DIM_USE_CASE u,
         LATERAL FLATTEN(input => u.USE_CASE_TEAM_NAME_LIST) name,
         LATERAL FLATTEN(input => u.USE_CASE_TEAM_ROLE_LIST) role
    WHERE name.index = role.index
      AND u.IS_DEPLOYED = FALSE
      AND u.IS_LOST = FALSE
      AND u.USE_CASE_EACV > 0
      AND u.THEATER_NAME IS NOT NULL
      AND u.THEATER_NAME NOT IN ('AcctsToDelete', '')
      AND role.value::VARCHAR IN (
          'SE - Workload FCTO',
          'SE - Enterprise Architect',
          'SE - Security FCTO',
          'SE - Performance FCTO',
          'SE - Solution Innovation Team',
          'SE - Industry CTO',
          'SE - Partner',
          'SE - Activation',
          'SE - Champion',
          'FCTO - Industry Architect',
          'FCTO - Platform Architect',
          'FCTO - Security Architect',
          'Industry Principal',
          'Platform Specialist',
          'Data Cloud Product Principal'
      )
      -- Theater filter (omit if "All"):
      -- AND u.THEATER_NAME = '<theater>'
      -- EACV filter (omit if "No minimum"):
      -- AND u.USE_CASE_EACV >= <threshold>
)
SELECT *
FROM specialist_engagements
-- Status filter (omit if "All"):
-- WHERE UPDATE_STATUS = '<status>'
ORDER BY
    CASE UPDATE_STATUS
        WHEN 'Needed Now' THEN 1
        WHEN 'Needed Soon' THEN 2
        WHEN 'Recent' THEN 3
    END,
    SPECIALIST_NAME,
    USE_CASE_EACV DESC NULLS LAST
```

### Step 4: Render Terminal Output

**First**, show a summary line:

```
Found X specialist engagements (Y specialists) — Z Needed Now, W Needed Soon, V Recent
```

**Then**, output a markdown table grouped by specialist. For each specialist, show their engagements:

```
### Specialist Name (Role) — X engagements

| Status | Account | Use Case | EACV | Stage | Days Since Update | Theater |
|--------|---------|----------|------|-------|-------------------|---------|
| Needed Now | Acme Corp | HD-012345: Migration | $1.2M | 3 - Technical Validation | 34 days | USMajors |
| Needed Soon | Beta Inc | HD-067890: Analytics | $500K | 2 - Scoping | 15 days | EMEA |
```

**Status formatting:**
- **Needed Now** — prefix with `[!]`
- **Needed Soon** — prefix with `[~]`
- **Recent** — prefix with `[ok]`

**EACV formatting:** Use $K / $M format (e.g., $1.2M, $500K, $45K).

**After the table**, offer follow-up options:

```
Would you like to:
1. Filter by a different status
2. See details for a specific specialist
3. Export as a list of names only
4. Done
```

If the user picks "See details for a specific specialist", show that specialist's full data including `SPECIALIST_COMMENTS` for each use case.

If the user picks "Export as a list of names only", output a simple newline-separated list of specialist names matching the current filter.

## Guardrails

**DO:**
- Always sort Needed Now first (most urgent at top)
- Include EACV so users can prioritize high-value engagements
- Show the specialist's role alongside their name
- Deduplicate: if a specialist appears on multiple use cases, show all use cases under that specialist

**DO NOT:**
- Include deployed use cases (IS_DEPLOYED = TRUE)
- Include lost use cases (IS_LOST = FALSE)
- Include non-specialist roles (Solution Engineer, Use Case Owner, etc.)
- Include use cases with zero or null EACV
- Generate HTML reports — this skill outputs to terminal only
