# Openflow half — infrastructure, endpoint proof, canvas

Three sections, loaded at different steps:
- **§ Infrastructure** — Step 2
- **§ Endpoint proof** — Step 3
- **§ Canvas** — Step 5

Substitute `<PREFIX>` throughout.

---

## § Infrastructure

Create in this order. The deployment is the long pole (~10 minutes), so start it and then go do
Step 3 rather than waiting.

### Admin role

```sql
USE ROLE ACCOUNTADMIN;
CREATE ROLE IF NOT EXISTS <PREFIX>_OF_ADMIN_RL COMMENT = 'Owns Openflow infra. [<PREFIX>]';
GRANT ROLE <PREFIX>_OF_ADMIN_RL TO USER <user>;
GRANT CREATE DATABASE, CREATE INTEGRATION ON ACCOUNT TO ROLE <PREFIX>_OF_ADMIN_RL;
GRANT CREATE COMPUTE POOL ON ACCOUNT       TO ROLE <PREFIX>_OF_ADMIN_RL;
GRANT CREATE OPENFLOW DEPLOYMENT ON ACCOUNT TO ROLE <PREFIX>_OF_ADMIN_RL;
```

### Infra database

```sql
USE ROLE <PREFIX>_OF_ADMIN_RL;
CREATE DATABASE IF NOT EXISTS <PREFIX>_OF COMMENT = 'Openflow infra. [<PREFIX>]';
CREATE SCHEMA IF NOT EXISTS <PREFIX>_OF.RUNTIMES COMMENT = 'Openflow control. [<PREFIX>]';
GRANT CREATE OPENFLOW RUNTIME ON SCHEMA <PREFIX>_OF.RUNTIMES TO ROLE <PREFIX>_OF_ADMIN_RL;
GRANT USAGE ON DATABASE <PREFIX>_OF          TO ROLE <PREFIX>_RUNTIME_RL;
GRANT USAGE ON SCHEMA   <PREFIX>_OF.RUNTIMES TO ROLE <PREFIX>_RUNTIME_RL;
```

Note that `CREATE OPENFLOW RUNTIME` is granted on the **schema**, and that even `ACCOUNTADMIN`
cannot create objects in a schema owned by another role — switch to the owning role rather than
escalating.

### Deployment

Confirm cost and time with the user before running this (rule 3).

```sql
CREATE OPENFLOW DEPLOYMENT <PREFIX>_DEPLOYMENT
  DEPLOYMENT_TYPE = SNOWFLAKE
  DISPLAY_NAME = '<PREFIX>_DEPLOYMENT'
  COMMENT = 'Openflow deployment. [<PREFIX>]';

SELECT SYSTEM$WAIT_FOR_STABLE_OPENFLOW_DEPLOYMENTS(900, '<PREFIX>_DEPLOYMENT');
SHOW OPENFLOW DEPLOYMENTS;   -- expect status ACTIVE
```

### Network rule and EAI

The runtime has no outbound access by default. It needs two hosts: the SFTP endpoint, and the
Snowflake internal stage host that serves presigned URLs.

Get the stage host from a real presigned URL (Step 4) — it is account- and region-specific and
must not be guessed.

```sql
CREATE OR REPLACE NETWORK RULE <PREFIX>_OF.RUNTIMES.EGRESS_RULE
  MODE = EGRESS TYPE = HOST_PORT
  VALUE_LIST = ('<SFTP_HOST>:<SFTP_PORT>', '<STAGE_HOST>:443')
  COMMENT = 'SFTP + stage egress. [<PREFIX>]';

CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION <PREFIX>_EAI
  ALLOWED_NETWORK_RULES = (<PREFIX>_OF.RUNTIMES.EGRESS_RULE)
  ENABLED = TRUE
  COMMENT = 'Runtime egress. [<PREFIX>]';

GRANT USAGE ON INTEGRATION <PREFIX>_EAI TO ROLE <PREFIX>_RUNTIME_RL;
```

### Runtime

```sql
USE SCHEMA <PREFIX>_OF.RUNTIMES;
CREATE OPENFLOW RUNTIME <PREFIX>_RT
  IN DEPLOYMENT <PREFIX>_DEPLOYMENT
  NODE_TYPE = SMALL NODE_TYPE_TIER = 'S1'
  MIN_NODES = 1 MAX_NODES = 1
  EXECUTE_AS_ROLE = <PREFIX>_RUNTIME_RL
  EXTERNAL_ACCESS_INTEGRATIONS = (<PREFIX>_EAI)
  DISPLAY_NAME = '<PREFIX>_RT'
  COMMENT = 'File mover. [<PREFIX>]';

SELECT SYSTEM$WAIT_FOR_STABLE_OPENFLOW_RUNTIMES(900, '<PREFIX>_RT');
DESCRIBE OPENFLOW RUNTIME <PREFIX>_RT;   -- capture server_url and key
```

`DESCRIBE` returns `server_url` ending in `/nifi/`. The **API** base replaces that with
`/nifi-api`. `DESCRIBE` needs a current database, so `USE SCHEMA` first.

### API access

The NiFi API authenticates with a Snowflake programmatic access token as a bearer token.

```sql
ALTER USER <user> ADD PROGRAMMATIC ACCESS TOKEN <PREFIX>_PAT
  ROLE_RESTRICTION = '<PREFIX>_OF_ADMIN_RL'
  DAYS_TO_EXPIRY = 7
  COMMENT = 'Runtime API access. [<PREFIX>]';
```

**This statement returns the token as its result set**, so it will appear in the transcript. Keep
`DAYS_TO_EXPIRY` short and remove the token at teardown. Tell the user this is happening rather
than letting them discover a credential in the log later.

Then write the profile — never echo the token to the console:

```bash
mkdir -p ~/.nipyapi && chmod 700 ~/.nipyapi
cat > ~/.nipyapi/profiles.yml << 'EOF'
<PREFIX>_rt:
  nifi_url: "<server_url with /nifi/ replaced by /nifi-api>"
  nifi_bearer_token: "<token>"
EOF
chmod 600 ~/.nipyapi/profiles.yml

nipyapi --profile <PREFIX>_rt system get_nifi_version_info
```

If the Snowflake CLI cannot authenticate (an OAuth-configured connection may fail with
`Oauth code flow requirement 'client_id' is empty`), that is fine — the token only needs to reach
`profiles.yml`, and SQL can be run through the agent's own SQL tool.

---

## § Endpoint proof

Run before building anything. Substitute the values from Step 1.5.

```bash
# Key auth: refuse to proceed on a world-readable key
chmod 600 <KEYFILE>

head -c 200 /dev/urandom > /tmp/probe.bin
md5 -q /tmp/probe.bin 2>/dev/null || md5sum /tmp/probe.bin

sftp -o StrictHostKeyChecking=no -o IdentitiesOnly=yes -i <KEYFILE> \
     <SFTP_USER>@<SFTP_HOST> << 'EOF'
pwd
ls
put /tmp/probe.bin <REMOTE_PATH>/probe.bin
get <REMOTE_PATH>/probe.bin /tmp/probe.roundtrip
rename <REMOTE_PATH>/probe.bin <REMOTE_PATH>/probe.renamed
rm <REMOTE_PATH>/probe.renamed
bye
EOF

md5 -q /tmp/probe.roundtrip 2>/dev/null || md5sum /tmp/probe.roundtrip
```

For password auth, omit `-i` and enter the password at the prompt.

Read the results carefully — **each of these four outcomes means something different**, and they
are easy to confuse:

| Symptom | Meaning | Action |
|---|---|---|
| `ls` denied, `put` succeeds | Write without list. Common on locked-down drops. | `Create Directory = false`, `Disable Directory Listing = true` |
| `pwd` is `/` and the target dir "does not exist", but `put` to `/` lands in the target | Home directory already maps into the target. | `Remote Path = /`, **not** the directory name — otherwise it resolves to `dir/dir` |
| `rename` fails | Dot-rename atomicity unavailable. | `Dot Rename = false`, and tell them delivery is no longer atomic |
| Connection refused / timeout | Possibly an IP allowlist that excludes this laptop. | Note that the first canvas run is now the first real connectivity test, and continue |

Do not conclude "permissions are broken" from a failed `ls` alone. A successful `put` after a
failed `ls` reverses that diagnosis entirely.

---

## § Canvas

Six processors in one process group. Build with `nipyapi` for parameter contexts and assets, and
`curl` against the NiFi API for components — `nipyapi`'s process-group PUT can fail with
**HTTP 414 URI Too Long**.

### Session setup

```bash
export BASE_URL="<runtime API base>"
export PAT=$(awk -v p="<PREFIX>_rt:" '$0 ~ p {f=1} f && /nifi_bearer_token:/ \
  {gsub(/.*nifi_bearer_token: *"?|"?$/,""); print; exit}' ~/.nipyapi/profiles.yml)
export AUTH="Authorization: Bearer $PAT"
```

### Bundle versions

**Get NAR bundle versions from the API — they are not the NiFi version.** A runtime reporting
NiFi `2026.9.15.2` may carry NAR bundles versioned `2026.9.15.21`. Creating a component with the
wrong bundle version returns an empty response body, which reads as an unexplained failure.

```bash
curl -sk -H "$AUTH" "$BASE_URL/flow/processor-types" \
  | python3 -c "import sys,json; [print(t['type'], t['bundle']['version']) for t in json.load(sys.stdin)['processorTypes']]" | sort -u
curl -sk -H "$AUTH" "$BASE_URL/flow/controller-service-types" | ...   # same shape
```

### Private key as a parameter asset

The key must live on the runtime, not the laptop. Upload it as an asset on a parameter context
bound to the process group.

Assets require `Content-Type: application/octet-stream`, a `filename:` header, and
`--data-binary` — **not** multipart form data:

```bash
curl -sk -X POST -H "$AUTH" \
  -H "Content-Type: application/octet-stream" \
  -H "filename: <keyname>" \
  --data-binary "@<KEYFILE>" \
  "$BASE_URL/parameter-contexts/$CTX/assets"
```

Then link it with `nipyapi.parameters.prepare_parameter_with_asset(...)` +
`upsert_parameter_to_context(...)`. The parameter's value becomes a runtime-local path; reference
it in `PutSFTP` as `#{SFTPPrivateKey}`.

For password auth, skip all of this and set `PutSFTP`'s `Password` property from a sensitive
parameter instead.

Bind the context to the process group with curl (this is the 414 case):

```bash
REV=$(curl -sk -H "$AUTH" "$BASE_URL/process-groups/$PG" | python3 -c "import sys,json;print(json.load(sys.stdin)['revision']['version'])")
curl -sk -X PUT -H "$AUTH" -H "Content-Type: application/json" "$BASE_URL/process-groups/$PG" \
 -d "{\"revision\":{\"version\":$REV},\"component\":{\"id\":\"$PG\",\"parameterContext\":{\"id\":\"$CTX\"}}}"
```

### Controller services

| Name | Type | Configuration |
|---|---|---|
| `Snowflake_Connection` | `com.snowflake.openflow.runtime.services.snowflake.SnowflakeConnectionService` | `Authentication Strategy = SNOWFLAKE_MANAGED`, `Database Name = <PREFIX>_DB`, `Schema = EXPORT`, `Warehouse = <PREFIX>_WH`, `Role = <PREFIX>_RUNTIME_RL` |
| `Json_Writer` | `org.apache.nifi.json.JsonRecordSetWriter` | defaults |

With `SNOWFLAKE_MANAGED`, the `Account`, `User`, and `Password` properties stay empty and the
service still validates — the runtime's own identity is used. Enable both services and confirm
`validationStatus = VALID` before wiring anything.

### Processors

| # | Name | Type | Key configuration |
|---|---|---|---|
| 1 | `Export_Data` | `ExecuteSQLRecord` | `SQL Query` = `CALL <PREFIX>_DB.EXPORT.SP_EXPORT_<FEED_NAME>()`, `Record Writer` = `Json_Writer`, `Database Connection Pooling Service` = `Snowflake_Connection`, scheduling `300 sec`, **left stopped** |
| 2 | `Unwrap_Result` | `EvaluateJsonPath` | `Destination = flowfile-content`, one path: `$[0].SP_EXPORT_<FEED_NAME>` |
| 3 | `Extract_Fields` | `EvaluateJsonPath` | `Destination = flowfile-attribute`, `export_filename = $.filename`, `export_presigned_url = $.presigned_url`, `export_row_count = $.row_count` |
| 4 | `Fetch_Bytes` | `InvokeHTTP` | `HTTP Method = GET`, `HTTP URL = ${export_presigned_url}` |
| 5 | `Set_Filename` | `UpdateAttribute` | `filename = ${export_filename}` |
| 6 | `Upload_To_SFTP` | `PutSFTP` | see below |

The property on `ExecuteSQLRecord` is **`SQL Query`** — not "SQL select query".

**Why two `EvaluateJsonPath` processors and not one.** A `CALL` returning a `VARIANT` comes back
through the JSON writer as an **escaped string**, not as nested JSON:

```json
[{"SP_EXPORT_FEED":"{\n  \"filename\": \"...\",\n  \"presigned_url\": \"...\"\n}"}]
```

So `$[0].SP_EXPORT_FEED.filename` never resolves. Processor 2 extracts that string **to
content**, which unescapes it into valid JSON; processor 3 then reads the fields into attributes.
Inspect real flowfile content before trusting any JSONPath here.

`PutSFTP` configuration:

| Property | Value |
|---|---|
| `Hostname` / `Port` / `Username` | from Step 1.5 |
| `Private Key Path` | `#{SFTPPrivateKey}` (or set `Password` instead) |
| `Remote Path` | from the Step 3 finding — often `/` |
| `Create Directory` | `false` if List permission is absent |
| `Disable Directory Listing` | `true` if List permission is absent |
| `Dot Rename` | `true` — uploads to `.name` then renames, so the receiver never sees a partial file |
| `Conflict Resolution` | `REPLACE` |
| `Strict Host Key Checking` | `false` for a demo; flag it as unsuitable for production |

### Connections

```
Export_Data --success--> Unwrap_Result --matched--> Extract_Fields
  --matched--> Fetch_Bytes --Response--> Set_Filename --success--> Upload_To_SFTP
```

Auto-terminate the unused relationships so processors validate. Be honest that auto-terminating
`failure`, `Retry`, and `No Retry` means a failed delivery is **silently dropped** — acceptable
for a session, but production needs a retry loop and bulletin alerting.

Deleting a connection needs both parameters: `DELETE /connections/{id}?version=N&clientId=X`.
Omitting `clientId` can fail silently and leave a duplicate edge, which then clones flowfiles
down two paths.

After wiring, confirm all six report `validationStatus = VALID`, and trace the connection list to
confirm it is a single linear chain with exactly five edges.

### Run it

Start processors 2–6, leave 1 stopped, then fire processor 1 with `RUN_ONCE`:

```bash
curl -sk -X PUT -H "$AUTH" -H "Content-Type: application/json" \
  "$BASE_URL/processors/$P1/run-status" \
  -d "{\"revision\":{\"version\":$REV},\"state\":\"RUN_ONCE\"}"
```

Check for problems:

```bash
curl -sk -H "$AUTH" "$BASE_URL/flow/process-groups/$PG/status"        # queued should return to 0
curl -sk -H "$AUTH" "$BASE_URL/flow/bulletin-board?groupId=$PG"       # expect zero bulletins
```

To inspect a queued flowfile's content, the queue APIs need a cluster node id or they return
"The id of the node in the cluster is required":

```bash
NODE=$(curl -sk -H "$AUTH" "$BASE_URL/controller/cluster" | python3 -c "import sys,json;print(json.load(sys.stdin)['cluster']['nodes'][0]['nodeId'])")
curl -sk -H "$AUTH" "$BASE_URL/flowfile-queues/$CONN/flowfiles/$FF/content?clusterNodeId=$NODE"
```

### Version control

Structural changes cannot be saved to Snowflake's connector registry, which is read-only. The
flow lives only on this runtime, so **export the flow definition before any teardown** if they
want to keep it.

---

## Next

**Continue** to Step 6 in `SKILL.md` to verify the delivered file byte-for-byte.
