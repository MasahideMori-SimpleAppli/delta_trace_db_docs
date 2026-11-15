---
name: delta-trace-db-python
description: Write correct Python code for DeltaTraceDB (delta-trace-db on PyPI, imported as delta_trace_db), an in-memory NoSQL database whose queries are serializable classes. Use this skill whenever Python code imports or mentions delta_trace_db, DeltaTraceDatabase, QueryBuilder, RawQueryBuilder, TransactionQuery or its query nodes, or when the user wants to store, search, sort, page, aggregate or audit data with DeltaTraceDB in Python, or run it on a server (e.g. FastAPI) with per-collection permissions, even if they only describe the task without naming every class.
---

# DeltaTraceDB for Python

DeltaTraceDB (`pip install delta-trace-db`, `import delta_trace_db`) is a lightweight in-memory NoSQL database.
Data is stored per **collection** as JSON-compatible dict items.
Every operation is a **Query object** that can be serialized to JSON, so queries can be received from a
front end, logged, and replayed to rebuild the database at any point in time.
(A Dart / Flutter version with the same query JSON exists, so a Python server can serve a Flutter app.)

This file holds the rules that prevent most mistakes. The details live in `references/`;
read the file that matches the task before writing non-trivial code.

| Task | Read |
|---|---|
| Any query type (add, update, delete, search, get_all, count, clear, clear_add, conform_to_template, rename_field, remove_collection, merge), paging, transactions | `references/queries.md` |
| Search conditions, `EnumValueType` (incl. exact `decimal_` / `rational_`), sorting | `references/conditions_and_sort.md` |
| count / sum / min / max / mean (`aggregate` query) | `references/aggregate.md` |
| Cause / Actor / Permission, server-side execution (FastAPI), saving and loading, listeners | `references/metadata_and_server.md` |
| Argument names that are not plain snake_case, enum member names, JSON keys | `references/naming.md` |

## Core rules

1. **Build, then execute.** Builder class methods return a builder, not a query.
   Call `.build()` and pass the `Query` to `db.execute_query(...)`. Passing the builder itself is
   unsupported: it may look like it works for some queries but breaks for others.
2. **Choose the right builder.**
   `QueryBuilder` takes model objects (subclasses of `CloneableFile`) in `add_data`.
   `RawQueryBuilder` takes plain dicts in `raw_add_data`. Everything else is identical.
3. **Check the result.** Queries do not raise for normal failures; they return a `QueryResult` with
   `is_success` and `error_message`. Always check it.
   Write queries (add, update, delete, ...) fail by default when they affect 0 items
   (`must_affect_at_least_one=True` by default).
4. **Use the right execute method.** `execute_query` accepts only a `Query`.
   A `TransactionQuery` goes to `execute_transaction_query`.
   `execute_query_object` accepts a `Query`, a `TransactionQuery` or a JSON dict,
   which is what a server should use for incoming requests.
5. **Names are snake_case with a few exceptions**: `type_`, `from_`, `reversed_`, `actor_type`, `actor_id`,
   and enum members keep their camelCase names (`EnumQueryType.updateOne`, `EnumValueType.floatStrict_`).
   See `references/naming.md`.
6. **Nested fields use dot notation** (`"address.city"`) in nodes, sorts and aggregations.
   `rename_field` only works on top-level fields.
7. **Store datetimes as UTC ISO 8601 strings** and compare them with `EnumValueType.datetime_`
   (passing a timezone-aware `datetime` as a node value sets it automatically). Aware and naive values are
   never compared with each other; pass `naive_utc_offset_minutes` (e.g. 540 for UTC+9) when stored values are naive.
8. **Store exact numbers such as prices as strings** (`"10.50"`) and compare / sort / aggregate them with
   `EnumValueType.decimal_` or `rational_`. These types convert only `str` and
   `PortableDecimal` / `PortableRational` objects; `int`, `float` and `bool` values are treated as conversion failures.
9. **Do not use the built-in `sum()` on DB values when results must match other systems** —
   use the `aggregate` query (Python 3.12+ `sum()` uses compensated summation).
10. **The database is single-threaded and in memory.** Do not share one instance across threads or
    processes; run one instance per server process, or load a per-user database on demand.

## Quick start

```python
from typing import Any, Dict

from delta_trace_db import (DeltaTraceDatabase, FieldGreaterThanOrEqual, QueryBuilder,
                            SingleSort)
from file_state_manager import CloneableFile


class User(CloneableFile):
    def __init__(self, id: int, name: str, age: int):
        super().__init__()
        self.id = id
        self.name = name
        self.age = age

    @classmethod
    def from_dict(cls, src: Dict[str, Any]) -> "User":
        return cls(id=src["id"], name=src["name"], age=src["age"])

    def to_dict(self) -> Dict[str, Any]:
        return {"id": self.id, "name": self.name, "age": self.age}

    def clone(self) -> "User":
        return User.from_dict(self.to_dict())


db = DeltaTraceDatabase()

add_result = db.execute_query(QueryBuilder.add(
    target="users",
    add_data=[User(id=-1, name="Alice", age=30), User(id=-1, name="Bob", age=25)],
    serial_key="id",  # overwrites "id" with 0, 1, 2, ...
).build())
if not add_result.is_success:
    print(add_result.error_message)

r = db.execute_query(QueryBuilder.search(
    target="users",
    query_node=FieldGreaterThanOrEqual("age", 26),
    sort_obj=SingleSort(field="age", reversed_=True),
    limit=10,
).build())
users = r.convert(User.from_dict)
print([u.name for u in users])  # ['Alice']
```

`CloneableFile` comes from the `file-state-manager` package, which is installed with DeltaTraceDB.
The serial key field must already exist in each added item (any placeholder value works).
Without model classes, use `RawQueryBuilder` and plain dicts instead.

## Result object

`QueryResult` attributes:

- `is_success`: whether the query succeeded.
- `result`: list of dicts. Search results, returned data, or the aggregate result dict.
- `db_length`: number of items in the collection after the query.
- `update_count`: number of items added, changed or deleted.
- `hit_count`: number of matched items (before paging for search).
- `error_message`: reason for failure, or `None`.
- `type`: the `EnumQueryType` of the query.
- `convert(from_dict)`: converts `result` to model objects.
- `to_dict()`: JSON-compatible dict (keys are camelCase, e.g. `"isSuccess"`).

A transaction returns `TransactionQueryResult` with `is_success`, `results` (one `QueryResult` per query)
and `error_message`.

## Things that look right but are wrong

- `RawQueryBuilder.add(target=..., add_data=[...])` — the raw builder's argument is `raw_add_data`.
- `QueryBuilder.add(add_data=[user.to_dict()])` — `QueryBuilder` needs model objects; use `RawQueryBuilder` for dicts.
- `db.execute_query(TransactionQuery(...))` — raises; use `execute_transaction_query` or `execute_query_object`.
- `SingleSort(field="age", reversed=True)` / `Cause(..., from="web")` — use `reversed_` / `from_`.
- `EnumQueryType.update_one` — enum members are not snake_case; it is `EnumQueryType.updateOne`.
- `FieldEquals("price", 10.5, v_type=EnumValueType.decimal_)` — numbers are not converted by `decimal_`; pass `"10.5"`.
- Expecting `count` to accept a condition — `count` returns the collection size;
  count matching items with `aggregate` + `EnumAggregateType.count_`.
- Putting `remove_collection` or `merge` in a transaction — they are not allowed there.
