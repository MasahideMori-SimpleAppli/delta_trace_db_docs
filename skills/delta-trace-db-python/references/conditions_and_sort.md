# Conditions and sorting (Python)

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
| `FieldContains(field, value)` | field (str or list) contains value |
| `FieldIn(field, values)` | field is one of values |
| `FieldNotIn(field, values)` | field is none of values |
| `FieldMatchesRegex(field, pattern)` | field matches the regular expression |
| `FieldStartsWith(field, value)` | field starts with value |
| `FieldEndsWith(field, value)` | field ends with value |

The six equality / relational nodes accept an optional `v_type` of type `EnumValueType`.
With any type other than `auto_`, both the field value and `value` are converted before comparing,
and a conversion failure makes the node `False` (it never raises).
`None` / missing fields are never greater or less than anything.

## Logical nodes

```python
from delta_trace_db import *

node = AndNode([
    FieldGreaterThanOrEqual("age", 20),
    OrNode([FieldEquals("city", "Tokyo"), FieldEquals("city", "Osaka")]),
    NotNode(FieldEquals("banned", True)),
])
print(node.evaluate({"age": 30, "city": "Osaka", "banned": False}))  # True
```

`node.evaluate(dict)` is handy for testing conditions without a database.

## EnumValueType

Member names are written exactly as below (camelCase with a trailing underscore).

| Value | Comparison |
|---|---|
| `auto_` (default) | Values as they are, without conversion. |
| `datetime_` | Both parsed as ISO 8601 and compared as datetimes. Set automatically when the value is a `datetime`. |
| `int_` | Both converted to int. |
| `floatStrict_` | Both converted to float, compared exactly. |
| `floatEpsilon12_` | Float; equal if the difference is below 1e-12. |
| `boolean_` | Both compared as booleans (string form, case-insensitive). |
| `string_` | Both converted to str. |
| `stringIgnoreCase_` | Strings, case-insensitive. |
| `decimal_` | Exact decimal comparison by value (`"1.50"` equals `"1.5"`). Set automatically when the value is a `PortableDecimal`. |
| `rational_` | Exact rational comparison; also accepts fractions such as `"1/3"`. Set automatically when the value is a `PortableRational`. |

### Exact numbers: decimal_ and rational_

Use these for money and other values where floating-point error is not acceptable.
They use the `simple-portable-numeric` package, which gives the same results as the Dart version
of DeltaTraceDB. `PortableDecimal`, `PortableRational` and `EnumRoundingMode` can be imported from `delta_trace_db`.

- Only **str** values (plain notation such as `"-12.340"`; no exponent, whitespace or `_`) and
  `PortableDecimal` / `PortableRational` objects are converted.
- `int`, `float` and `bool` values are treated as conversion failures, both in the database and as the
  node value, because their exactness cannot be guaranteed across systems.
  So store such values as strings, and write `"11"`, not `11`, in conditions.
- Queries using these types need DeltaTraceDB 0.2.0 or later to be read.

```python
price = FieldEquals("price", "1.5", v_type=EnumValueType.decimal_)
print(price.evaluate({"price": "1.50"}))  # True
print(price.evaluate({"price": 1.5}))  # False
tiny = FieldGreaterThan("price", PortableDecimal.parse("0.1"))
print(tiny.v_type.name)  # decimal_
print(tiny.evaluate({"price": "0.10000000000000000001"}))  # True
print(FieldLessThan("r", "1/3", v_type=EnumValueType.rational_).evaluate({"r": "0.333"}))  # True
```

### Datetimes

Store datetimes as UTC ISO 8601 strings and use timezone-aware `datetime` values in conditions.
Comparing naive (local) and aware (UTC) datetimes mixes up the results.

```python
from datetime import datetime, timezone

after = FieldGreaterThan("createdAt", datetime(2025, 1, 1, tzinfo=timezone.utc))
print(after.evaluate({"createdAt": "2025-06-01T00:00:00+00:00"}))  # True
```

## Sorting

`SingleSort(field, reversed_=False, v_type=EnumValueType.auto_)` sorts by one field
(note the trailing underscore of `reversed_`). `MultiSort([...])` applies several `SingleSort`s in order.

- `None` / missing values go last in ascending order and first in descending order.
- `v_type` converts values before comparing, with the same values as above.
  With `decimal_` / `rational_`, values that cannot be converted are treated as `None`.
  For the other types, a value that cannot be converted makes the query fail,
  so store consistent types in a sorted field.
- `stringIgnoreCase_` gives case-insensitive alphabetical order.

```python
db = DeltaTraceDatabase()
db.execute_query(RawQueryBuilder.add(target="items", raw_add_data=[
    {"name": "b", "price": "10.5"},
    {"name": "a", "price": "9.99"},
    {"name": "c", "price": "10.50"},
]).build())
sorted_r = db.execute_query(RawQueryBuilder.get_all(target="items", sort_obj=MultiSort([
    SingleSort(field="price", v_type=EnumValueType.decimal_),
    SingleSort(field="name", reversed_=True),
])).build())
print([e["name"] for e in sorted_r.result])  # ['a', 'c', 'b']
```
