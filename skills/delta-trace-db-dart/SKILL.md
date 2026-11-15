---
name: delta-trace-db-dart
description: Write correct Dart / Flutter code for DeltaTraceDB (the delta_trace_db package on pub.dev), an in-memory NoSQL database whose queries are serializable classes. Use this skill whenever Dart or Flutter code imports or mentions delta_trace_db, DeltaTraceDatabase, QueryBuilder, RawQueryBuilder, TransactionQuery or its query nodes, or when the user wants to store, search, sort, page, aggregate or audit data with DeltaTraceDB in Dart, or use it for Flutter state management, even if they only describe the task without naming every class.
---

# DeltaTraceDB for Dart

DeltaTraceDB (`delta_trace_db` on pub.dev) is a lightweight in-memory NoSQL database for Dart and Flutter.
Data is stored per **collection** as JSON-compatible `Map<String, dynamic>` items.
Every operation is a **Query object** that can be serialized to JSON, so queries can be sent to a
server, logged, and replayed to rebuild the database at any point in time.
(A Python version with the same query JSON exists, so a Dart front end can talk to a Python server.)

This file holds the rules that prevent most mistakes. The details live in `references/`;
read the file that matches the task before writing non-trivial code.

| Task | Read |
|---|---|
| Any query type (add, update, delete, search, getAll, count, clear, clearAdd, conformToTemplate, renameField, removeCollection, merge), paging, transactions | `references/queries.md` |
| Search conditions, `EnumValueType` (incl. exact `decimal_` / `rational_`), sorting | `references/conditions_and_sort.md` |
| count / sum / min / max / mean (`aggregate` query) | `references/aggregate.md` |
| Cause / Actor / Permission, server-side execution, saving and loading, listeners | `references/metadata_and_server.md` |

## Core rules

1. **Build, then execute.** Builder constructors return a builder, not a query.
   Call `.build()` and pass the `Query` to `db.executeQuery(...)`.
2. **Choose the right builder.**
   `QueryBuilder` takes model objects (subclasses of `CloneableFile`) in `addData`.
   `RawQueryBuilder` takes plain maps in `rawAddData`. Everything else is identical.
3. **Check the result.** Queries do not throw for normal failures; they return a `QueryResult` with
   `isSuccess` and `errorMessage`. Always check it.
   Write queries (add, update, delete, ...) fail by default when they affect 0 items
   (`mustAffectAtLeastOne` is `true` by default).
4. **Use the right execute method.** `executeQuery` accepts only a `Query`.
   A `TransactionQuery` goes to `executeTransactionQuery`.
   `executeQueryObject` accepts a `Query`, a `TransactionQuery` or a JSON map,
   which is what a server should use for incoming requests.
5. **Nested fields use dot notation** (`"address.city"`) in nodes, sorts and aggregations.
   `renameField` only works on top-level fields.
6. **Store date-times as UTC ISO 8601 strings** and compare them with `EnumValueType.datetime_`
   (passing a `DateTime` as a node value sets it automatically). Aware (UTC) and naive (local) values are
   never compared with each other; pass `naiveUtcOffsetMinutes` (e.g. 540 for UTC+9) when stored values are naive.
7. **Store exact numbers such as prices as strings** (`"10.50"`) and compare / sort / aggregate them with
   `EnumValueType.decimal_` or `rational_`. These types convert only strings and
   `PortableDecimal` / `PortableRational` objects; `int` and `double` values are treated as conversion failures.
8. **The database is single-threaded and in memory.** Do not share one instance across isolates;
   run one instance per server, or load a per-user database on demand.

## Quick start

```dart
import 'package:delta_trace_db/delta_trace_db.dart';
import 'package:file_state_manager/file_state_manager.dart';

class User extends CloneableFile {
  final int id;
  final String name;
  final int age;

  User({required this.id, required this.name, required this.age});

  static User fromDict(Map<String, dynamic> src) =>
      User(id: src["id"], name: src["name"], age: src["age"]);

  @override
  Map<String, dynamic> toDict() => {"id": id, "name": name, "age": age};

  @override
  User clone() => User.fromDict(toDict());
}

final db = DeltaTraceDatabase();

final addResult = db.executeQuery(
  QueryBuilder.add(
    target: "users",
    addData: [User(id: -1, name: "Alice", age: 30), User(id: -1, name: "Bob", age: 25)],
    serialKey: "id", // overwrites "id" with 0, 1, 2, ...
  ).build(),
);
if (!addResult.isSuccess) print(addResult.errorMessage);

final r = db.executeQuery<User>(
  QueryBuilder.search(
    target: "users",
    queryNode: FieldGreaterThanOrEqual("age", 26),
    sortObj: SingleSort(field: "age", reversed: true),
    limit: 10,
  ).build(),
);
final List<User> users = r.convert(User.fromDict);
print(users.map((e) => e.name).toList()); // [Alice]
```

`CloneableFile` comes from the `file_state_manager` package. It is installed with DeltaTraceDB,
but add it to `pubspec.yaml` as a direct dependency before importing it.
The serial key field must already exist in each added item (any placeholder value works).
Without model classes, use `RawQueryBuilder` and plain maps instead.

## Result object

`QueryResult` fields:

- `isSuccess`: whether the query succeeded.
- `result`: `List<Map<String, dynamic>>`. Search results, returned data, or the aggregate result map.
- `dbLength`: number of items in the collection after the query.
- `updateCount`: number of items added, changed or deleted.
- `hitCount`: number of matched items (before paging for search).
- `errorMessage`: reason for failure, or null.
- `convert(fromDict)`: converts `result` to model objects.

A transaction returns `TransactionQueryResult` with `isSuccess`, `results` (one `QueryResult` per query)
and `errorMessage`.

## Things that look right but are wrong

- `RawQueryBuilder.add(target: ..., addData: [...])` — the raw builder's argument is `rawAddData`.
- `QueryBuilder.add(addData: [user.toDict()])` — `QueryBuilder` needs model objects; use `RawQueryBuilder` for maps.
- `db.executeQuery(TransactionQuery(...))` — use `executeTransactionQuery` or `executeQueryObject`.
- `FieldEquals("price", 10.5, vType: EnumValueType.decimal_)` — numbers are not converted by `decimal_`; pass `"10.5"`.
- Expecting `count` to accept a condition — `count` returns the collection size;
  count matching items with `aggregate` + `EnumAggregateType.count_`.
- Putting `removeCollection` or `merge` in a transaction — they are not allowed there.
