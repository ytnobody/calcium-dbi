# DBI Text-Stream Protocol

A common, RDBMS-agnostic request/response protocol exchanged over a text
stream (stdin/stdout) between the DBI layer and a long-lived **driver backend**
process. It mirrors the `calcium-dbi` adapter contract so any RDBMS only needs
a small executable that speaks this wire format.

- **Status**: Draft
- **Version**: 1.0
- **Transport**: UTF-8 text over stdio (one backend process per DBI connection)
- **Framing**: one message per line (NDJSON); every line is a complete, flushed message

## Goals

- Keep the DBI/adapter layer free of any RDBMS-specific logic.
- Isolate driver concerns (SQLite, Postgres, MySQL, ...) into a backend
  executable behind one identical wire format.
- Support persistent connections, transactions, and prepared statements.
- Remain simple enough to implement in ~100 lines in any language.

## Framing

- Each message is a single JSON object on one line terminated by `\n`.
- Requests flow client -> backend; responses flow backend -> client, in the
  same order (no pipelining in v1; a request is answered before the next).
- The backend must flush after every response line so the driver never blocks.
- Unknown fields MUST be ignored (forward compatible). Unknown `op` -> error
  response.
- All numbers in parameters/results follow JSON semantics (integers and floats
  represented as JSON numbers; SQLite/other DB integers should be emitted as
  JSON numbers).

## Connection setup

A backend process is launched once per DBI connection with the connection
parameters supplied out-of-band:

```text
dbd-sqlite <dsn> [--param k=v ...]     # backend-specific CLI
```

The backend reads requests from stdin and writes responses to stdout. stderr is
reserved for logs and diagnostics (never used for protocol responses).

## Requests

Every request has an `op` field. Common optional fields: `params` (array), and
for prepared statements `stmt` (integer handle).

### `execute`

Run a non-query statement (DDL / DML / transaction control).

```json
{"op": "execute", "sql": "INSERT INTO users(name) VALUES (?)", "params": ["alice"]}
```

Response:

```json
{"ok": true, "rows_affected": 1, "last_insert_id": 42}
```

### `query`

Run a SELECT and return rows.

```json
{"op": "query", "sql": "SELECT id, name FROM users WHERE id = ?", "params": [1]}
```

Response:

```json
{"ok": true, "columns": ["id", "name"], "rows": [{"id": 1, "name": "alice"}]}
```

`rows` is an array of hashes (column name -> JSON value). `columns` is optional
but encouraged.

### `query_row`

Like `query` but backend returns only the first row. Endpoint is optional; a
backend may treat it as `query` and let the DBI trim.

```json
{"op": "query_row", "sql": "SELECT id, name FROM users", "params": []}
```

Response:

```json
{"ok": true, "row": {"id": 1, "name": "alice"}}
```

### `prepare`

Compile a statement for reuse.

```json
{"op": "prepare", "sql": "SELECT ? ?"}
```

Response:

```json
{"ok": true, "stmt": 3}
```

`stmt` is an opaque backend-owned integer handle, valid until `finish`.

### `execute_prepared`

```json
{"op": "execute_prepared", "stmt": 3, "params": [7]}
```

Response: same as `execute`.

### `query_prepared`

```json
{"op": "query_prepared", "stmt": 3, "params": [7]}
```

Response: same as `query`.

### `finish`

Release a prepared statement.

```json
{"op": "finish", "stmt": 3}
```

Response:

```json
{"ok": true}
```

### `begin` / `commit` / `rollback`

Transaction control.

```json
{"op": "begin"}
```

Responses:

```json
{"ok": true}
```

### `ping`

Keep-alive / liveness check.

```json
{"op": "ping"}
```

Response:

```json
{"ok": true, "pong": true, "driver": "sqlite", "version": "1.0.0"}
```

### `close`

Gracefully shut down the backend (the driver then exits the process).

```json
{"op": "close"}
```

Response:

```json
{"ok": true}
```

After responding `close`, the backend SHOULD exit(0).

## Responses

Every response has at least an `ok` field.

Success:

```json
{"ok": true, ...}
```

Error:

```json
{"ok": false, "error": "constraint failed: NOT NULL constraint failed: users.name"}
```

`error` is a human-readable message. Optional `code` (string) may be included
for programmatic classification (e.g. `"constraint"`, `"syntax"`, `"io"`).

## Type mapping

### Parameters (client -> backend)

| Calcium / JSON value | DB binding |
|----------------------|------------|
| `null`               | NULL       |
| `true` / `false`     | BOOLEAN/INT |
| integer              | INTEGER    |
| float                | REAL       |
| string               | TEXT       |
| array                | list (backend-dependent) |

### Results (backend -> client)

| SQL type        | JSON value        |
|-----------------|-------------------|
| NULL            | `null`            |
| INTEGER         | integer           |
| REAL            | number (float)    |
| TEXT            | string            |
| BLOB            | string (byte-safe encoding; see below) |
| BOOLEAN         | `true` / `false`  |

BLOB bytes that are not valid UTF-8 MUST be encoded (e.g. base64) and the value
shape documented by the backend. v1: SQLite TEXT/INTEGER/REAL/NULL/BLOB; BLOB
encoded as a string.

## Invariants

- One backend process handles exactly one connection; `close` ends it.
- Messages are strictly sequential; responses are in request order.
- A backend MUST never produce protocol bytes on stderr.
- Unknown fields must be ignored; unknown `op` must error.
- The DBI adapter is a thin, stateless mapper between the `calcium-dbi` adapter
  contract and this wire format.

## Example session (sqlite backend)

```
$ dbd-sqlite app.db
{"op": "execute", "sql": "CREATE TABLE IF NOT EXISTS users(id INTEGER PRIMARY KEY, name TEXT)", "params": []}
{"ok": true, "rows_affected": 0, "last_insert_id": null}
{"op": "execute", "sql": "INSERT INTO users(name) VALUES (?)", "params": ["alice"]}
{"ok": true, "rows_affected": 1, "last_insert_id": 1}
{"op": "query", "sql": "SELECT id, name FROM users", "params": []}
{"ok": true, "columns": ["id", "name"], "rows": [{"id": 1, "name": "alice"}]}
{"op": "close"}
{"ok": true}
$ echo $?   # 0
```