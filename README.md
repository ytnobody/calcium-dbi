# NAME

dbi - a driver-agnostic Database Interface for Calcium, written in **pure Calcium**

# SYNOPSIS

```calcium
use ytnobody/dbi;
use core.io!;

// Provide an adapter (a hash of functions). Real drivers inject their backend
// here; for this example a tiny in-memory fake is used.
func my_connect(dsn, options) = success({mem: options.scratch});
func my_disconnect(handle) = success(true);
func my_execute(handle, sql, params) = success(len(params));          // fake
func my_query(handle, sql, params) = success([{id: 1, name: "alice"}]); // fake
func my_prepare(handle, sql) = success("stmt-1");
func my_execute_prepared(stmt, params) = success(1);
func my_query_prepared(stmt, params) = success([{x: params[0]}]);
func my_finish(stmt) = success(true);

adapter = {
    name: "mem",
    connect: my_connect,
    disconnect: my_disconnect,
    execute: my_execute,
    query: my_query,
    prepare: my_prepare,
    execute_prepared: my_execute_prepared,
    query_prepared: my_query_prepared,
    finish: my_finish
};

dbh = dbi.connect(adapter, "mem://", {scratch: {}}) !? {
    success(v) => v
    failure(e) => { handled: false }
};

rows = dbi.query(dbh, "SELECT * FROM users", []) !? {
    success(v) => v
    failure(e) => []
};
io.println(to_string(rows[0].name)); // "alice"
```

# DESCRIPTION

`dbi` is a DBI-style database interface implemented entirely in the Calcium
language. It provides a small, consistent API (`connect`, `execute`, `query`,
`prepare`, prepared-statement execution, `finish`, and SQL-literal `quote`) --
without depending on any Go/VML primitive or baked-in database driver.

Database backends are supplied by the caller as an **adapter**: a hash of
functions (dependency injection). The DBI layer handles argument validation,
handle/statement bookkeeping, consistent `success(...)` / `failure(...)`
results, parameter arrays, integer/float/string/bool/null quoting, and first-row
helpers -- all in pure Calcium.

This design keeps the interface portable and unit-testable: any host, embedder,
or pure-Calcium driver that satisfies the adapter contract can be plugged in,
and the module can be exercised with a purely in-memory fake driver for tests.

# ADAPTER CONTRACT

A driver adapter is a hash with these callable keys:

```calcium
{
  name                         : "driver-name",
  connect(dsn, options)        : -> success(handle) | failure(msg)
  disconnect(handle)           : -> success(any)    | failure(msg)
  execute(handle, sql, params) : -> success(rowsAffected) | failure(msg)
  query(handle, sql, params)   : -> success(rows)  | failure(msg)
  prepare(handle, sql)         : -> success(stmtHandle) | failure(msg)
  execute_prepared(stmtHandle, params) : -> success(rowsAffected) | failure(msg)
  query_prepared(stmtHandle, params)   : -> success(rows)  | failure(msg)
  finish(stmtHandle)           : -> success(any)    | failure(msg)
}
```

- `handle` / `stmtHandle` are opaque values owned by the adapter.
- `params` is an array of positional bind parameters (use `[]` for none).
- `rows` is an array of hashes (column name -> value).

# FUNCTIONS

## connect(adapter, dsn, options)

Validates the adapter (must provide all callable methods) then connects.
Returns `success(dbh)` where `dbh` is `{driver, adapter, handle, open: true}`,
or `failure(message)`.

```calcium
dbi.connect(adapter, "file:app.db", {})   // success(dbh) | failure(msg)
```

## disconnect(dbh)

Closes a database handle. Returns `success` or `failure`.

## execute(dbh, sql, params)

Runs a non-query statement (DDL / DML). Returns `success(rowsAffected)` or
`failure`.

```calcium
dbi.execute(dbh, "INSERT INTO users(name) VALUES (?)", ["alice"]);
```

## query(dbh, sql, params)

Runs a SELECT. Returns `success(rows)` (array of hashes) or `failure`.

```calcium
rows = dbi.query(dbh, "SELECT id, name FROM users WHERE id = ?", [1]) !? {
    success(v) => v
    failure(_) => []
};
```

## query_row(dbh, sql, params)

Like `query` but returns only the first row as `success(row)`, or
`failure("no rows returned")` when empty.

## query_meta(dbh, sql, params)

Runs a SELECT and returns its result **together with schema metadata**:
`success({columns, column_types, rows})`. `column_types` holds each result
column's declared SQL type (parallel to `columns`). This is driver-agnostic
plumbing; deriving per-column constraints/validators from the types is done by
the DBD (driver), not the generic DBI layer.

## prepare(dbh, sql)

Compiles a statement. Returns `success(stmt)` or `failure`.

## execute_prepared(stmt, params)

Executes a prepared non-query statement. Returns `success(rowsAffected)` or
`failure`.

## query_prepared(stmt, params)

Executes a prepared SELECT. Returns `success(rows)` or `failure`.

## finish(stmt)

Releases a prepared statement. Returns `success` or `failure`.

## quote(value)

Returns a SQL literal safe for interpolation:
- `int` / `float` -- as-is (via `to_string`)
- `string` -- wrapped in `'...'` with `'` escaped as `''`
- `bool` -- `1` / `0`
- `null` -- `NULL`
- `array` -- `(v1,v2,...)` for IN-lists

```calcium
dbi.quote("it's")   // "'it''s'"
dbi.quote(42)       // "42"
dbi.quote([1, 2])   // "(1,2)"
```

# TESTS

`mod.test.ca` drives the whole DBI layer against an injected fake driver, so no
database is required:

```bash
mkdir -p dbi && ln -sf ../mod.ca dbi/mod.ca   # make local module resolvable
calcium mod.test.ca
```