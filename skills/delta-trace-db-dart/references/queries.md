# Queries (Dart)

Every operation is built with `QueryBuilder` (model objects) or `RawQueryBuilder` (plain maps),
converted with `.build()`, and executed. All builder arguments are named arguments.
The examples use `RawQueryBuilder`; `QueryBuilder` is identical except that it takes
`addData` (a `List<CloneableFile>`) instead of `rawAddData` (a `List<Map<String, dynamic>>`).

Contents: [Common arguments](#common-arguments) · [add](#add) · [search / searchOne / getAll](#search--searchone--getall) ·
[Paging](#paging) · [update / updateOne](#update--updateone) · [delete / deleteOne](#delete--deleteone) ·
[count](#count) · [clear / clearAdd](#clear--clearadd) · [conformToTemplate](#conformtotemplate) ·
[renameField](#renamefield) · [removeCollection](#removecollection) · [Transactions](#transactions) · [merge](#merge)

## Common arguments

| Argument | Meaning |
|---|---|
| `target` | Collection name. A missing collection is created automatically. |
| `queryNode` | Condition (see `conditions_and_sort.md`). |
| `sortObj` | `SingleSort` or `MultiSort`. |
| `returnData` | Return the added / changed / deleted items in `result`. Default `false`. |
| `mustAffectAtLeastOne` | Fail when 0 items are affected. Default `true`. |
| `serialKey` | Field that receives an auto-increment integer (0, 1, 2, ... per collection). The field must exist in every added item. |
| `resetSerial` | Reset the serial counter to 0 (clear / clearAdd). |
| `cause` | Audit metadata (see `metadata_and_server.md`). |

## add

```dart
import 'package:delta_trace_db/delta_trace_db.dart';

final db = DeltaTraceDatabase();
final r = db.executeQuery(
  RawQueryBuilder.add(
    target: "users",
    rawAddData: [
      {"id": -1, "name": "Alice", "age": 30, "address": {"city": "Tokyo"}},
      {"id": -1, "name": "Bob", "age": 25, "address": {"city": "Osaka"}},
      {"id": -1, "name": "Carol", "age": 35, "address": {"city": "Tokyo"}},
    ],
    serialKey: "id",
    returnData: true,
  ).build(),
);
print(r.result.map((e) => e["id"]).toList()); // [0, 1, 2]
```

## search / searchOne / getAll

- `search`: all matching items. Arguments: `queryNode` (required), `sortObj`, `offset`, `startAfter`, `endBefore`, `limit`.
- `searchOne`: the first matching item only (stops early). Arguments: `queryNode`.
- `getAll`: every item, no condition. Arguments: `sortObj`, `offset`, `startAfter`, `endBefore`, `limit`.

`hitCount` is the number of matches before paging and `limit` are applied.

```dart
final s = db.executeQuery(
  RawQueryBuilder.search(
    target: "users",
    queryNode: FieldEquals("address.city", "Tokyo"),
    sortObj: SingleSort(field: "age"),
  ).build(),
);
print(s.result.map((e) => e["name"]).toList()); // [Alice, Carol]

final one = db.executeQuery(
  RawQueryBuilder.searchOne(target: "users", queryNode: FieldEquals("id", 1)).build(),
);
print(one.result.first["name"]); // Bob

final all = db.executeQuery(RawQueryBuilder.getAll(target: "users").build());
print(all.result.length); // 3
```

## Paging

Use one of these together with `limit`:

- `offset`: skip that many items.
- `startAfter`: pass the last item of the previous page; returns the items after it.
  Stable even if items are added meanwhile (it compares item values, so items must be distinguishable).
- `endBefore`: returns up to `limit` items before the given item.

Builders also have `setOffset`, `setStartAfter`, `setEndBefore` and `setLimit` to change these before `build()`.

```dart
final page1 = db.executeQuery(
  RawQueryBuilder.getAll(target: "users", sortObj: SingleSort(field: "id"), limit: 2).build(),
);
final page2 = db.executeQuery(
  RawQueryBuilder.getAll(
    target: "users",
    sortObj: SingleSort(field: "id"),
    startAfter: page1.result.last,
    limit: 2,
  ).build(),
);
print(page2.result.map((e) => e["id"]).toList()); // [2]
```

## update / updateOne

`overrideData` contains only the fields to change; other fields are kept.
`update` changes every match, `updateOne` only the first.

```dart
final u = db.executeQuery(
  RawQueryBuilder.update(
    target: "users",
    queryNode: FieldEquals("address.city", "Tokyo"),
    overrideData: {"tier": "gold"},
  ).build(),
);
print(u.updateCount); // 2
db.executeQuery(
  RawQueryBuilder.updateOne(
    target: "users",
    queryNode: FieldEquals("id", 1),
    overrideData: {"age": 26},
  ).build(),
);
```

## delete / deleteOne

`delete` deletes every match, `deleteOne` only the first.

```dart
final d = db.executeQuery(
  RawQueryBuilder.delete(
    target: "users",
    queryNode: FieldLessThan("age", 30),
    returnData: true,
  ).build(),
);
print(d.result.map((e) => e["name"]).toList()); // [Bob]
```

## count

Returns the number of items in the collection in `hitCount` (and `dbLength`).
It takes no condition; to count matching items use the `aggregate` query (`aggregate.md`).

```dart
print(db.executeQuery(RawQueryBuilder.count(target: "users").build()).hitCount); // 2
```

## clear / clearAdd

`clear` empties the collection. `clearAdd` empties it and adds new items in one step,
which is convenient for replacing UI state (listeners are notified once).
Both accept `resetSerial`.

```dart
db.executeQuery(
  RawQueryBuilder.clearAdd(
    target: "tags",
    rawAddData: [
      {"id": -1, "label": "new"},
    ],
    serialKey: "id",
    resetSerial: true,
  ).build(),
);
db.executeQuery(RawQueryBuilder.clear(target: "tags").build());
```

## conformToTemplate

Changes the shape of every item: fields missing from `template` are deleted, and fields only in
`template` are added with the template value. Existing values of kept fields are not changed.
Back up the database first; this cannot be undone.

```dart
db.executeQuery(
  RawQueryBuilder.conformToTemplate(
    target: "users",
    template: {"id": -1, "name": "", "age": 0, "email": ""},
  ).build(),
);
```

## renameField

Renames a **top-level** field in every item that has it (nested fields cannot be renamed).

```dart
db.executeQuery(
  RawQueryBuilder.renameField(target: "users", renameBefore: "email", renameAfter: "mail").build(),
);
```

## removeCollection

Deletes the collection itself. Listeners are not called, and it cannot be used in a transaction.

```dart
db.executeQuery(RawQueryBuilder.removeCollection(target: "tags", mustAffectAtLeastOne: false).build());
```

## Transactions

`TransactionQuery` runs queries in order and reverts every involved collection if any query fails.
Listeners are notified only on success. `removeCollection` and `merge` are not allowed inside.
Execute it with `executeTransactionQuery` (or `executeQueryObject`), not `executeQuery`.

```dart
final tx = db.executeTransactionQuery(
  TransactionQuery(
    queries: [
      RawQueryBuilder.add(target: "orders", rawAddData: [{"item": "pen", "qty": 1}]).build(),
      RawQueryBuilder.updateOne(
        target: "users",
        queryNode: FieldEquals("id", 0),
        overrideData: {"age": 31},
      ).build(),
    ],
  ),
);
print(tx.isSuccess); // true
print(tx.results.map((e) => e.updateCount).toList()); // [1, 1]
```

## merge

Builds a new `output` collection by joining each `base` item with at most one item from each
`source` collection (`relationKey` of base == `sourceKeys[i]` of source i). Existing collections are
not modified. Intended for maintenance; not allowed in transactions.

`dslTmp` defines each output field:

| Expression | Meaning |
|---|---|
| `base.field` | Field of the base item (dot notation allowed). |
| `0.field`, `1.field` | Field of the matched item of `source[0]`, `source[1]` (null if no match). |
| `int(1)`, `float(1.5)`, `bool(true)`, `str(text)` | Literal values. |
| `[0.email]` | Wrap one expression in a one-element list (only a single expression). |
| `popped.base[a,b]`, `popped.0[a]` | Copy of the item with the listed fields removed. |

`serialBase` inherits the serial counter of a collection; `serialKey` assigns new serial numbers in the output.
Permissions: search or searchOne on base / source / serialBase, and merge on output.

```dart
db.executeQuery(
  RawQueryBuilder.add(target: "profiles", rawAddData: [{"userId": 0, "bio": "hello"}]).build(),
);
final m = db.executeQuery(
  RawQueryBuilder.merge(
    mergeQueryParams: MergeQueryParams(
      base: "users",
      source: ["profiles"],
      relationKey: "id",
      sourceKeys: ["userId"],
      output: "userViews",
      dslTmp: {"name": "base.name", "bio": "0.bio", "active": "bool(true)"},
    ),
  ).build(),
);
print(m.isSuccess); // true
```
