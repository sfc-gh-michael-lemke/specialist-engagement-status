# Specialist Engagement Status

A [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) skill that shows which specialists have recent engagement updates and which are overdue, based on use case modification dates in Snowflake. Designed for SE managers and ops leads who need to ensure specialists are actively updating their engagements.

## What It Does

When triggered, the skill:

1. Prompts for an update status filter: **Recent**, **Needed Soon**, **Needed Now**, or All
2. Optionally narrows by theater and minimum EACV
3. Queries `MDM.MDM_INTERFACES.DIM_USE_CASE`, flattening the use case team arrays to extract specialists
4. Classifies each engagement by staleness and outputs a terminal markdown table grouped by specialist

## Update Status Classification

| Status | Condition | Meaning |
|--------|-----------|---------|
| **Recent** | Updated within last 7 days | No action needed |
| **Needed Soon** | 8–21 days since last update | Plan to update |
| **Needed Now** | 21+ days since last update | Update immediately |

## Install

### Prerequisites

- [Cortex Code CLI](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) installed and authenticated
- A Snowflake connection with read access to `MDM.MDM_INTERFACES.DIM_USE_CASE`

### Setup

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.cortex/skills

# Clone
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
specialist update needed for USMajors
```

```
engagement status for deals over $1M
```

## Data Source

| Field | Column | Type |
|-------|--------|------|
| Specialist names | `USE_CASE_TEAM_NAME_LIST` | ARRAY |
| Specialist roles | `USE_CASE_TEAM_ROLE_LIST` | ARRAY |
| Last update | `LAST_MODIFIED_DATE` | DATE |
| EACV | `USE_CASE_EACV` | FLOAT |
| Stage | `USE_CASE_STAGE` | VARCHAR |
| Theater | `THEATER_NAME` | VARCHAR |
| Comments | `SPECIALIST_COMMENTS` | VARCHAR |

### Specialist Roles Included

SE - Workload FCTO, SE - Enterprise Architect, SE - Security FCTO, SE - Performance FCTO, SE - Solution Innovation Team, SE - Industry CTO, SE - Partner, SE - Activation, SE - Champion, FCTO - Industry Architect, FCTO - Platform Architect, FCTO - Security Architect, Industry Principal, Platform Specialist, Data Cloud Product Principal.

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
