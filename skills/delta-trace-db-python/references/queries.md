# Queries (Python)

Every operation is built with `QueryBuilder` (model objects) or `RawQueryBuilder` (plain dicts),
converted with `.build()`, and executed. Pass builder arguments as keyword arguments.
The examples use `RawQueryBuilder`; `QueryBuilder` is identical except that it takes
`add_data` (a list of `CloneableFile` objects) instead of `raw_add_data` (a list of dicts).

Contents: [Common arguments](#common-arguments) · [add](#add) · [search / search_one / get_all](#search--search_one--get_all) ·
[Paging](#paging) · [update / update_one](#update--update_one) · [delete / delete_one](#delete--delete_one) ·
[count](#count) · [clear / clear_add](#clear--clear_add) · [add_csv / clear_add_csv](#add_csv--clear_add_csv) · [conform_to_template](#conform_to_template) ·
[rename_field](#rename_field) · [remove_collection](#remove_collection) · [Transactions](#transactions) · [merge](#merge)

## Common arguments

| Argument | Meaning |
|---|---|
| `target` | Collection name. A missing collection is created automatically. |
| `query_node` | Condition (see `conditions_and_sort.md`). |
| `sort_obj` | `SingleSort` or `MultiSort`. |
| `return_data` | Return the added / changed / deleted items in `result`. Default `False`. |
| `must_affect_at_least_one` | Fail when 0 items are affected. Default `True`. |
| `serial_key` | Field that receives an auto-increment integer (0, 1, 2, ... per collection). The field must exist in every added item. |
| `reset_serial` | Reset the serial counter to 0 (clear / clear_add). |
| `cause` | Audit metadata (see `metadata_and_server.md`). |

## add

```python
from delta_trace_db import *

db = DeltaTraceDatabase()
r = db.execute_query(RawQueryBuilder.add(
    target="users",
    raw_add_data=[
        {"id": -1, "name": "Alice", "age": 30, "address": {"city": "Tokyo"}},
        {"id": -1, "name": "Bob", "age": 25, "address": {"city": "Osaka"}},
        {"id": -1, "name": "Carol", "age": 35, "address": {"city": "Tokyo"}},
    ],
    serial_key="id",
    return_data=True,
).build())
print([e["id"] for e in r.result])  # [0, 1, 2]
```

## search / search_one / get_all

- `search`: all matching items. Arguments: `query_node` (required), `sort_obj`, `offset`, `start_after`, `end_before`, `limit`.
- `search_one`: the first matching item only (stops early). Arguments: `query_node`.
- `get_all`: every item, no condition. Arguments: `sort_obj`, `offset`, `start_after`, `end_before`, `limit`.

`hit_count` is the number of matches before paging and `limit` are applied.

```python
s = db.execute_query(RawQueryBuilder.search(
    target="users",
    query_node=FieldEquals("address.city", "Tokyo"),
    sort_obj=SingleSort(field="age"),
).build())
print([e["name"] for e in s.result])  # ['Alice', 'Carol']

one = db.execute_query(RawQueryBuilder.search_one(target="users", query_node=FieldEquals("id", 1)).build())
print(one.result[0]["name"])  # Bob

all_items = db.execute_query(RawQueryBuilder.get_all(target="users").build())
print(len(all_items.result))  # 3
```

## Paging

Use one of these together with `limit`:

- `offset`: skip that many items.
- `start_after`: pass the last item of the previous page; returns the items after it.
  Stable even if items are added meanwhile (it compares item values, so items must be distinguishable).
- `end_before`: returns up to `limit` items before the given item.

Builders also have `set_offset`, `set_start_after`, `set_end_before` and `set_limit` to change these before `build()`.

```python
page1 = db.execute_query(RawQueryBuilder.get_all(target="users", sort_obj=SingleSort(field="id"), limit=2).build())
page2 = db.execute_query(RawQueryBuilder.get_all(
    target="users", sort_obj=SingleSort(field="id"), start_after=page1.result[-1], limit=2).build())
print([e["id"] for e in page2.result])  # [2]
```

## update / update_one

`override_data` contains only the fields to change; other fields are kept.
`update` changes every match, `update_one` only the first.

```python
u = db.execute_query(RawQueryBuilder.update(
    target="users", query_node=FieldEquals("address.city", "Tokyo"), override_data={"tier": "gold"}).build())
print(u.update_count)  # 2
db.execute_query(RawQueryBuilder.update_one(
    target="users", query_node=FieldEquals("id", 1), override_data={"age": 26}).build())
```

## delete / delete_one

`delete` deletes every match, `delete_one` only the first.

```python
d = db.execute_query(RawQueryBuilder.delete(
    target="users", query_node=FieldLessThan("age", 30), return_data=True).build())
print([e["name"] for e in d.result])  # ['Bob']
```

## count

Returns the number of items in the collection in `hit_count` (and `db_length`).
It takes no condition; to count matching items use the `aggregate` query (`aggregate.md`).

```python
print(db.execute_query(RawQueryBuilder.count(target="users").build()).hit_count)  # 2
```

## clear / clear_add

`clear` empties the collection. `clear_add` empties it and adds new items in one step
(listeners are notified once). Both accept `reset_serial`.

```python
db.execute_query(RawQueryBuilder.clear_add(
    target="tags", raw_add_data=[{"id": -1, "label": "new"}], serial_key="id", reset_serial=True).build())
db.execute_query(RawQueryBuilder.clear(target="tags").build())
```

## add_csv / clear_add_csv

`add_csv` adds the rows of CSV text (like `add`), and `clear_add_csv` empties the collection first (like `clear_add`).
They accept `serial_key`, `return_data` and `must_affect_at_least_one` (and `reset_serial` for `clear_add_csv`).
The first record is the header, and each following row becomes an item of `{column name: cell text}`.
**All values are strings** — use `decimal_` (or another `v_type`) to compare, sort or aggregate them as numbers.
Quoted values (`"a, b"`, `""` for a quote, line breaks inside quotes), LF / CRLF / CR line ends, a leading BOM
and trailing empty lines are supported. An invalid CSV fails the query without changing anything
(no collection is created), and `error_message` shows the kind and line, e.g. `CsvFormatError(columnCount, line 3)`.
`UtilCsv.to_items(text)` checks and converts a CSV with the same rules and raises `CsvFormatError`
(`kind`: `EnumCsvErrorKind.header` / `columnCount` / `format`, and `line`). Requires DeltaTraceDB 0.4.0 or later.

```python
imported = db.execute_query(QueryBuilder.add_csv(
    target="items",
    csv_text='id,name,price\n,"Pen, blue",1.20\n,Notebook,3.50\n',
    serial_key="id",
    return_data=True,
).build())
print(imported.result[0])  # {'id': 0, 'name': 'Pen, blue', 'price': '1.20'}

bad = db.execute_query(QueryBuilder.add_csv(target="items", csv_text="id,name\n1").build())
print(bad.error_message)  # CsvFormatError(columnCount, line 2)
```

## conform_to_template

Changes the shape of every item: fields missing from `template` are deleted, and fields only in
`template` are added with the template value. Existing values of kept fields are not changed.
Back up the database first; this cannot be undone.

```python
db.execute_query(RawQueryBuilder.conform_to_template(
    target="users", template={"id": -1, "name": "", "age": 0, "email": ""}).build())
```

## rename_field

Renames a **top-level** field in every item that has it (nested fields cannot be renamed).

```python
db.execute_query(RawQueryBuilder.rename_field(target="users", rename_before="email", rename_after="mail").build())
```

## remove_collection

Deletes the collection itself. Listeners are not called, and it cannot be used in a transaction.

```python
db.execute_query(RawQueryBuilder.remove_collection(target="tags", must_affect_at_least_one=False).build())
```

## Transactions

`TransactionQuery` runs queries in order and reverts every involved collection if any query fails.
Listeners are notified only on success. `remove_collection` and `merge` are not allowed inside.
Execute it with `execute_transaction_query` (or `execute_query_object`); `execute_query` raises for it.

```python
tx = db.execute_transaction_query(TransactionQuery(queries=[
    RawQueryBuilder.add(target="orders", raw_add_data=[{"item": "pen", "qty": 1}]).build(),
    RawQueryBuilder.update_one(target="users", query_node=FieldEquals("id", 0), override_data={"age": 31}).build(),
]))
print(tx.is_success)  # True
print([e.update_count for e in tx.results])  # [1, 1]
```

## merge

Builds a new `output` collection by joining each `base` item with at most one item from each
`source` collection (`relation_key` of base == `source_keys[i]` of source i). Existing collections are
not modified. Intended for maintenance; not allowed in transactions.
`MergeQueryParams` takes keyword arguments only.

`dsl_tmp` defines each output field:

| Expression | Meaning |
|---|---|
| `base.field` | Field of the base item (dot notation allowed). |
| `0.field`, `1.field` | Field of the matched item of `source[0]`, `source[1]` (None if no match). |
| `int(1)`, `float(1.5)`, `bool(true)`, `str(text)` | Literal values (`bool(true)` is lowercase in the DSL). |
| `[0.email]` | Wrap one expression in a one-element list (only a single expression). |
| `popped.base[a,b]`, `popped.0[a]` | Copy of the item with the listed fields removed. |

`serial_base` inherits the serial counter of a collection; `serial_key` assigns new serial numbers in the output.
Permissions: search or searchOne on base / source / serial_base, and merge on output.

```python
db.execute_query(RawQueryBuilder.add(target="profiles", raw_add_data=[{"userId": 0, "bio": "hello"}]).build())
m = db.execute_query(RawQueryBuilder.merge(merge_query_params=MergeQueryParams(
    base="users",
    source=["profiles"],
    relation_key="id",
    source_keys=["userId"],
    output="userViews",
    dsl_tmp={"name": "base.name", "bio": "0.bio", "active": "bool(true)"},
)).build())
print(m.is_success)  # True
```
