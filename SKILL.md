---
name: specialist-engagement-status
description: "Show specialist engagement update status. Lists specialists classified as Recent, Needed Now, or Needed Soon based on their last comment or activity date. Use when: specialist engagement status, who needs updates, specialist update needed, engagement staleness, stale engagements, which specialists need to update. Triggers: specialist engagement status, engagement status, who needs updates, specialist update needed, stale engagements, update status."
---

# Specialist Engagement Status

**Role:** Sales Operations Analyst

Show which specialists have recent engagement updates and which need to provide updates, based on their last specialist comment or Vivun activity date.

## Data Source

- **Table:** `SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT` (materialized, refreshed every 30 min by task `SALES_DEV.PUBLIC.Refresh_Specialist_Engagement_Status`)
- The underlying view `SALES_DEV.PUBLIC.Specialist_Engagement_Status` joins SE hierarchy, specialist metadata, SFDC deliverable history (specialist comments), and Vivun activity data. The materialized table pre-computes this for fast queries.
- **IMPORTANT:** Always query the `_MAT` table, not the view, to avoid slow execution.

## Connection

- **Connection:** Coco2
- **Read role:** SALES_ENGINEER
- **Write role:** SALES_DEV_RW_RL (for table/task management only)

### Output Columns (always shown)

| Column | Type | Description |
|--------|------|-------------|
| `PREFERRED_NAME` | VARCHAR | Specialist's preferred name |
| `MANAGER_NAME` | VARCHAR | Specialist's manager |
| `SPECIALIST_GROUP` | VARCHAR | Specialist group (e.g., AFE - AI/ML, AFE - DE, Architect, etc.) |
| `LAST_UPDATE` | VARCHAR | Date of most recent comment or activity (YYYY-MM-DD) |
| `DAYS_UNTIL_UPDATE_NEEDED` | NUMBER | Days remaining before update is needed (can be negative = overdue) |
| `UPDATE_NEEDED_STATUS` | VARCHAR | **"Recent"** (>4 days left), **"Needed Soon"** (1–4 days left), **"Needed Now"** (≤0 days left) |

### Detail Columns (shown only if user asks for detail)

| Column | Type | Description |
|--------|------|-------------|
| `SPECIALIST_COMMENTS_14D` | NUMBER | Count of specialist comment updates in last 14 days |
| `SPECIALIST_COMMENTS_7D` | NUMBER | Count of specialist comment updates in last 7 days |
| `ACTIVITIES_14D` | NUMBER | Count of Vivun activities in last 14 days |
| `ACTIVITIES_7D` | NUMBER | Count of Vivun activities in last 7 days |
| `ACTIVE_STATUS` | BOOLEAN | TRUE if any comments or activities in last 14 days |
| `ACTIVE_REASON` | VARCHAR | Why active: "Activity and Comments", "Specialist Comments", "Activity Data", or "No Update" |

### Internal Columns (never shown unless specifically asked)

| Column | Type | Description |
|--------|------|-------------|
| `IS_PEOPLE_MANAGER` | BOOLEAN | Whether this person is a people manager |
| `ORIGINAL_HIRE_DATE` | DATE | Hire date |
| `TENURE` | NUMBER | Tenure in days |
| `HIERARCHY_3` | VARCHAR | Org hierarchy level 3 (e.g., VP) |
| `HIERARCHY_4` | VARCHAR | Org hierarchy level 4 (e.g., Director) |
| `HIERARCHY_5` | VARCHAR | Org hierarchy level 5 (e.g., Manager) |
| `SFDC_ID` | VARCHAR | Salesforce user ID |

### Update Needed Status Logic

The status is derived from the most recent date across specialist comments and Vivun activities:

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | More than 4 days until update needed | Recently active — no action needed |
| **Needed Soon** | 1–4 days until update needed | Getting stale — plan to update soon |
| **Needed Now** | 0 or fewer days remaining (overdue) | Overdue — update immediately |

## Excluded Specialists

The following specialists are excluded from all queries (data issues, inactive, etc.):
- Ajita Sharma

When querying, apply: `AND PREFERRED_NAME NOT IN ('Ajita Sharma')`
To add/remove exclusions, update this list and the corresponding `NOT IN` clause in Step 3.

## Workflow

### Step 0: Infer Filters from User Message

**Before prompting, parse the user's message for implicit filters.** If the intent is clear, skip Steps 1–2 and go directly to Step 3.

| User phrase | Inferred filter |
|-------------|-----------------|
| "who needs updates", "overdue", "stale", "need to update" | Needed Now + Needed Soon |
| "who's current", "who's good", "recent updates" | Recent |
| "how many need updates", "count overdue" | Needed Now + Needed Soon (count only) |
| Mentions a manager name (e.g., "on Zaki's team") | Apply `MANAGER_NAME` filter |
| Mentions a group (e.g., "AFE specialists", "architects") | Apply `SPECIALIST_GROUP` filter |
| Generic: "specialist engagement status", "show me the status" | Proceed to Steps 1–2 |

**Only proceed to Steps 1–2 if the user's request is ambiguous or generic.**

### Step 1: Select Filter (only if Step 0 could not determine)

Ask the user which update status to filter by. Use a single AskUserQuestion call:

1. **Update status filter** (single-select):
   - All (show everything)
   - Recent
   - Needed Soon
   - Needed Now

### Step 2: Optional Scope Filters (only if Step 0 could not determine)

Ask the user if they want to narrow by scope. Use a single AskUserQuestion call:

1. **Specialist group** (single-select):
   - All
   - AFE (all AFE groups)
   - Architect
   - Scale
   - Innovation

2. **Manager or hierarchy filter** (single-select):
   - All
   - Filter by manager name
   - Filter by hierarchy level

If the user selects "Filter by manager name" or "Filter by hierarchy level", ask a follow-up to get the specific value.

### Step 3: Query

Use the following verified base query, applying the selected filters as WHERE clauses:

```sql
SELECT * FROM SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT
WHERE IS_PEOPLE_MANAGER = FALSE
  AND PREFERRED_NAME NOT IN ('Ajita Sharma')
  -- Status filter (omit if "All"):
  -- AND UPDATE_NEEDED_STATUS = '<status>'
  -- Specialist group filter (omit if "All"):
  -- AND SPECIALIST_GROUP = '<group>'
  -- For "AFE (all)" use: AND SPECIALIST_GROUP LIKE 'AFE%'
  -- Manager filter (omit if "All"):
  -- AND MANAGER_NAME = '<manager>'
  -- Hierarchy filter (omit if "All"):
  -- AND (HIERARCHY_3 = '<value>' OR HIERARCHY_4 = '<value>' OR HIERARCHY_5 = '<value>')
ORDER BY
    CASE UPDATE_NEEDED_STATUS
        WHEN 'Needed Now' THEN 1
        WHEN 'Needed Soon' THEN 2
        WHEN 'Recent' THEN 3
    END,
    DAYS_UNTIL_UPDATE_NEEDED ASC,
    PREFERRED_NAME
```

**Verified example query:**
```sql
SELECT * FROM SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT
WHERE UPDATE_NEEDED_STATUS = 'Needed Now'
  AND MANAGER_NAME = 'Zaki Bajwa'
  AND IS_PEOPLE_MANAGER = FALSE
  AND PREFERRED_NAME NOT IN ('Ajita Sharma');
```

### Step 4: Render Terminal Output

**First**, show a summary line with counts only for the statuses included in the current filter:

```
# If filtering Needed Now + Needed Soon:
Found X specialists — Y Needed Now, Z Needed Soon

# If filtering all statuses:
Found X specialists — Y Needed Now, Z Needed Soon, W Recent

# If filtering a single status:
Found X specialists — all Needed Now
```

**Then**, output a single markdown table with all specialists:

```markdown
| Status | Name | Group | Manager | Last Update | Days Until Needed |
|--------|------|-------|---------|-------------|-------------------|
| [!] Needed Now | Jane Doe | AFE - AI/ML | John Smith | 2026-02-15 | -19 |
| [~] Needed Soon | Bob Lee | Architect | Jane Kim | 2026-03-10 | 3 |
| [ok] Recent | Ali Hassan | AFE - DE | John Smith | 2026-03-12 | 6 |
```

**Status formatting:**
- **Needed Now** — prefix with `[!]`
- **Needed Soon** — prefix with `[~]`
- **Recent** — prefix with `[ok]`

**Days Until Needed:** Show negative values as overdue (e.g., "-19" means 19 days overdue).

**After the table**, offer follow-up options:

```
Would you like to:
1. Filter by a different status
2. Group by manager to see team-level view
3. Export as a list of names only
4. Done
```

If the user picks **"Group by manager"**, re-query using:

```sql
SELECT * FROM SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT
WHERE IS_PEOPLE_MANAGER = FALSE
  AND PREFERRED_NAME NOT IN ('Ajita Sharma')
  -- Apply same status/group filters as the original query
ORDER BY MANAGER_NAME,
    CASE UPDATE_NEEDED_STATUS
        WHEN 'Needed Now' THEN 1
        WHEN 'Needed Soon' THEN 2
        WHEN 'Recent' THEN 3
    END,
    DAYS_UNTIL_UPDATE_NEEDED ASC
```

Then display grouped:

```
### Manager Name — X reports (Y need updates)

| Status | Name | Group | Last Update | Days Until Needed |
|--------|------|-------|-------------|-------------------|
...
```

If the user picks **"Export as a list of names only"**, output a simple newline-separated list of specialist names matching the current filter.

## Guardrails

**DO:**
- **Always filter out people managers** (`IS_PEOPLE_MANAGER = FALSE`) in all queries, totals, and lists — managers are excluded by default
- Always sort Needed Now first (most urgent at top)
- Only show these columns in tables: Status, Name, Group, Manager, Last Update, Days Until Needed
- Show activity counts and ACTIVE_REASON only if the user specifically asks for detail
- When grouping by manager, show a count of how many reports need updates

**DO NOT:**
- Generate HTML reports — this skill outputs to terminal only
- Modify the underlying view or its data
- Show SFDC_ID in the output (internal field)
- Show IS_PEOPLE_MANAGER, ORIGINAL_HIRE_DATE, or TENURE unless specifically asked
