# Experiment Profile Syntax and Semantics

This document describes the YAML syntax for Pioreactor experiment profiles using the current action schema.

## Profile Structure

An experiment profile is a YAML document with these top-level fields:
- `experiment_profile_name` (required): name for the profile.
- `metadata` (optional): `author`, `description`.
- `plugins` (optional): list of `{name, version}` requirements.
- `inputs` (optional): map of values available to expressions.
- `common` (optional): jobs shared by all pioreactors.
- `pioreactors` (optional): per-unit definitions with optional `label` and `jobs`.

### Skeleton

```yaml
experiment_profile_name: <string>

metadata:
  author: <string>
  description: <string>

plugins:
  - name: <string>
    version: <string>

inputs:
  <input_name>: <value>

common:
  jobs:
    <job_name>:
      description: <string>
      actions:
        - # action entries (see below)

pioreactors:
  <pioreactor_unit_name>:
    label: <string>
    jobs:
      <job_name>:
        description: <string>
        actions:
          - # action entries (see below)
```

## Actions

Actions live inside `actions` arrays and share this shape:

```yaml
- type: <action_type>
  t: <time_string_or_float>
  if: <bool_or_expression>          # optional guard
  options: {<option_name>: <value>} # optional, expressions allowed via ${{ }}
  args: [<string>, ...]             # start only
  config_overrides: {<config_name>: <value>} # start only
```

### Basic actions
- `log`: `t`, optional `if`, `options.message`, optional `options.level` (`DEBUG|INFO|NOTICE|WARNING|ERROR`, case-insensitive, default `notice`).
- `start`: `t`, optional `if`, optional `options`, `args`, `config_overrides`.
- `update`: `t`, optional `if`, optional `options`.
- `pause`, `resume`, `stop`: `t`, optional `if`.

### Container actions
- `repeat`: `t`, optional `if`, `every` (time between iterations), optional `while` stop condition, optional `max_time`, and `actions` containing basic actions. Nested action `t` values are relative to the start of each loop iteration.
- `when`: `t`, optional `if`, `wait_until` expression, and `actions` (basic or container). Nested action `t` values are relative to when the condition is satisfied.

## Expression Syntax

Expressions are used in `if`, `while`, `wait_until`, and inside `${{ ... }}` in option values.
- Literals: integers, floats, and booleans (`true` / `false`, case-insensitive).
- Operators: `+`, `-`, `*`, `/`, `**`, comparisons (`<`, `<=`, `==`, `>=`, `>`), boolean `and` / `or` / `not`, parentheses for grouping.
- Functions: `random()`, `unit()`, `hours_elapsed()`, `experiment()`, `job_name()`.
- MQTT lookups: `<unit>:<job>:<setting>[.<nested_key>]*` or `::<job>:<setting>[.<nested_key>]*` (uses current unit).
- Names resolve to values provided in the expression environment (inputs, etc.); otherwise they remain strings.
- Conversion rules: numeric strings become floats; `"true"`/`"false"` become booleans; other strings stay strings.

## Time Strings

`t` and other durations accept either a number (hours) or a numeric string with a unit suffix: `s`, `m`, `h`, or `d` (case-insensitive). Examples: `0.5` (30 minutes), `"30s"`, `"2m"`, `"1.5h"`, `"2d"`. Whitespace, extra text, or negative values are rejected.
