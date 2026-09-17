# Snowflake CoCo Skills

Skills for [Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) that
walk you through building something real in your own Snowflake account — interview first, then
build, then verify.

## Contents

| Skill | Description |
|-------|-------------|
| [sftp-export-deep-dive](sftp-export-deep-dive/) | Deliver Snowflake data to an SFTP endpoint as Parquet or CSV, orchestrated by Openflow — export view, SSE-encrypted internal stage, owner's-rights procedure, least-privilege runtime role, and a six-processor file-mover flow, verified byte-for-byte against what Snowflake produced |

## Install

**[⬇ Download all skills (.zip)](https://github.com/sfc-gh-npalkar/snowflake-coco-skills/archive/refs/heads/main.zip)**

1. Download and unzip
2. In Cortex Code, click **+** → **Skills** → **Add local skill**
3. Select the skill's folder — the one containing `SKILL.md`, e.g. `sftp-export-deep-dive/`

The skill then triggers automatically when you describe a matching task, or explicitly via
`/<skill-name>`.

Prefer git? `git clone https://github.com/sfc-gh-npalkar/snowflake-coco-skills.git` — then
`git pull` to pick up fixes.

## Before you run one

Each skill folder has its own `references/prereqs.md` listing what to have ready — access,
credentials, endpoints, local tooling. Read it before a working session; several of the items
take time to obtain, and one or two are the kind of thing that blocks a build halfway through.

## About

These are practical starting points, not finished products. Each one encodes the gotchas found
while building the thing for real, which is most of their value — the failure modes are
documented alongside the happy path.

For questions, reach out to your Snowflake account team.
