# Naming (Python)

Method and argument names are snake_case. A few names differ from that rule, and the JSON produced by
`to_dict()` uses camelCase keys (so the same JSON works with the Dart version).

## Exception 1: names that clash with Python built-ins or keywords get a trailing `_`

| Argument | Where | JSON key |
|---|---|---|
| `type_` | `Query`, `QueryBuilder`, `RawQueryBuilder`, `QueryResult` constructors, `Aggregation` | `"type"` |
| `from_` | `Cause` | `"from"` |
| `reversed_` | `SingleSort` | `"reversed"` |

The attribute of a constructed object is still `type` (e.g. `result.type`, `aggregation.type`).

## Exception 2: Actor uses an `actor_` prefix

`Actor(actor_type, actor_id, ...)`. The JSON keys are `"type"` and `"id"`.

## Exception 3: enum members are not snake_case

Enum members keep their original names, including camelCase and trailing underscores:

- `EnumQueryType`: `add`, `update`, `updateOne`, `delete`, `deleteOne`, `search`, `searchOne`, `getAll`,
  `conformToTemplate`, `renameField`, `count`, `clear`, `clearAdd`, `removeCollection`, `merge`, `aggregate`,
  `addCsv`, `clearAddCsv`
- `EnumValueType`: `auto_`, `datetime_`, `int_`, `floatStrict_`, `floatEpsilon12_`, `boolean_`, `string_`,
  `stringIgnoreCase_`, `decimal_`, `rational_`
- `EnumAggregateType`: `count_`, `sum_`, `min_`, `max_`, `mean_`
- `EnumAggregateGroupUnit`: `day`, `week`, `month`, `quarter`, `year`
- `EnumCsvErrorKind`: `header`, `columnCount`, `format`
- `EnumRoundingMode`: `halfEven`, `halfUp`, `down`, `floor`, `ceiling`
- `EnumActorType`: `human`, `ai`, `system`

## Exception 4: listener callbacks are named `cb`

`db.add_listener(target, cb, name=None)` / `db.remove_listener(target, cb, name=None)`.
The callback takes no arguments.

## Argument names and their JSON keys

When reading or writing query JSON (for example, logs or requests from a Flutter front end),
the keys are the camelCase form of the Python names:

| Python | JSON key |
|---|---|
| `add_data` / `raw_add_data` | `addData` |
| `query_node` | `queryNode` |
| `override_data` | `overrideData` |
| `sort_obj` | `sortObj` |
| `start_after` / `end_before` | `startAfter` / `endBefore` |
| `return_data` | `returnData` |
| `must_affect_at_least_one` | `mustAffectAtLeastOne` |
| `serial_key` / `reset_serial` | `serialKey` / `resetSerial` |
| `rename_before` / `rename_after` | `renameBefore` / `renameAfter` |
| `merge_query_params` | `mergeQueryParams` |
| `relation_key` / `source_keys` / `dsl_tmp` / `serial_base` | `relationKey` / `sourceKeys` / `dslTmp` / `serialBase` |
| `v_type` | `vType` |
| `csv_text` | `csvText` |
| `group_by` | `groupBy` |
| `local_key` / `foreign_key` | `localKey` / `foreignKey` |
| `utc_offset_minutes` / `week_start` / `start_month` | `utcOffsetMinutes` / `weekStart` / `startMonth` |
| `naive_utc_offset_minutes` | `naiveUtcOffsetMinutes` |
| `is_success`, `db_length`, `update_count`, `hit_count`, `error_message` | `isSuccess`, `dbLength`, `updateCount`, `hitCount`, `errorMessage` |
| `chain_parent_serial`, `confidence_score` | `chainParentSerial`, `confidenceScore` |
