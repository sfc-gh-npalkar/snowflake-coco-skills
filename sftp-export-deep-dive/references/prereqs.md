# Prerequisites — what to have ready

This is the pre-read. It is written to be forwarded to whoever will run the session, so it is
plain language rather than agent instructions. Check each item in Step 1 before interviewing.

---

## 1. Snowflake access

- A Snowflake account you can create objects in — a dev or sandbox account is ideal. This
  session creates a database, a warehouse, two roles, a stage, a procedure, and Openflow
  infrastructure.
- A role that can create account-level objects. `ACCOUNTADMIN` is simplest for a session; the
  specific privileges are `CREATE DATABASE`, `CREATE INTEGRATION`, `CREATE COMPUTE POOL`, and
  `CREATE OPENFLOW DEPLOYMENT` on the account.
- Whether **Openflow is already deployed** in this account. If not, someone with the right
  privileges must accept the Openflow terms of service first, and creating a deployment takes
  around ten minutes.

**Note on cost:** an Openflow deployment holds a compute pool that bills continuously from the
moment it is created, not per run. Plan to tear it down after the session, or reuse a deployment
that already exists.

---

## 2. The SFTP endpoint

Have all of these to hand:

| Item | Notes |
|---|---|
| Hostname | |
| Port | Usually 22 |
| Username | |
| Credential | Either a private key file **or** a password |
| Target directory | The path files should land in |

And know the answers to these three, because each one changes how the flow is configured:

- **Can that account list the target directory?** Many partner drops allow write but not list.
  That is fine, but the flow has to be told not to try.
- **Is the server firewalled by source IP?** This matters more than people expect. Openflow
  egresses from Snowflake's infrastructure, not from your laptop, and Snowflake does not publish
  a stable outbound IP range for this — `SYSTEM$GET_SNOWFLAKE_PLATFORM_INFO()` returns VPC
  identifiers, which a typical SFTP firewall cannot use. If the server allowlists source IPs,
  raise it before the session so there is a plan.
- **Is the key passphrase-protected?** A passphrase is supported but adds a step. Check with
  `ssh-keygen -y -f <keyfile>` — if it asks for a passphrase, it has one.

If the key file was just downloaded, tighten its permissions or SSH will refuse to use it:

```bash
chmod 600 <keyfile>
```

---

## 3. The data

Either:

- **An existing table or view** you want to export — have the fully-qualified name ready, and a
  view of which columns actually belong in the feed. Exporting every column of a wide table is
  rarely what the receiving system wants.
- **Or nothing yet** — the session can generate a small realistic sample set instead. Come with
  a domain in mind so the data is plausible rather than `foo`/`bar`.

Two data-shape details worth checking in advance:

- **Timestamp types.** Parquet unload rejects `TIMESTAMP_TZ` and `TIMESTAMP_LTZ`. If the source
  uses either, the export view needs to cast to `TIMESTAMP_NTZ`.
- **Volume.** A single-file export is the simplest thing to demonstrate and to receive. If the
  dataset is large enough that one file is unreasonable, decide up front whether the receiving
  system can handle a multi-file drop.

---

## 4. The receiving system's spec

The most common cause of rework is discovering the file spec after building. Ideally bring:

- Format — Parquet, or CSV
- For CSV: delimiter, header row, quoting, encoding, line endings
- Filename convention — prefix, timestamp format, extension
- Whether each delivery should be a unique filename or overwrite a fixed one
- Expected cadence
- What the receiver should see when there is no new data — usually nothing at all, since an
  unexpected empty file tends to be treated as a broken feed

If the spec is not available, that is workable; the session will use sensible defaults and flag
the open questions.

---

## 5. Local tooling

Only needed on the machine driving the session:

- `sftp` and `ssh-keygen` — present by default on macOS and Linux
- Python 3.11+
- For Parquet verification: `pip install pyarrow`
- For building the Openflow canvas programmatically: `pip install 'nipyapi[cli]'` — the `[cli]`
  extra matters, without it the CLI reports a missing `fire` package

Alternatively the canvas can be built by hand in the Openflow UI while the agent drives the
Snowflake side. Decide which you prefer before the session.
