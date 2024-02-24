---
title: Sqlalchemy autocommit
date: "2024-02-24T22:15:03.284Z"
description: "Sqlahchemy autocommit explanation"
---

tl;dr

`autocommit=False` means this:

```python
query = "SELECT 1"
if not autocommit:
	query = f'BEGIN {query}' # implicit transaction

conn.execute(query)
conn.commit()  # without this it won't commit/rollback
```

`autocommit=True` means this:

```python
query = "SELECT 1"
conn.execute(query)  # commited automatically, no transaction

# explicit transaction
with conn.begin():
  conn.execute(query)
  # commited (or rolled back on error)
```

Long answer:

Python has DBAPI [PEP 249](https://peps.python.org/pep-0249/). All existing drivers try to be PEP-249-compatible.

One of the main PEP's design choices is an `implicit transaction` - basically, PEP 249 has no `begin` method - which means there must always be a transaction.

Postgres has no such feature as an implicit transaction so [`psycopg`](https://www.psycopg.org/docs/usage.html#transactions-control) for example just adds `BEGIN` to all queries - this is emulated PEP 249 implicit transaction.

`autocommit` in psycopg (and in sqlachemy) is a way to disable this implicit `BEGIN`. Once `autocommit=True` psycopg won't append any `BEGIN` to you query. This means that each query is executed and commited in one go - you can not roll it back. If you want to rollback you have to call `.begin()` yourself and this will let you commit/rollback explicitly.

Sqlalchemy 2.0 dropped library-level `autocommit` - which means that you can not execute a query without a transaction - sqlalchemy will always open a transaction for you query but you need to commit it yourself.

So the main difference is that in sqlalchemy this query commits if `autocommit=True` :

```python
conn.execute('DELETE FROM users WHERE id = 1')
```

but in sqlachemy 2.0 it won't. You need to commit explicitly now

```python
conn.execute('DELETE FROM users WHERE id = 1')
conn.commit()
```

As sqla docs [says](https://docs.sqlalchemy.org/en/20/changelog/migration_20.html#driver-level-autocommit-remains-available) , "true" `autocommit` is available via driver.
> For sqlahchemy "true" autocommit these days [means](https://docs.sqlalchemy.org/en/20/core/connections.html#dbapi-autocommit) a separate transaction level.

One interesting advice from docs is to use a separate engine for read-only queries with `AUTOCOMMIT`:

>One such use case is an application that has operations that break into “transactional” and “read-only” operations, a separate [`Engine`](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Engine "sqlalchemy.engine.Engine") that makes use of `"AUTOCOMMIT"` may be separated off from the main engine:

```python
from sqlalchemy import create_engine

eng = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

autocommit_engine = eng.execution_options(isolation_level="AUTOCOMMIT")
```

`autocommit_engine` here will be used for read-only queries so settings `AUTOCOMMIT` here will tell the driver not to emit `BEGIN` as the doc says

> It is important to note that “autocommit” mode persists even when the [`Connection.begin()`](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Connection.begin "sqlalchemy.engine.Connection.begin") method is called; the DBAPI will not emit any BEGIN to the database, nor will it emit COMMIT when [`Connection.commit()`](https://docs.sqlalchemy.org/en/20/core/connections.html#sqlalchemy.engine.Connection.commit "sqlalchemy.engine.Connection.commit") is called.

Links:
- https://www.oddbird.net/2014/06/14/sqlalchemy-postgres-autocommit/
- https://python-gino.org/docs/en/1.0/explanation/why.html
- https://www.psycopg.org/docs/usage.html#transactions-control
- https://www.psycopg.org/psycopg3/docs/basic/transactions.html
- https://docs.sqlalchemy.org/en/20/changelog/migration_20.html#library-level-but-not-driver-level-autocommit-removed-from-both-core-and-orm
- https://docs.sqlalchemy.org/en/20/changelog/migration_20.html#driver-level-autocommit-remains-available
- https://docs.sqlalchemy.org/en/20/core/connections.html#dbapi-autocommit
- https://docs.sqlalchemy.org/en/20/core/connections.html#dbapi-autocommit-understanding
