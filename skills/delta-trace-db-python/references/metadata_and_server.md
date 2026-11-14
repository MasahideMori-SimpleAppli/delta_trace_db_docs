# Metadata, server-side use, persistence and listeners (Python)

## Cause: who / when / what / why / from

Any query builder accepts `cause`. It is saved with the query, so logged queries become an audit trail.

- `Cause(who, when, what, why, from_, serial=None, chain_parent_serial=None, context=None, confidence_score=1.0)`
  — `who`, `when`, `what`, `why` and `from_` are required.
- `Actor(actor_type, actor_id, collection_permissions=None, context=None, name=None, email=None, created_at=None,
  updated_at=None, last_access=None, last_access_day=None, operation_in_day=None, device_ids=None)`.
  `EnumActorType` is `human`, `ai` or `system`.
- `TemporalTrace(nodes=[...])` records the time trail across systems; each step is
  `TimestampNode(timestamp, location, context=None)`. Use timezone-aware UTC datetimes.

```python
from datetime import datetime, timezone

from delta_trace_db import *

cause = Cause(
    who=Actor(EnumActorType.human, actor_id="user_123", name="Alice"),
    when=TemporalTrace(nodes=[TimestampNode(timestamp=datetime.now(timezone.utc), location="web app")]),
    what="Update profile",
    why="User edited the form",
    from_="web",
)
q = RawQueryBuilder.update_one(
    target="users", query_node=FieldEquals("id", 0), override_data={"name": "Alice Smith"}, cause=cause).build()
print(q.to_dict()["cause"]["what"])  # Update profile
```

## Permissions and server-side execution

`collection_permissions` maps a collection name to `Permission([...EnumQueryType])`.
It is a whitelist: a query runs only if its type is listed for its target collection
(`EnumQueryType.aggregate` and `EnumQueryType.merge` need their own entries).
Passing `None` allows everything — fine in a local script, never on a server.
Grant `aggregate` only together with read access: counts and min / max values can reveal the
contents of a collection even though no items are returned.

Never trust permissions sent by a client. Build them on the server from your own user data
(for example from the authenticated user's `Actor.collection_permissions` stored on the server).

Server flow: authenticate the request → restore the query from JSON → execute with server-side
permissions → save the query (it is the operation log) → return `result.to_dict()`.

```python
from fastapi import FastAPI, Request

from delta_trace_db import DeltaTraceDatabase, EnumQueryType, Permission, UtilQuery

app = FastAPI()
db = DeltaTraceDatabase()


@app.post("/db")
async def run_query(request: Request):
    user_permissions = {  # look these up for the authenticated user
        "users": Permission([EnumQueryType.search, EnumQueryType.add]),
    }
    query = UtilQuery.convert_from_json(await request.json())  # Query or TransactionQuery
    result = db.execute_query_object(query, collection_permissions=user_permissions)
    return result.to_dict()
```

`execute_query_object` also accepts the JSON dict directly.
A query without permission returns `is_success` False with `"Operation not permitted."`.

```python
server = DeltaTraceDatabase()
query_json = RawQueryBuilder.add(target="users", raw_add_data=[{"name": "x"}]).build().to_dict()
denied = server.execute_query_object(query_json, collection_permissions={"users": Permission([EnumQueryType.search])})
print(denied.is_success)  # False
```

Run one database instance per server process (it is single-threaded and in memory; do not use
multiple worker processes with one shared database), or load a per-user database from storage on demand.

## Saving and loading

The whole database, a single collection, queries and results are all JSON-compatible (use the `json` module).
Back up the database regularly and also keep every executed query; replaying queries from a backup
restores any later state.

```python
db = DeltaTraceDatabase()
db.execute_query(RawQueryBuilder.add(target="users", raw_add_data=[{"name": "Alice"}]).build())
saved = db.to_dict()  # e.g. json.dump(saved, f)
restored = DeltaTraceDatabase.from_dict(saved)
print(restored.execute_query(RawQueryBuilder.count(target="users").build()).hit_count)  # 1

query_json = RawQueryBuilder.count(target="users").build().to_dict()
again = UtilQuery.convert_from_json(query_json)  # Query or TransactionQuery
print(isinstance(again, Query))  # True
```

`db.collection_to_dict(name)` / `db.collection_from_dict(name, src)` save and restore one collection.
`QueryResult.from_dict(src)` restores a result.

## Listeners

`db.add_listener(target, cb, name=None)` registers a no-argument callback that runs after a query
changes the collection (after the whole transaction for transactions). `remove_collection` does not notify.
Pass the same `cb` (and `name`, if used) to `remove_listener`.

```python
changes = []


def on_users_changed():
    changes.append(1)


db.add_listener("users", on_users_changed)
db.execute_query(RawQueryBuilder.add(target="users", raw_add_data=[{"name": "Bob"}]).build())
db.remove_listener("users", on_users_changed)
print(len(changes))  # 1
```
