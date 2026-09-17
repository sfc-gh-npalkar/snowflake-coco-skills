---
name: sftp-export-deep-dive
description: "Interactive guided build that gets Snowflake data landing on a customer's own SFTP endpoint, orchestrated by Openflow. Interviews the user step by step: source table or view, file format, SFTP host and auth, target path, naming convention, cadence. Proves the SFTP endpoint works before building anything, then builds the Snowflake export (COPY INTO a stage as Parquet or CSV, owner's-rights procedure, least-privilege runtime role), then the Openflow file-mover flow, then verifies the delivered file byte-for-byte. Designed to be run in the customer's own account during a live deep-dive session. Use when someone wants to export Snowflake data to SFTP, deliver files to a partner or vendor SFTP drop, connect Openflow to SFTP, send Parquet or CSV to an SFTP server, or set up an outbound file feed. Triggers: snowflake to sftp, export to sftp, openflow sftp, put files on sftp, sftp drop, partner file feed, outbound file export, deliver parquet to sftp, azure blob sftp, send data to a vendor sftp, file-based integration, sftp deep dive."
---

# Snowflake → Openflow → SFTP: Guided Build

Walk a person through **their table or view → Parquet/CSV on a stage → their SFTP endpoint**,
using Openflow as the mover.

Designed to be handed to a customer and run in *their* account, by them, with an SE watching.
Everything this skill creates is namespaced and disposable, because the customer has to be able
to delete all of it afterwards without thinking.

---

## NON-NEGOTIABLE RULES

Read these before the first tool call.

1. **Their existing data is read-only.** Against any pre-existing table or view, only `SHOW`,
   `DESCRIBE`, `SELECT`, and `INFORMATION_SCHEMA` are permitted. Never `INSERT`, `UPDATE`,
   `DELETE`, `MERGE`, `TRUNCATE`, `ALTER`, or `DROP` an object you did not create in this session.
2. **Namespace everything you create** with a prefix the user picks in Step 1.2, and put
   `[<prefix>]` in every `COMMENT`. This is what makes teardown safe and complete.
3. **An Openflow deployment holds a compute pool that bills continuously from creation** —
   not per run. Say this out loud before creating one, and make sure teardown is scheduled.
   This is the single most expensive thing in the session.
4. **Never print a secret.** SSH private keys and SFTP passwords go from a file or a prompt
   straight into a parameter context or a `SECRET`. Note that
   `ALTER USER ... ADD PROGRAMMATIC ACCESS TOKEN` **returns the token as its result set**, so
   it will land in the transcript — mint it with a short expiry and remove it at teardown.
5. **Prove the SFTP endpoint before building anything on top of it.** Auth, write, and rename
   are the three things that fail, and finding out during a canvas run costs 20 minutes of
   misdiagnosis. Step 3 exists for this reason.
6. **Show DDL and get approval before executing it.** Write it to a local file, show it, ask.
7. **One question at a time.** They are learning. Every question goes through
   `ask_user_question` — never ask in prose. Explain a concept at the moment it becomes
   load-bearing, not before.

---

## The constraint that shapes the whole build

Say this early, because it determines the architecture and it is the most useful thing the
customer will learn today:

> The Openflow runtime image does not bundle a Parquet record writer. `ExecuteSQLRecord` can
> emit CSV, JSON, or Avro — not Parquet. So if the destination needs Parquet, **Snowflake makes
> the file** (`COPY INTO @stage`) and **Openflow only moves it**.

That is not a workaround, it is the supported shape, and it has a real benefit: the conversion
happens in Snowflake's compute where it is cheap and observable, and Openflow stays a dumb,
reliable transport. If they need CSV, Openflow *can* query and write it directly — but the
stage-then-move pattern still gives them a re-drivable artifact, so prefer it either way.

---

## Workflow

```
Step 1  Pre-flight + interview        ← the questions
Step 2  Openflow infrastructure       ← long pole, start it early
Step 3  Prove the SFTP endpoint       ← before anything is built on it
Step 4  Snowflake half                ← stage, view, procedure, roles
Step 5  Openflow half                 ← the canvas
Step 6  Verify byte-for-byte
Step 7  Hand off + teardown
```

Steps 2 and 3 run **concurrently** with the interview tail — the deployment takes ~10 minutes
to reach ACTIVE and there is no reason to sit and watch it.

---

### Step 1: Pre-flight and interview

**Goal:** know everything needed to build, and fail fast on anything that blocks.

First, **load** `references/prereqs.md` and check each item. If something is missing, say which
item and what it unblocks, then ask whether to continue on the parts that are not blocked.

Then interview. Ask these **one at a time** via `ask_user_question`, in this order — it is
ordered so the expensive discoveries come first.

**1.1 — Does this account already have an Openflow deployment?**
Run `SHOW OPENFLOW DEPLOYMENTS` first and lead with the answer rather than making them guess.
- Rows returned, one `ACTIVE` → reuse it. Skip most of Step 2.
- No rows → greenfield. Requires Openflow terms accepted and account-level privileges. This is
  the long pole; confirm they want to spend the ~10 minutes and the compute-pool cost.
- Syntax error on `SHOW OPENFLOW DEPLOYMENTS` → this account predates the gen 2 SQL grammar.
  **Load** the `openflow` skill and follow its surface-detection matrix instead of continuing here.

**1.2 — What prefix should everything created today use?**
Suggest something like `SFTP_DEMO`. Used for the database, roles, stage, deployment, runtime,
EAI, and every `COMMENT`. Confirm they understand this is the teardown handle.

**1.3 — What data are we exporting?**
- A table or view they already have → ask for the fully-qualified name, then `DESCRIBE` it and
  show them the columns. **Do not** export `SELECT *` from a wide table; help them name a
  curated column list, and create a *new* view over it rather than modifying theirs.
- Nothing suitable yet → offer to generate a small realistic sample set. If they take this,
  ask what domain so the data is plausible; generic `foo/bar` data undermines the demo.

**1.4 — What format does the receiving system need?**
This is the teaching moment. Deliver the constraint above, then:
- Parquet → Snowflake converts, Openflow moves. Note the type limits: `TIMESTAMP_TZ` and
  `TIMESTAMP_LTZ` **cannot** be unloaded to Parquet; use `TIMESTAMP_NTZ`.
- CSV → same shape still recommended. Ask about delimiter, header, quoting, and encoding,
  because a partner spec usually pins these.
- "They didn't say" → flag it as a real risk to resolve before production, default to Parquet
  for the demo, and move on.

**1.5 — SFTP endpoint details.**
Host, port, username, target directory. Then auth mode:
- Private key → path to the key file on this machine. Check it is not passphrase-protected
  (`ssh-keygen -y -f <path>` prompts if it is) and `chmod 600` it if needed.
- Password → take it at the prompt, never echo it.

Also ask two things people forget:
- **Does the account have List permission on that directory?** If not, `Create Directory` must
  be `false` and `Disable Directory Listing` must be `true`, or every run fails on a listing it
  is not allowed to do.
- **Is the SFTP server firewalled by source IP?** If yes, this is a blocker to surface now:
  Openflow egresses from Snowflake infrastructure, not from their laptop, and
  `SYSTEM$GET_SNOWFLAKE_PLATFORM_INFO()` returns **VPC IDs**, not IP ranges. Allowlisting a
  laptop IP will make Step 3 pass and Step 5 fail. Options are a broad allowlist, a private
  connectivity path, or accepting it as a known gap for the demo.

**1.6 — Filename convention.**
Partners usually pin this. Get the pattern (prefix, timestamp format, extension) and whether
the same name should be overwritten or every run should be unique.

**1.7 — Cadence, and what happens on zero rows.**
For the session, manual trigger is right — it keeps control in the room. Ask what production
cadence they'd want so the scheduling story is concrete. Then: if the source has no new rows,
should the run produce an empty file or nothing at all? Most partners treat an unexpected empty
file as a failed feed, so "nothing at all" is usually correct — and it must be an explicit guard,
because `COPY INTO` with zero rows writes **no file**, which downstream steps will happily
misinterpret as something to fetch and deliver.

**Output:** a filled-in parameter set. Echo it back as a table for confirmation before building.

---

### Step 2: Openflow infrastructure

**Load** `references/openflow-half.md` § Infrastructure.

Create in this order, then move to Step 3 while it provisions: admin role and grants → infra
database and schema → **deployment** → network rule and EAI → runtime.

The EAI needs two hosts: their SFTP host on its port, and the Snowflake internal stage host on
443. **Derive the stage host, do not guess it** — it is account- and region-specific. Step 4
produces a presigned URL; take the hostname from it.

---

### Step 3: Prove the SFTP endpoint

**Goal:** confirm auth, write, rename, and delete work — from a shell, before Openflow exists.

**Load** `references/openflow-half.md` § Endpoint proof and run it. It writes a small probe
file, reads it back, compares checksums, and cleans up.

Interpret failures carefully — this is where misdiagnosis is most likely:
- `Permission denied` on `ls` but `put` succeeds → the account can write but not list. Normal
  for a locked-down drop. Set the two listing properties from 1.5 and continue.
- Home directory already maps into the target container → then `Remote Path` must be `/`, not
  the container name, or it resolves to `container/container` and silently writes to the wrong
  place. Verify by `put`ting to `/` and confirming where it lands.
- Connection refused or timeout → if the server is IP-firewalled (1.5), their laptop may simply
  not be allowlisted. Say so plainly, note that the first Openflow run is now the first real
  test of connectivity, and continue.

---

### Step 4: Snowflake half

**Load** `references/snowflake-half.md`.

Builds: warehouse → database and schema → export view → internal stage → owner's-rights
procedure → owner and runtime roles with grants.

Two things that are easy to get wrong and hard to notice:
- The stage **must** be `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')`. The internal default is
  client-side encryption, which yields a presigned URL that downloads **corrupted bytes with no
  error**. `DESCRIBE STAGE` does not expose this for internal stages, so the only proof is
  downloading the file.
- Verify by actually downloading and parsing the output, not by trusting `rows_unloaded`.

Then run the procedure once, take the presigned URL, and use its hostname to finish the EAI
from Step 2.

---

### Step 5: Openflow half

**Load** `references/openflow-half.md` § Canvas.

Six processors: call the procedure → unwrap the returned VARIANT → extract filename and URL →
fetch the bytes over HTTPS → set the filename → `PutSFTP`.

Leave the first processor **stopped** and fire it with **Run Once**. That keeps the trigger in
the room, which is what makes the demo legible.

---

### Step 6: Verify byte-for-byte

Not "the flow ran green" — actually prove the file arrived intact:

1. `LIST @<stage>` and note the file's md5 and size.
2. Pull the delivered file back over SFTP.
3. Compare md5 and byte count. They must match exactly.
4. Re-parse it (`pyarrow` for Parquet) and confirm row count, column count, column **names**,
   and that decimal types survived.

Column names matter: without `HEADER = TRUE` a Parquet unload produces `col1, col2, ...`, which
looks fine until the receiving system opens it.

---

### Step 7: Hand off and teardown

Summarise what was built, then present the run-of-show and the teardown script.

**Load** `references/teardown.md`. Offer to run it now or leave it as a script. Be direct that
the deployment's compute pool bills until it is dropped.

If they ask about scheduling the export, mention it can be driven by a Snowflake task or a
`cortex automation`, and offer to wire it up.

---

## Stopping Points

- ✋ After the interview, before creating anything — confirm the parameter table.
- ✋ Before creating the Openflow deployment — cost and provisioning time.
- ✋ Before executing any generated DDL.
- ✋ After Step 3 if the endpoint proof failed — decide whether to continue.
- ✋ Before teardown.

**Resume rule:** on approval, continue to the next step without re-asking.

---

## Troubleshooting

**Load** `references/troubleshooting.md` when something fails. It is keyed by symptom and
covers the failures that cost real time on this build: escaped-VARIANT JSONPath misses, NAR
bundle version mismatches, cluster-node-id requirements on queue APIs, presigned-URL argument
binding, corrupted downloads from client-side-encrypted stages, and secondary roles silently
defeating least-privilege demonstrations.

## Output

- A working Snowflake → Openflow → SFTP pipeline in the customer's account.
- A verified delivered file with a matching checksum.
- A teardown script that removes everything.
