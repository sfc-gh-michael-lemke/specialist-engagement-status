# Specialist Engagement Status

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that shows which specialists have recent engagement updates and which are overdue, based on their last specialist comment or Vivun activity date. Designed for SE managers and ops leads who need to ensure specialists are actively updating their engagements.

## What It Does

When triggered, the skill:

1. Infers filters from your natural language prompt (e.g., "who needs updates" automatically filters to Needed Now + Needed Soon)
2. Only prompts for filters if your request is ambiguous or generic
3. Queries the `SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT` materialized table (refreshed every 30 min)
4. Outputs a terminal markdown table sorted by urgency (Needed Now first)

People managers are automatically excluded from all counts and lists.

## Update Status Classification

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | >4 days until update needed | Recently active — no action needed |
| **Needed Soon** | 1–4 days until update needed | Getting stale — plan to update |
| **Needed Now** | ≤0 days remaining (overdue) | Update immediately |

## Data Source

**Materialized Table:** `SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT`

Refreshed every 30 minutes by task `SALES_DEV.PUBLIC.Refresh_Specialist_Engagement_Status`.

The underlying view `SALES_DEV.PUBLIC.Specialist_Engagement_Status` joins:
- `SALES.SE_REPORTING.SE_HIERARCHY` — specialist names, managers, org hierarchy
- `SIGMA_WRITEBACK.SALES.DIM_SE_SPECIALIST_METADATA` — specialist group
- `FIVETRAN.SALESFORCE.VH_DELIVERABLE_HISTORY` — specialist comment timestamps
- `SALES.SE_REPORTING.DIM_SE_ACTIVITY` — Vivun activity timestamps

## Connection

- **Connection:** Coco2
- **Read role:** SALES_ENGINEER
- **Write role:** SALES_DEV_RW_RL (for table/task management only)

## Install

### Prerequisites

- [Cortex Code CLI](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) installed and authenticated
- A Snowflake connection with read access to `SALES_DEV.PUBLIC.Specialist_Engagement_Status_MAT`

### Setup

```bash
mkdir -p ~/.cortex/skills

git clone https://github.com/sfc-gh-michael-lemke/specialist-engagement-status.git \
  ~/.cortex/skills/specialist-engagement-status
```

Cortex Code automatically detects skills in `~/.cortex/skills/`.

## Sample Prompts

```
who needs updates
```

```
how many specialists need to write an update this week
```

```
show me overdue specialists on Zaki's team
```

```
show me stale AFE engagements
```

```
specialist engagement status
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
