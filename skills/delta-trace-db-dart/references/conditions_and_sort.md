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
Numeric strings follow `int.parse` / `double.parse` (e.g. `int_` accepts `"0x10"`; `"1_000"` and `"inf"`
are not converted), and `boolean_` accepts only `"true"` / `"false"`.
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
  For a value in exponent notation, such as `"1.2E-05"` in a CSV file exported from spreadsheet software,
  convert it with `PortableDecimal.parseLenient` and store its `toString()`
  (needs simple_portable_numeric 0.1.0 or later).
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

In a data class, keep the value as `PortableDecimal` and save it as a `String` in `toDict`.
Never write a double into `toDict` for a field compared with `decimal_`: the item silently stops matching.

```dart
import 'package:file_state_manager/file_state_manager.dart';

class Product extends CloneableFile {
  final String name;
  final PortableDecimal price;

  Product({required this.name, required this.price});

  static Product fromDict(Map<String, dynamic> src) =>
      Product(name: src["name"], price: PortableDecimal.parse(src["price"]));

  @override
  Map<String, dynamic> toDict() => {"name": name, "price": price.toString()}; // always a String

  @override
  Product clone() => Product.fromDict(toDict());
}

final pdb = DeltaTraceDatabase();
pdb.executeQuery(
  QueryBuilder.add(
    target: "products",
    addData: [
      Product(name: "pen", price: PortableDecimal.parse("1.50")),
      Product(name: "book", price: PortableDecimal.parse("12.00")),
    ],
  ).build(),
);
final cheap = pdb.executeQuery<Product>(
  QueryBuilder.search(target: "products", queryNode: FieldLessThan("price", PortableDecimal.parse("10"))).build(),
);
print(cheap.convert(Product.fromDict).map((e) => "${e.name}: ${e.price}").toList()); // [pen: 1.50]
```

### Date-times

Store date-times as UTC ISO 8601 strings and use UTC `DateTime` values in conditions.

- Only valid ISO 8601 date-times are converted (e.g. `2025-01-01T09:00:00+09:00`, `2025-01-01T00:00:00Z`,
  `2025-01-01`). Month 13, hour 24, a lowercase `z` and week dates are conversion failures.
- An aware value (`Z` or an offset; a UTC `DateTime`) and a naive value (no offset; a local `DateTime`)
  are never compared: the node is false (even `FieldNotEquals`), so such items are not returned.
- If stored values are naive, pass `naiveUtcOffsetMinutes` (e.g. `540` for UTC+9): naive values are then
  read as times at UTC + that many minutes, regardless of the machine's timezone.

```dart
final after = FieldGreaterThan("createdAt", DateTime.utc(2025, 1, 1));
print(after.evaluate({"createdAt": "2025-06-01T00:00:00Z"})); // true
print(after.evaluate({"createdAt": "2025-06-01T00:00:00"})); // false (naive vs aware)
final jst = FieldGreaterThan(
  "createdAt",
  DateTime.utc(2025, 1, 1),
  vType: EnumValueType.datetime_,
  naiveUtcOffsetMinutes: 540,
);
print(jst.evaluate({"createdAt": "2025-06-01T00:00:00"})); // true
```

## Sorting

`SingleSort(field: ..., reversed: false, vType: EnumValueType.auto_)` sorts by one field.
`MultiSort([...])` applies several `SingleSort`s in order.

- `null` / missing values go last in ascending order and first in descending order.
- `vType` converts values before comparing, with the same values as above.
  A value that cannot be converted is treated as null (except with `auto_`, which compares values as they
  are and fails the query if they cannot be compared).
- With `datetime_`, a naive value compared with an aware value is treated as null, so aware values come first.
  Pass `naiveUtcOffsetMinutes` to `SingleSort` to sort them together.
- In a `MultiSort`, these rules apply to each `SingleSort`, and `naiveUtcOffsetMinutes` is set on each `SingleSort`.
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
