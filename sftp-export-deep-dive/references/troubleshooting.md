# Troubleshooting — keyed by symptom

Every entry here is a failure that has actually happened on this build. Match the symptom, do not
guess.

---

## Snowflake side

### Downloaded file is the right size but will not parse

Parquet reader reports a bad magic number; CSV looks like binary. No error at any layer — the
`COPY INTO` succeeded, `rows_unloaded` was correct, the HTTP fetch returned 200 with a plausible
byte count.

**Cause:** the stage uses client-side encryption (the internal default). Presigned URLs return the
still-encrypted bytes.

**Fix:** recreate the stage with `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` and re-export.
`DESCRIBE STAGE` does not expose this for internal stages, so if there is any doubt, recreate.

### `Argument 2 to function 'GET_PRESIGNED_URL' cannot be null or empty`

Raised even though the variable holding the filename is populated — and it fires *after* the
`COPY INTO` has already succeeded, which makes it look like a stage problem.

**Cause:** `GET_PRESIGNED_URL` will not accept a bound variable for the relative-path argument.

**Fix:** build the call as dynamic SQL with the filename interpolated as a literal, then capture
the value with `RESULTSET` + `CURSOR` + `FETCH`. See `snowflake-half.md` § 4.

### Exported file is named `data` with no extension

**Cause:** `SINGLE = TRUE` ignores `FILE_EXTENSION`.

**Fix:** put the filename and extension in the `COPY INTO` path itself.

### Parquet columns are `col1`, `col2`, ...

**Cause:** `HEADER = TRUE` missing.

### `COPY INTO` fails on a timestamp column

Parquet unload rejects `TIMESTAMP_TZ` and `TIMESTAMP_LTZ`.

**Fix:** cast to `TIMESTAMP_NTZ` in the export view. Mention that NTZ still lands as
`timestamp[ms, tz=UTC]` in Parquet metadata, so consumers should not double-convert.

### Least-privilege demo fails to fail — restricted role can read the table

`SELECT` against the view succeeds under the runtime role when it should be denied.

**Cause:** secondary roles. `CURRENT_SECONDARY_ROLES()` is `ALL` and may include `ACCOUNTADMIN`.

**Fix:** `USE SECONDARY ROLES NONE` before `USE ROLE`. This is an interactive-session artifact
only — `EXECUTE_AS_ROLE` in Openflow has no secondary roles.

### `Actual statement count N did not match the desired statement count 1`

**Cause:** several statements submitted as one call.

**Fix:** run them as sequential single statements in the same session.

### Downstream delivers a small XML file instead of data

**Cause:** the source had zero rows, so `COPY INTO` wrote **no file**, and the fetch retrieved a
`NoSuchKey` error body — which the flow then dutifully uploaded.

**Fix:** the zero-row guard returning `NULL` in the procedure. Not optional.

### `Insufficient privileges to operate on schema` as ACCOUNTADMIN

**Cause:** the schema is owned by another role. Ownership beats `ACCOUNTADMIN` for `CREATE` in a
schema.

**Fix:** `USE ROLE <owning role>`.

### `Cannot perform DESCRIBE. This session does not have a current database`

**Fix:** `USE SCHEMA <db>.<schema>` first, or fully qualify.

### `CREATE ROLE` blocked with "Restricted session scope ... does not include CREATE ROLE"

**Cause:** a Restricted Session Scope is bound to this chat session — often a read-only scope.

**Fix:** it is a per-session binding that **survives an application restart**, and removal is a
toggle in the UI (`+` menu → `Restrict this session` → re-select the already-checked entry). Never
attempt `ALTER SESSION` to clear it. Diagnose with:

```sql
SELECT SYS_CONTEXT('SNOWFLAKE$SESSION', 'ACTIVE_RESTRICTED_SESSION_SCOPES');
```

---

## Openflow side

### Component creation returns an empty response body

No error, no JSON, nothing to parse.

**Cause:** wrong NAR bundle version. The bundle version is **not** the NiFi version — a runtime on
NiFi `2026.9.15.2` may carry bundles at `2026.9.15.21`.

**Fix:** read versions from `/flow/processor-types` and `/flow/controller-service-types`.

### JSONPath extracts nothing from a `CALL` result

Attributes come out empty; `Path Not Found Behavior = warn` logs misses.

**Cause:** the `VARIANT` arrives as an **escaped JSON string**, not nested JSON, so
`$[0].PROC_NAME.field` cannot resolve.

**Fix:** two `EvaluateJsonPath` steps — unwrap to content, then extract to attributes. Inspect
real flowfile content rather than reasoning about the shape.

### `HTTP 414 URI Too Long` updating a process group via nipyapi

**Fix:** use curl for the process-group PUT. `nipyapi` remains the better tool for parameter
contexts and assets.

### `CLI requires the 'fire' package`

**Fix:** `pip install 'nipyapi[cli]'`.

### `The id of the node in the cluster is required`

On flowfile-queue listing or content endpoints.

**Fix:** append `?clusterNodeId=<id>` from `GET /controller/cluster`.

### Connection delete appears to succeed but the edge remains

curl reports HTTP `000` or no body, and the connection is still listed — leaving a duplicate edge
that clones flowfiles down two paths.

**Fix:** `DELETE /connections/{id}?version=N&clientId=X` — both parameters are needed. Then list
connections and confirm the chain is linear.

### `UnknownHostException` or connection timeout from a processor

**Cause:** the host is not in the EAI's network rule, or the EAI is not attached to the runtime.

**Fix:** confirm both the SFTP host and the **stage** host are in the rule, that the integration
is `ENABLED`, that the runtime lists it in `external_access_integrations`, and that the runtime
role has `USAGE` on it. The stage host is account- and region-specific — read it from a live
presigned URL.

### Processor stays INVALID: "Relationship X is invalid because it is not connected"

**Fix:** connect it or auto-terminate it. Note that auto-terminating `failure` discards failed
deliveries silently.

### `PutSFTP` fails on the target directory

Depending on the message:
- cannot create directory → `Create Directory = false`
- listing denied → `Disable Directory Listing = true`
- path not found, but a manual `put` to `/` works → `Remote Path` should be `/`; the home
  directory already maps into the target, so naming it again resolves to `dir/dir`
- rename failed → `Dot Rename = false`, and note delivery is no longer atomic

### SSH refuses the private key

`WARNING: UNPROTECTED PRIVATE KEY FILE!`

**Fix:** `chmod 600 <keyfile>`.

### 401/403 against the NiFi API

**Cause:** the programmatic access token expired, or its `ROLE_RESTRICTION` does not match the
role that owns the runtime.

**Fix:** mint a new token restricted to the Openflow admin role and rewrite `profiles.yml`.

### `Oauth code flow requirement 'client_id' is empty` from the Snowflake CLI

The CLI cannot authenticate on an OAuth-configured connection.

**Fix:** not a blocker. Run SQL through the agent's SQL tool; the CLI is only needed if you want
`snow sql` specifically.
