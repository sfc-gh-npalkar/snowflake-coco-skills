# Snowflake half — stage, view, procedure, roles

All identifiers use `<PREFIX>` from Step 1.2. Substitute it everywhere. Put `[<PREFIX>]` in every
`COMMENT` so teardown can find everything.

Write the DDL to a local file, show it, and get approval before executing (rule 6).

---

## 1. Warehouse, database, schema

```sql
CREATE WAREHOUSE IF NOT EXISTS <PREFIX>_WH
  WAREHOUSE_SIZE = XSMALL
  AUTO_SUSPEND = 60
  INITIALLY_SUSPENDED = TRUE
  COMMENT = 'Export compute. [<PREFIX>]';

CREATE DATABASE IF NOT EXISTS <PREFIX>_DB COMMENT = 'SFTP export demo. [<PREFIX>]';
CREATE SCHEMA IF NOT EXISTS <PREFIX>_DB.EXPORT COMMENT = 'Export objects. [<PREFIX>]';
```

`AUTO_SUSPEND = 60` matters: the export runs for seconds, and an idle warehouse is the easiest
accidental cost in the build.

---

## 2. The export view

The view is the contract. It is the only object the runtime role can reach, which is what makes
the least-privilege story real rather than decorative.

```sql
CREATE OR REPLACE VIEW <PREFIX>_DB.EXPORT.V_<FEED_NAME> AS
SELECT
  <curated column list>
FROM <their schema>.<their table>
WHERE <any filter the feed needs>;
```

Rules for building it:

- **Name every column explicitly.** `SELECT *` couples the feed to their table's physical shape,
  so an added column silently changes the delivered file.
- **Cast timestamps to `TIMESTAMP_NTZ`** if the format is Parquet — `TIMESTAMP_TZ` and
  `TIMESTAMP_LTZ` cannot be unloaded to Parquet at all.
- **Do not modify their table.** If a cast or rename is needed, it happens here in the view.

Worth telling them: `TIMESTAMP_NTZ` still lands in Parquet metadata as `timestamp[ms, tz=UTC]`.
The values are correct, but a consumer that assumes local time will double-convert. Better said
now than debugged later.

---

## 3. The stage

```sql
CREATE STAGE IF NOT EXISTS <PREFIX>_DB.EXPORT.EXPORT_STAGE
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  COMMENT = 'Parquet/CSV export staging. [<PREFIX>]';
```

`SNOWFLAKE_SSE` is **required**, not a preference. The internal default is client-side
encryption, and a presigned URL against a client-side-encrypted stage downloads **corrupted
bytes with no error at any layer** — the HTTP request succeeds, the byte count looks plausible,
and the file simply will not parse.

`DESCRIBE STAGE` does not expose the encryption type for internal stages, so there is no way to
check after the fact. If a stage might have been created without it, recreate it.

An internal stage is deliberate here. `GET_PRESIGNED_URL` works on internal stages needing only
`READ`, so there is no storage integration, no cloud credentials, and nothing tied to one cloud.
That is what keeps the pattern portable to any SFTP destination.

---

## 4. The export procedure

Owner's rights. This is the least-privilege mechanism: the runtime role gets `USAGE` on this
procedure and nothing else — no `SELECT` on the view, no privileges on the stage.

```sql
CREATE OR REPLACE PROCEDURE <PREFIX>_DB.EXPORT.SP_EXPORT_<FEED_NAME>()
RETURNS VARIANT
LANGUAGE SQL
AS
DECLARE
  v_filename STRING;
  v_rows     NUMBER;
  v_unloaded NUMBER;
  v_bytes    NUMBER;
  v_url      STRING;
  v_sql      STRING;
BEGIN
  SELECT COUNT(*) INTO :v_rows FROM <PREFIX>_DB.EXPORT.V_<FEED_NAME>;

  -- Zero rows must return NULL, not an empty file. COPY INTO with no rows writes
  -- no file at all, so a downstream fetch would retrieve a NoSuchKey error body
  -- and deliver that instead.
  IF (v_rows = 0) THEN
    RETURN NULL;
  END IF;

  v_filename := '<name pattern>_'
             || TO_VARCHAR(CURRENT_TIMESTAMP(), 'YYYYMMDD_HH24MISS')
             || '.<ext>';

  v_sql := 'COPY INTO @<PREFIX>_DB.EXPORT.EXPORT_STAGE/' || :v_filename
        || ' FROM (SELECT * FROM <PREFIX>_DB.EXPORT.V_<FEED_NAME>)'
        || ' FILE_FORMAT = (TYPE = PARQUET)'
        || ' HEADER = TRUE SINGLE = TRUE OVERWRITE = TRUE';

  EXECUTE IMMEDIATE :v_sql;

  SELECT "rows_unloaded", "output_bytes"
    INTO :v_unloaded, :v_bytes
    FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

  -- GET_PRESIGNED_URL will not accept a bound variable for argument 2 -- it fails
  -- with "cannot be null or empty" even when the variable is populated. Build the
  -- call as dynamic SQL with the filename as a literal.
  v_sql := 'SELECT GET_PRESIGNED_URL(@<PREFIX>_DB.EXPORT.EXPORT_STAGE, '''
        || :v_filename || ''', 3600)';

  LET rs RESULTSET := (EXECUTE IMMEDIATE :v_sql);
  LET cur CURSOR FOR rs;
  OPEN cur;
  FETCH cur INTO :v_url;
  CLOSE cur;

  RETURN OBJECT_CONSTRUCT(
    'filename',      :v_filename,
    'row_count',     :v_unloaded,
    'bytes',         :v_bytes,
    'presigned_url', :v_url
  );
END;
```

### COPY INTO options that bite

| Option | Why it matters |
|---|---|
| `HEADER = TRUE` | Without it a Parquet unload writes `col1, col2, ...`. Looks fine until the receiver opens the file. |
| `SINGLE = TRUE` | One file instead of a split set. **Ignores `FILE_EXTENSION`** — the filename and extension must be in the `COPY INTO` path, or the file is literally named `data`. Mutually exclusive with `INCLUDE_QUERY_ID` and `PARTITION BY`. |
| `OVERWRITE = TRUE` | Makes reruns idempotent. Safe here because the filename is timestamped. |

For **CSV**, swap the file format and pin whatever the partner specified:

```sql
FILE_FORMAT = (TYPE = CSV FIELD_DELIMITER = ',' FIELD_OPTIONALLY_ENCLOSED_BY = '"'
               COMPRESSION = NONE NULL_IF = ())
```

`COMPRESSION = NONE` is worth being explicit about — CSV unload gzips by default, which is
usually not what a partner spec means by "CSV".

---

## 5. Roles and grants

Two roles: one owns the objects, one is what Openflow executes as.

```sql
CREATE ROLE IF NOT EXISTS <PREFIX>_OWNER_RL COMMENT = 'Owns export objects. [<PREFIX>]';
CREATE ROLE IF NOT EXISTS <PREFIX>_RUNTIME_RL COMMENT = 'Openflow execute-as role. [<PREFIX>]';

GRANT ROLE <PREFIX>_OWNER_RL   TO USER <user>;
GRANT ROLE <PREFIX>_RUNTIME_RL TO USER <user>;
```

The runtime role gets exactly five grants — and deliberately **no** `SELECT` on the view and
**no** privileges on the stage:

```sql
GRANT USAGE ON DATABASE  <PREFIX>_DB                                TO ROLE <PREFIX>_RUNTIME_RL;
GRANT USAGE ON SCHEMA    <PREFIX>_DB.EXPORT                         TO ROLE <PREFIX>_RUNTIME_RL;
GRANT USAGE ON PROCEDURE <PREFIX>_DB.EXPORT.SP_EXPORT_<FEED_NAME>() TO ROLE <PREFIX>_RUNTIME_RL;
GRANT USAGE, OPERATE ON WAREHOUSE <PREFIX>_WH                       TO ROLE <PREFIX>_RUNTIME_RL;
```

### Demonstrating it — and the trap

The demonstration is: the runtime role can call the procedure but cannot read the underlying
data. Run it as four statements:

```sql
USE SECONDARY ROLES NONE;                      -- without this the demo silently passes
USE ROLE <PREFIX>_RUNTIME_RL;
SELECT COUNT(*) FROM <PREFIX>_DB.EXPORT.V_<FEED_NAME>;   -- must be denied
LIST @<PREFIX>_DB.EXPORT.EXPORT_STAGE;                    -- must be denied
CALL <PREFIX>_DB.EXPORT.SP_EXPORT_<FEED_NAME>();          -- must succeed
```

`USE SECONDARY ROLES NONE` is load-bearing. If the user's secondary roles are `ALL` — common,
and it may include `ACCOUNTADMIN` — the `SELECT` **succeeds** and appears to disprove the
least-privilege claim. Check with `SELECT CURRENT_SECONDARY_ROLES()` if it behaves unexpectedly.

This is an interactive-session artifact only: Openflow's `EXECUTE_AS_ROLE` runs with no secondary
roles, so the real pipeline is genuinely constrained. Worth saying out loud, because a customer
who spots this will otherwise reasonably doubt the whole governance story.

---

## 6. Verify — by downloading, not by trusting

```sql
CALL <PREFIX>_DB.EXPORT.SP_EXPORT_<FEED_NAME>();
LIST @<PREFIX>_DB.EXPORT.EXPORT_STAGE;   -- capture md5 and size
```

Take the `presigned_url` from the return value, `curl` it to a local file, and confirm:

- byte count matches `bytes` from the procedure and `size` from `LIST`
- Parquet: `pyarrow.parquet.read_table()` succeeds; row count, column count, and column
  **names** are right; decimal columns are still `decimal128(p,s)` rather than floats
- CSV: header row, delimiter, and encoding match the spec

`rows_unloaded` proves Snowflake wrote something. It does not prove the bytes are readable —
that is exactly the failure mode a client-side-encrypted stage produces.

**Then take the hostname from the presigned URL.** That is the stage host the Openflow EAI needs
on port 443, and it is account- and region-specific, so it must be read from a real URL rather
than guessed.

---

## Next

**Continue** to `references/openflow-half.md` § Canvas.
