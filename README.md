# Specialist Engagement Status

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that shows which specialists have recent engagement updates and which are overdue, based on their last specialist comment or Vivun activity date. Designed for SE managers and ops leads who need to ensure specialists are actively updating their engagements.

## What It Does

When triggered, the skill:

1. Prompts for an update status filter: **Recent**, **Needed Soon**, **Needed Now**, or All
2. Optionally narrows by specialist group, manager, or hierarchy
3. Queries the `SALES_DEV.PUBLIC.Specialist_Engagement_Status` view
4. Outputs a terminal markdown table sorted by urgency (Needed Now first)

## Update Status Classification

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | >4 days until update needed | Recently active — no action needed |
| **Needed Soon** | 1–4 days until update needed | Getting stale — plan to update |
| **Needed Now** | ≤0 days remaining (overdue) | Update immediately |

## Data Source

**View:** `SALES_DEV.PUBLIC.Specialist_Engagement_Status`

This view joins:
- `SALES.SE_REPORTING.SE_HIERARCHY` — specialist names, managers, org hierarchy
- `SIGMA_WRITEBACK.SALES.DIM_SE_SPECIALIST_METADATA` — specialist group
- `FIVETRAN.SALESFORCE.VH_DELIVERABLE_HISTORY` — specialist comment timestamps
- `SALES.SE_REPORTING.DIM_SE_ACTIVITY` — Vivun activity timestamps

## Install

### Prerequisites

- [Cortex Code CLI](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) installed and authenticated
- A Snowflake connection with read access to `SALES_DEV.PUBLIC.Specialist_Engagement_Status`

### Setup

```bash
mkdir -p ~/.cortex/skills

git clone https://github.com/sfc-gh-michael-lemke/specialist-engagement-status.git \
  ~/.cortex/skills/specialist-engagement-status
```

Cortex Code automatically detects skills in `~/.cortex/skills/`.

## Sample Prompts

```
specialist engagement status
```

```
who needs updates
```

```
show me stale specialist engagements
```

```
specialist update needed — Needed Now only
```

## File Structure

```
specialist-engagement-status/
├── SKILL.md     # Skill definition (workflow, SQL, output format)
├── README.md    # This file
├── LICENSE
└── .gitignore
```

## License

MIT — see [LICENSE](LICENSE).
