# Aggregate query (Dart)

`aggregate` calculates several aggregations over the items that match an optional condition and
returns them as **one map** in `result.first` (one map per group with `groupBy`, see
[Joins and groups](#joins-and-groups)). `hitCount` is the number of matched items.
It is read-only, can be used in transactions, and needs its own `EnumQueryType.aggregate` permission
on servers (a `search` permission does not allow it). Requires DeltaTraceDB 0.2.0 or later.

**Security:** treat the `aggregate` permission as equivalent to read (`search`) access.
No items are returned, but the contents can still be discovered — for example, by repeating `count_`
with `FieldStartsWith` while extending the prefix one character at a time — and `min_` / `max_`
return field values directly. Grant it only to users who may read the target collection.

```
QueryBuilder.aggregate({required target, required aggregations, queryNode, joins, groupBy, sortObj})   // RawQueryBuilder is the same
```

## Aggregation

`Aggregation(type, {field, fields, vType = EnumValueType.floatStrict_, alias, scale, rounding})`

| Argument | Meaning |
|---|---|
| `type` (positional) | `EnumAggregateType.count_`, `sum_`, `min_`, `max_` or `mean_`. |
| `field` | Target field (dot notation allowed). This or `fields` is required except for `count_`. |
| `fields` | Instead of `field`: aggregate the **product** of these fields, e.g. `["quantity", "unitPrice"]` for sales. Each value is converted with `vType`; an item with a null value in any of them is skipped. 0.4.0 or later. |
| `vType` | Calculation type: `floatStrict_` (default), `decimal_` or `rational_`. Other types make the query fail. Ignored by `count_`. |
| `alias` | Key in the result map. Default: type name + field, e.g. `"sum_price"` (`fields` are joined with `*`, e.g. `"sum_quantity*unitPrice"`); `"count_"` for `count_` without a field. Duplicate keys make the query fail. |
| `scale`, `rounding` | Only for `mean_` with `decimal_`, and required there: digits after the decimal point and `EnumRoundingMode` (`halfEven`, `halfUp`, `down`, `floor`, `ceiling`, `up`; `up` rounds away from zero and needs simple_portable_numeric 0.1.0 or later). |

Rules:

- `count_` without a field counts matched items; with a field it counts items whose value is not null.
  It never converts values, so it works for any type.
- `null` / missing values are skipped by every aggregation.
- A value that cannot be converted makes the whole query fail (`isSuccess` false, reason in `errorMessage`),
  so a wrong total is never returned silently.
- With no values: `count_` is 0, `sum_` is 0 (`0.0` or `"0"`), and `min_` / `max_` / `mean_` are null.

| vType | Accepts | Result |
|---|---|---|
| `floatStrict_` | Numbers, and plain number strings such as `"1.5"`, `"1e3"`, `".5"` (not `"inf"`, `"NaN"`, `"1_000"`, whitespace) | `double` |
| `decimal_` | Strings and `PortableDecimal` (not numbers) | `String`, e.g. `"43.80"` (sum keeps the largest scale; min / max keep the original string form) |
| `rational_` | Strings (including `"1/3"`), `PortableDecimal`, `PortableRational` (not numbers) | `String`, e.g. `"73/5"` (mean is exact) |

`floatStrict_` adds values left to right in the stored order, so the Python version of DeltaTraceDB
returns the same double for the same data.

## Joins and groups

Requires DeltaTraceDB 0.4.0 or later. Processing order: **`joins` → `queryNode` → `groupBy` → aggregations → `sortObj`**.

**AggregateJoin** `AggregateJoin(collection:, localKey:, foreignKey:, alias:)` attaches the item of `collection`
whose `foreignKey` equals the item's `localKey`, under `alias`, so its fields are `alias.field` in the
condition, the group keys and the aggregations. Joins are applied in order, so a later `localKey` may use an
earlier alias. **Exactly one item must match**: a missing joined collection, a null `localKey`, no match, or more
than one match (a duplicate would double-count) makes the query fail — for every item, even ones the condition
excludes later. Values are compared by their JSON form (`1` and `"1"` do not match). The `alias` must not be
empty, contain a dot, or equal a field of the item. On servers, each joined collection also needs the
`search` or `searchOne` permission.

**AggregateGroupKey** `AggregateGroupKey(field, {unit, utcOffsetMinutes, weekStart, startMonth, alias})` groups by the field value (null forms its own group). With `unit`
(`EnumAggregateGroupUnit.day` / `week` / `month` / `quarter` / `year`) the value is read as an ISO 8601
date-time and grouped by period, in the time zone `utcOffsetMinutes` (default 0 = UTC; values without a time zone
are read in it). `weekStart` (week only) is the ISO weekday of the first day (1 = Monday ... 7 = Sunday, default 1).
`startMonth` (quarter / year only) is the first month of the year (e.g. 4 for an April fiscal year, default 1).

| unit | Label |
|---|---|
| `day` | `2026-10-05` |
| `week` | first day of the week, `2026-10-05` |
| `month` | `2026-10` |
| `quarter` | `2026-04/2026-06` |
| `year` | `2026-01/2026-12`, or `2026-04/2027-03` with `startMonth` 4 |

Each result map holds the group key values (under `alias`, default `field`) and then the aggregations.
Groups are in first-appearance order, or sorted by `sortObj`. Sort by output keys without dots, so give a dotted
field such as `o.orderedAt` an `alias`. Output keys of group keys and aggregations must be unique.
A product with no lines in a period does not appear in that period.

## Example

```dart
import 'package:delta_trace_db/delta_trace_db.dart';

final db = DeltaTraceDatabase();
db.executeQuery(
  RawQueryBuilder.add(
    target: "orders",
    rawAddData: [
      {"status": "paid", "price": "10.50", "qty": 2},
      {"status": "paid", "price": "3.3", "qty": 1.5},
      {"status": "paid", "price": "30", "qty": 4},
      {"status": "cancel", "price": "100", "qty": 10},
    ],
  ).build(),
);
final r = db.executeQuery(
  RawQueryBuilder.aggregate(
    target: "orders",
    queryNode: FieldEquals("status", "paid"),
    aggregations: [
      Aggregation(EnumAggregateType.count_, alias: "n"),
      Aggregation(EnumAggregateType.sum_, field: "price", vType: EnumValueType.decimal_, alias: "total"),
      Aggregation(
        EnumAggregateType.mean_,
        field: "price",
        vType: EnumValueType.decimal_,
        scale: 2,
        rounding: EnumRoundingMode.halfEven,
        alias: "avg",
      ),
      Aggregation(EnumAggregateType.max_, field: "price", vType: EnumValueType.decimal_),
      Aggregation(EnumAggregateType.sum_, field: "qty"),
    ],
  ).build(),
);
print(r.isSuccess); // true
print(r.result.first); // {n: 3, total: 43.80, avg: 14.60, max_price: 30, sum_qty: 7.5}
```

Failure example (numbers are not converted by `decimal_`):

```dart
final bad = db.executeQuery(
  RawQueryBuilder.aggregate(
    target: "orders",
    aggregations: [Aggregation(EnumAggregateType.sum_, field: "qty", vType: EnumValueType.decimal_)],
  ).build(),
);
print(bad.isSuccess); // false
print(bad.errorMessage); // Cannot convert the value of field 'qty' to decimal_.
```

Monthly sales and units per product, from tables imported with `addCsv` (all values are strings):

```dart
final shop = DeltaTraceDatabase();
for (final (name, csv) in [
  ("orders", "orderId,orderedAt\no1,2026-10-01T10:00:00+09:00\no2,2026-11-02T09:00:00+09:00\n"),
  ("order_items", "orderId,productId,quantity,unitPrice\no1,P1,2,100.50\no2,P1,1,100.50\no2,P2,3,0\n"),
  ("products", "id,name,unitsPerSet\nP1,apple,6\nP2,orange,10\n"),
]) {
  shop.executeQuery(QueryBuilder.addCsv(target: name, csvText: csv).build());
}
final monthly = shop.executeQuery(
  QueryBuilder.aggregate(
    target: "order_items",
    joins: [
      AggregateJoin(collection: "orders", localKey: "orderId", foreignKey: "orderId", alias: "o"),
      AggregateJoin(collection: "products", localKey: "productId", foreignKey: "id", alias: "p"),
    ],
    groupBy: [
      AggregateGroupKey("o.orderedAt", unit: EnumAggregateGroupUnit.month, utcOffsetMinutes: 540, alias: "month"),
      AggregateGroupKey("productId"),
    ],
    aggregations: [
      Aggregation(EnumAggregateType.sum_, fields: ["quantity", "unitPrice"], vType: EnumValueType.decimal_, alias: "sales"),
      Aggregation(EnumAggregateType.sum_, fields: ["quantity", "p.unitsPerSet"], vType: EnumValueType.decimal_, alias: "units"),
    ],
    sortObj: MultiSort([SingleSort(field: "month"), SingleSort(field: "productId")]),
  ).build(),
);
print(monthly.result.length); // 3
print(monthly.result.first); // {month: 2026-10, productId: P1, sales: 201.00, units: 12}
```
