---
name: specialist-engagement-status
description: "Show specialist engagement update status. Lists specialists classified as Recent, Needed Now, or Needed Soon based on their last comment or activity date. Use when: specialist engagement status, who needs updates, specialist update needed, engagement staleness, stale engagements, which specialists need to update. Triggers: specialist engagement status, engagement status, who needs updates, specialist update needed, stale engagements, update status."
---

# Specialist Engagement Status

**Role:** Sales Operations Analyst

Show which specialists have recent engagement updates and which need to provide updates, based on their last specialist comment or Vivun activity date.

## Data Source

- **View:** `SALES_DEV.PUBLIC.Specialist_Engagement_Status`
- This view joins SE hierarchy, specialist metadata, SFDC deliverable history (specialist comments), and Vivun activity data to produce a per-specialist engagement status.

### Columns

| Column | Type | Description |
|--------|------|-------------|
| `PREFERRED_NAME` | VARCHAR | Specialist's preferred name |
| `MANAGER_NAME` | VARCHAR | Specialist's manager |
| `IS_PEOPLE_MANAGER` | BOOLEAN | Whether this person is a people manager |
| `ORIGINAL_HIRE_DATE` | DATE | Hire date |
| `TENURE` | NUMBER | Tenure in days |
| `HIERARCHY_3` | VARCHAR | Org hierarchy level 3 (e.g., VP) |
| `HIERARCHY_4` | VARCHAR | Org hierarchy level 4 (e.g., Director) |
| `HIERARCHY_5` | VARCHAR | Org hierarchy level 5 (e.g., Manager) |
| `SFDC_ID` | VARCHAR | Salesforce user ID |
| `SPECIALIST_GROUP` | VARCHAR | Specialist group (e.g., AFE - AI/ML, AFE - DE, Architect, etc.) |
| `SPECIALIST_COMMENTS_14D` | NUMBER | Count of specialist comment updates in last 14 days |
| `SPECIALIST_COMMENTS_7D` | NUMBER | Count of specialist comment updates in last 7 days |
| `ACTIVITIES_14D` | NUMBER | Count of Vivun activities in last 14 days |
| `ACTIVITIES_7D` | NUMBER | Count of Vivun activities in last 7 days |
| `ACTIVE_STATUS` | BOOLEAN | TRUE if any comments or activities in last 14 days |
| `ACTIVE_REASON` | VARCHAR | Why active: "Activity and Comments", "Specialist Comments", "Activity Data", or "No Update" |
| `LAST_UPDATE` | VARCHAR | Date of most recent comment or activity (YYYY-MM-DD) |
| `DAYS_UNTIL_UPDATE_NEEDED` | NUMBER | Days remaining before update is needed (can be negative = overdue) |
| `UPDATE_NEEDED_STATUS` | VARCHAR | **"Recent"** (>4 days left), **"Needed Soon"** (1–4 days left), **"Needed Now"** (≤0 days left) |

### Update Needed Status Logic

The status is derived from the most recent date across specialist comments and Vivun activities:

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | More than 4 days until update needed | Recently active — no action needed |
| **Needed Soon** | 1–4 days until update needed | Getting stale — plan to update soon |
| **Needed Now** | 0 or fewer days remaining (overdue) | Overdue — update immediately |

## Workflow

### Step 1: Select Filter

Ask the user which update status to filter by. Use a single AskUserQuestion call:

1. **Update status filter** (single-select):
   - All (show everything)
   - Recent
   - Needed Soon
   - Needed Now

### Step 2: Optional Scope Filters

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

Run the following query, applying the selected filters:

```sql
SELECT
    PREFERRED_NAME,
    MANAGER_NAME,
    SPECIALIST_GROUP,
    HIERARCHY_4,
    HIERARCHY_5,
    SPECIALIST_COMMENTS_7D,
    SPECIALIST_COMMENTS_14D,
    ACTIVITIES_7D,
    ACTIVITIES_14D,
    ACTIVE_STATUS,
    ACTIVE_REASON,
    LAST_UPDATE,
    DAYS_UNTIL_UPDATE_NEEDED,
    UPDATE_NEEDED_STATUS
FROM SALES_DEV.PUBLIC.Specialist_Engagement_Status
WHERE 1=1
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

### Step 4: Render Terminal Output

**First**, show a summary line:

```
Found X specialists — Z Needed Now, W Needed Soon, V Recent
```

**Then**, output a single markdown table with all specialists:

```markdown
| Status | Name | Group | Manager | Last Update | Days Until Needed | Comments (7d/14d) | Activities (7d/14d) | Reason |
|--------|------|-------|---------|-------------|-------------------|--------------------|--------------------|--------|
| [!] Needed Now | Jane Doe | AFE - AI/ML | John Smith | 2026-02-15 | -19 | 0/0 | 0/0 | No Update |
| [~] Needed Soon | Bob Lee | Architect | Jane Kim | 2026-03-10 | 3 | 1/2 | 0/1 | Specialist Comments |
| [ok] Recent | Ali Hassan | AFE - DE | John Smith | 2026-03-12 | 6 | 2/4 | 3/5 | Activity and Comments |
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

If the user picks **"Group by manager"**, re-query and display grouped:

```
### Manager Name — X reports (Y need updates)

| Status | Name | Group | Last Update | Days Until Needed | Comments (7d/14d) | Activities (7d/14d) |
|--------|------|-------|-------------|-------------------|--------------------|---------------------|
...
```

If the user picks **"Export as a list of names only"**, output a simple newline-separated list of specialist names matching the current filter.

## Guardrails

**DO:**
- Always sort Needed Now first (most urgent at top)
- Show activity counts so managers can see engagement volume
- Include the ACTIVE_REASON to explain what type of activity was recorded
- When grouping by manager, show a count of how many reports need updates

**DO NOT:**
- Generate HTML reports — this skill outputs to terminal only
- Modify the underlying view or its data
- Show SFDC_ID in the output (internal field)
- Show IS_PEOPLE_MANAGER, ORIGINAL_HIRE_DATE, or TENURE unless specifically asked
