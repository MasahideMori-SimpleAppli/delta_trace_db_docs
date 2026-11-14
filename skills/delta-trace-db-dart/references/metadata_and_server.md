# Metadata, server-side use, persistence and listeners (Dart)

## Cause: who / when / what / why / from

Any query builder accepts `cause`. It is saved with the query, so logged queries become an audit trail.

- `Cause({required who, required when, required what, required why, required from, serial, chainParentSerial, context, confidenceScore = 1.0})`
- `Actor(type, id, {collectionPermissions, context, name, email, createdAt, updatedAt, lastAccess, lastAccessDay, operationInDay, deviceIds})`
  — `type` and `id` are positional. `EnumActorType` is `human`, `ai` or `system`.
- `TemporalTrace(nodes: [...])` records the time trail across systems; each step is
  `TimestampNode(timestamp: ..., location: ..., context: {...})`. Use UTC timestamps.

```dart
import 'package:delta_trace_db/delta_trace_db.dart';

final cause = Cause(
  who: Actor(EnumActorType.human, "user_123", name: "Alice"),
  when: TemporalTrace(nodes: [TimestampNode(timestamp: DateTime.now().toUtc(), location: "web app")]),
  what: "Update profile",
  why: "User edited the form",
  from: "web",
);
final q = RawQueryBuilder.updateOne(
  target: "users",
  queryNode: FieldEquals("id", 0),
  overrideData: {"name": "Alice Smith"},
  cause: cause,
).build();
print(q.toDict()["cause"]["what"]); // Update profile
```

## Permissions and server-side execution

`collectionPermissions` maps a collection name to `Permission([...EnumQueryType])`.
It is a whitelist: a query runs only if its type is listed for its target collection
(`aggregate` and `merge` need their own entries). Passing `null` allows everything —
fine in a front-end app, never on a server.
Grant `aggregate` only together with read access: counts and min / max values can reveal the
contents of a collection even though no items are returned.

Never trust permissions sent by a client. Build them on the server from your own user data
(for example from the authenticated user's `Actor.collectionPermissions` stored on the server).

Server flow: authenticate the request → restore the query from JSON → execute with server-side
permissions → save the query (it is the operation log) → return `result.toDict()` as JSON.
`executeQueryObject` accepts a `Query`, a `TransactionQuery` or the decoded JSON map directly.
A query without permission returns `isSuccess` false with `"Operation not permitted."`.

```dart
final server = DeltaTraceDatabase();
final json = RawQueryBuilder.add(target: "users", rawAddData: [{"name": "x"}]).build().toDict();
final denied = server.executeQueryObject(
  json,
  collectionPermissions: {"users": Permission([EnumQueryType.search])},
);
print(denied.isSuccess); // false
```

Run one database instance per server process (it is single-threaded and in memory), or load a
per-user database from storage on demand.

## Saving and loading

The whole database, a single collection, queries and results are all JSON-compatible
(use `jsonEncode` / `jsonDecode` from `dart:convert`).
Back up the database regularly and also keep every executed query; replaying queries from a backup
restores any later state.

```dart
final db = DeltaTraceDatabase();
db.executeQuery(RawQueryBuilder.add(target: "users", rawAddData: [{"name": "Alice"}]).build());
final saved = db.toDict(); // e.g. jsonEncode(saved) and write to a file
final restored = DeltaTraceDatabase.fromDict(saved);
print(restored.executeQuery(RawQueryBuilder.count(target: "users").build()).hitCount); // 1

final queryJson = RawQueryBuilder.count(target: "users").build().toDict();
final again = UtilQuery.convertFromJson(queryJson); // Query or TransactionQuery
print(again is Query); // true
```

`collectionToDict(name)` / `collectionFromDict(name, src)` save and restore one collection.
`QueryResult.fromDict` restores a result.

## Listeners (UI updates, Flutter state management)

`addListener(target, cb, {name})` registers a no-argument callback that runs after a query changes
the collection (after the whole transaction for transactions). `removeCollection` does not notify.
Pass the same `cb` (and `name`, if used) to `removeListener`.

```dart
var changes = 0;
void onUsersChanged() => changes++;
db.addListener("users", onUsersChanged);
db.executeQuery(RawQueryBuilder.add(target: "users", rawAddData: [{"name": "Bob"}]).build());
db.removeListener("users", onUsersChanged);
print(changes); // 1
```

In Flutter, call `setState` (or notify your state object) in the callback, and remove the listener in `dispose`.
