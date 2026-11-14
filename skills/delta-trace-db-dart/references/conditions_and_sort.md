# Conditions and sorting (Dart)

## Comparison nodes

All nodes take the field name first. Nested fields use dot notation (`"address.city"`).

| Node | True when |
|---|---|
| `FieldEquals(field, value)` | field == value |
| `FieldNotEquals(field, value)` | field != value |
| `FieldGreaterThan(field, value)` | field > value |
| `FieldGreaterThanOrEqual(field, value)` | field >= value |
| `FieldLessThan(field, value)` | field < value |
| `FieldLessThanOrEqual(field, value)` | field <= value |
| `FieldContains(field, value)` | field (string or list) contains value |
| `FieldIn(field, values)` | field is one of values |
| `FieldNotIn(field, values)` | field is none of values |
| `FieldMatchesRegex(field, pattern)` | field matches the regular expression |
| `FieldStartsWith(field, value)` | field starts with value |
| `FieldEndsWith(field, value)` | field ends with value |

The six equality / relational nodes accept an optional named argument `vType` of type
`EnumValueType`. With any type other than `auto_`, both the field value and `value` are converted
before comparing, and a conversion failure makes the node false (it never throws).
`null` / missing fields are never greater or less than anything.

## Logical nodes

```dart
import 'package:delta_trace_db/delta_trace_db.dart';

final node = AndNode([
  FieldGreaterThanOrEqual("age", 20),
  OrNode([FieldEquals("city", "Tokyo"), FieldEquals("city", "Osaka")]),
  NotNode(FieldEquals("banned", true)),
]);
print(node.evaluate({"age": 30, "city": "Osaka", "banned": false})); // true
```

`node.evaluate(map)` is handy for testing conditions without a database.

## EnumValueType

| Value | Comparison |
|---|---|
| `auto_` (default) | Values as they are, without conversion. |
| `datetime_` | Both parsed as ISO 8601 and compared as date-times. Set automatically when the value is a `DateTime`. |
| `int_` | Both converted to integers. |
| `floatStrict_` | Both converted to doubles, compared exactly. |
| `floatEpsilon12_` | Doubles; equal if the difference is below 1e-12. |
| `boolean_` | Both compared as booleans (string form, case-insensitive). |
| `string_` | Both converted to strings. |
| `stringIgnoreCase_` | Strings, case-insensitive. |
| `decimal_` | Exact decimal comparison by value (`"1.50"` equals `"1.5"`). Set automatically when the value is a `PortableDecimal`. |
| `rational_` | Exact rational comparison; also accepts fractions such as `"1/3"`. Set automatically when the value is a `PortableRational`. |

### Exact numbers: decimal_ and rational_

Use these for money and other values where floating-point error is not acceptable.
They use the `simple_portable_numeric` package, which gives the same results as the Python version
of DeltaTraceDB. `PortableDecimal`, `PortableRational` and `EnumRoundingMode` are re-exported by
`package:delta_trace_db/delta_trace_db.dart`.

- Only **strings** (plain notation such as `"-12.340"`; no exponent, whitespace or `_`) and
  `PortableDecimal` / `PortableRational` objects are converted.
- `int` and `double` values are treated as conversion failures, both in the database and as the node
  value, because their exactness cannot be guaranteed (on the web, Dart ints are doubles).
  So store such values as strings, and write `"11"`, not `11`, in conditions.
- Queries using these types need DeltaTraceDB 0.2.0 or later to be read.

```dart
final price = FieldEquals("price", "1.5", vType: EnumValueType.decimal_);
print(price.evaluate({"price": "1.50"})); // true
print(price.evaluate({"price": 1.5})); // false
final tiny = FieldGreaterThan("price", PortableDecimal.parse("0.1"));
print(tiny.vType.name); // decimal_
print(tiny.evaluate({"price": "0.10000000000000000001"})); // true
print(FieldLessThan("r", "1/3", vType: EnumValueType.rational_).evaluate({"r": "0.333"})); // true
```

### Date-times

Store date-times as UTC ISO 8601 strings and use UTC `DateTime` values in conditions.
Comparing local and UTC times mixes up the results.

```dart
final after = FieldGreaterThan("createdAt", DateTime.utc(2025, 1, 1));
print(after.evaluate({"createdAt": "2025-06-01T00:00:00Z"})); // true
```

## Sorting

`SingleSort(field: ..., reversed: false, vType: EnumValueType.auto_)` sorts by one field.
`MultiSort([...])` applies several `SingleSort`s in order.

- `null` / missing values go last in ascending order and first in descending order.
- `vType` converts values before comparing, with the same values as above.
  With `decimal_` / `rational_`, values that cannot be converted are treated as null.
  For the other types, what happens on a conversion failure depends on the type
  (it may be treated as null or make the query fail), so store consistent types in a sorted field.
- `stringIgnoreCase_` gives case-insensitive alphabetical order.

```dart
final db = DeltaTraceDatabase();
db.executeQuery(
  RawQueryBuilder.add(
    target: "items",
    rawAddData: [
      {"name": "b", "price": "10.5"},
      {"name": "a", "price": "9.99"},
      {"name": "c", "price": "10.50"},
    ],
  ).build(),
);
final sorted = db.executeQuery(
  RawQueryBuilder.getAll(
    target: "items",
    sortObj: MultiSort([
      SingleSort(field: "price", vType: EnumValueType.decimal_),
      SingleSort(field: "name", reversed: true),
    ]),
  ).build(),
);
print(sorted.result.map((e) => e["name"]).toList()); // [a, c, b]
```
