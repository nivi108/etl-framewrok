# SQL Server Reader Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Ingestion_Framework.md (v1.0), Read_Config_Component.md (v1.0)
**Category:** Components

## Purpose
Handles the actual connection to and read from SQL Server, for all three load types (Full, Incremental, CDC). This is the source-specific component `Full_Load_Component.md`, `Incremental_Load_Component.md`, and `CDC_Component.md` all call into for the actual data retrieval — they define *what to read*, this component defines *how the connection and query execution happens*.

## Scope
Covers connection handling and query execution against SQL Server only. Does NOT decide which rows to read (that's determined by the calling Full/Incremental/CDC component, based on config) and does NOT write anywhere (that's `Raw_Framework.md`'s write logic).

## Inputs
| Input | Type | Description |
|---|---|---|
| `connection_config` | object | Server, database, auth details (sourced from a secrets store, never hardcoded — see `Deployment/Secrets_Management.md`) |
| `query` | string | The SQL query or CDC function call to execute |
| `fetch_mode` | enum(full, batch) | Whether to fetch all rows at once or paginate for very large tables |

## Outputs
A Spark DataFrame representing the query result, with source column names and types preserved as-is (no transformation at this stage — that belongs to Silver).

## Configuration
- Connection details are never embedded in this component's code or in any config table in plaintext — they are retrieved via Databricks secret scopes at runtime.
- JDBC connection parameters (driver, fetch size, timeout) are set here as sensible defaults, overridable via `connection_config` if a specific source needs tuning.

## Implementation Rules
- Always use the JDBC connection with `fetchsize` tuned appropriately for large tables — never default to unbounded single-batch fetch for tables with a large row count.
- Connections must be closed/released properly after each read — no lingering open connections across notebook runs.
- Must never log connection strings or credentials, even at debug level — per the privacy principle in `Logging_Framework.md`.

## Error Handling
| Scenario | Behavior |
|---|---|
| Connection failure | Treated as transient — retried per `Error_Handling_Framework.md` retry rules |
| Query timeout | Treated as transient — retried with backoff, up to `Workflow_Config.timeout_minutes` |
| Query returns malformed/unexpected structure | Not retried — raised as a data problem, routed to schema validation failure path |
| Authentication failure | Not retried — this is a configuration problem, not transient; raised immediately with a clear error, alerted |

## Logging
Logs query start/end time, row count returned, and any retry attempts — never logs the query text itself if it might contain literal filter values considered sensitive (e.g., PII in a WHERE clause); logs a sanitized/parameterized version instead.

## Return Values
Returns a Spark DataFrame, or raises a named exception (`SourceConnectionError`, `SourceQueryError`, `SourceAuthError`).

## Pseudo Logic
```
FUNCTION read_from_sql_server(connection_config, query, fetch_mode):
    conn = OPEN_CONNECTION(connection_config)   # via secret scope, never plaintext
    TRY:
        df = EXECUTE_QUERY(conn, query, fetch_mode)
        LOG(row_count=df.count(), query_sanitized=SANITIZE(query))
        RETURN df
    CATCH AuthError:
        RAISE SourceAuthError(no_retry=True)
    CATCH TimeoutError, ConnectionError:
        RAISE SourceConnectionError(retryable=True)
    FINALLY:
        CLOSE_CONNECTION(conn)
```

## Acceptance Criteria
- [ ] No credentials or connection strings appear in logs, code, or config tables in plaintext.
- [ ] Query timeouts and connection failures are retried; authentication failures are not.
- [ ] Returned DataFrame preserves source schema exactly, with zero transformation applied.

## Example (Illustrative Only)
```
Input: query = "SELECT * FROM dbo.Orders WHERE modified_date > @watermark"
Output: DataFrame with columns matching Orders table schema, row count logged
```

## Dependencies
- `Config_Framework.md` (v1.0) — connection details are sourced per `Source_Config.source_system`.
- `Ingestion_Framework.md` (v1.0) — this component implements the "Read Source" step for all three load types.
- `Read_Config_Component.md` (v1.0) — supplies the config this component's caller uses to build the query.

## Future Extension Points
- Could add support for additional source types (Salesforce, REST APIs) by introducing sibling components (e.g., `Salesforce_Reader_Component.md`) following the same input/output contract — this is the extension point flagged in the genericity discussion.

## AI Generation Notes
Any agent generating this component's implementation must retrieve credentials exclusively via Databricks secret scopes, never via hardcoded values or plaintext config fields — this is a hard security requirement, not a style preference.