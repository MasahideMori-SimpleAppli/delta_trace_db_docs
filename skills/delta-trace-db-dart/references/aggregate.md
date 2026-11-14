# Aggregate query (Dart)

`aggregate` calculates several aggregations over the items that match an optional condition and
returns them as **one map** in `result.first`. `hitCount` is the number of matched items.
It is read-only, can be used in transactions, and needs its own `EnumQueryType.aggregate` permission
on servers (a `search` permission does not allow it). Requires DeltaTraceDB 0.2.0 or later.

```
QueryBuilder.aggregate({required target, required aggregations, queryNode})   // RawQueryBuilder is the same
```

## Aggregation

`Aggregation(type, {field, vType = EnumValueType.floatStrict_, alias, scale, rounding})`

| Argument | Meaning |
|---|---|
| `type` (positional) | `EnumAggregateType.count_`, `sum_`, `min_`, `max_` or `mean_`. |
| `field` | Target field (dot notation allowed). Required except for `count_`. |
| `vType` | Calculation type: `floatStrict_` (default), `decimal_` or `rational_`. Other types make the query fail. Ignored by `count_`. |
| `alias` | Key in the result map. Default: type name + field, e.g. `"sum_price"`; `"count_"` for `count_` without a field. Duplicate keys make the query fail. |
| `scale`, `rounding` | Only for `mean_` with `decimal_`, and required there: digits after the decimal point and `EnumRoundingMode` (`halfEven`, `halfUp`, `down`, `floor`, `ceiling`). |

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
print(bad.errorMessage); // Cannot convert the value of field 'qty' to decimal_: 2
```
