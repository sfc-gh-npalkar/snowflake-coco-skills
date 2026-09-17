# Teardown

Everything created carries `[<PREFIX>]` in its `COMMENT`, so the inventory is discoverable rather
than remembered.

**Lead with the cost point:** the Openflow deployment holds a compute pool that bills
continuously until it is dropped. If only one thing gets done, it is dropping the runtime and
deployment.

---

## Before dropping anything

Two things are unrecoverable afterwards:

- **The flow definition.** It lives only on the runtime and cannot be saved to Snowflake's
  read-only connector registry. Export it if they want to keep it:
  ```bash
  curl -sk -H "$AUTH" "$BASE_URL/process-groups/$PG/download" -o <PREFIX>_flow.json
  ```
- **Files already delivered to the SFTP endpoint.** These are the customer's; do not remove them
  without asking.

Ask whether to keep the Snowflake data objects. Teardown is often partial — customers frequently
want the export view and procedure to stay while the Openflow infrastructure goes.

---

## Order matters

Drop the runtime before the deployment; a deployment with a live runtime will not drop cleanly.

```sql
-- 1. Openflow compute first -- this is what stops the billing
USE ROLE <PREFIX>_OF_ADMIN_RL;
DROP OPENFLOW RUNTIME <PREFIX>_OF.RUNTIMES.<PREFIX>_RT;
DROP OPENFLOW DEPLOYMENT <PREFIX>_DEPLOYMENT;

-- 2. Egress
DROP EXTERNAL ACCESS INTEGRATION <PREFIX>_EAI;
DROP NETWORK RULE <PREFIX>_OF.RUNTIMES.EGRESS_RULE;

-- 3. Infra database
DROP DATABASE <PREFIX>_OF;

-- 4. Data objects -- skip this block if they want to keep the export
DROP DATABASE <PREFIX>_DB;
DROP WAREHOUSE <PREFIX>_WH;

-- 5. Credentials
ALTER USER <user> REMOVE PROGRAMMATIC ACCESS TOKEN <PREFIX>_PAT;

-- 6. Roles last -- they own the objects above
USE ROLE ACCOUNTADMIN;
DROP ROLE <PREFIX>_RUNTIME_RL;
DROP ROLE <PREFIX>_OWNER_RL;
DROP ROLE <PREFIX>_OF_ADMIN_RL;
```

Account-level grants disappear with the roles, so there is nothing to revoke separately.

---

## Local cleanup

```bash
rm -f ~/.nipyapi/profiles.yml          # contains the bearer token
rm -f /tmp/probe.* /tmp/*.parquet
```

The uploaded private key asset goes with the runtime. The original key file on disk is the
customer's — leave it, but remind them it is still there.

---

## Verify

```sql
SHOW OPENFLOW DEPLOYMENTS;            -- expect none matching <PREFIX>
SHOW DATABASES LIKE '<PREFIX>%';
SHOW WAREHOUSES LIKE '<PREFIX>%';
SHOW ROLES LIKE '<PREFIX>%';
SHOW INTEGRATIONS LIKE '<PREFIX>%';
SELECT * FROM TABLE(INFORMATION_SCHEMA.PROGRAMMATIC_ACCESS_TOKENS(USER_NAME => '<user>'));
```

All should come back empty for the prefix, except anything the customer chose to keep — say
explicitly what was kept so there are no surprises on the next bill.
