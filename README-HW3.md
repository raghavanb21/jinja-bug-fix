# HW3: Bug Fixing with Coding Agents

## Assignment overview

This assignment investigates and repairs a Jinja bug using repository
exploration, a failing regression test, symbol-tracking diagnostics, a focused
patch, and test-suite validation. The target is Jinja 3.1.6 and GitHub Issue
#2069, which concerns `meta.find_undeclared_variables()`.

## Repository and baseline

- Repository: [pallets/jinja](https://github.com/pallets/jinja)
- Assigned issue: [GitHub Issue #2069](https://github.com/pallets/jinja/issues/2069)
- Starting version: 3.1.6
- Original commit: `15206881c006c79667fe5154fe80c01c65410679`
- Original commit subject: `release version 3.1.6`
- Reproduction tag: `hw3-reproduction`

The local `hw3-reproduction` tag points to the original commit above. It does
not contain the assignment patch or the added tests.

## Reported bug

The issue reports that `meta.find_undeclared_variables()` includes a variable
that is assigned in every branch of an `if` statement. For example:

```jinja
{% if control == 'something' %}
    {% set output = 1 %}
{% elif control == 'something else' %}
    {% set output = 2 %}
{% else %}
    {% set output = 3 %}
{% endif %}
{{ output }}
```

Only `control` is an external input. Every runtime path assigns `output`
before the final use, so the expected result is `{'control'}`. On the original
3.1.6 commit, the result is `{'control', 'output'}`.

## Experimentally validated root cause

`meta.find_undeclared_variables()` uses `TrackingCodeGenerator`, which reports
the names whose frame-symbol load action is `resolve`. Conditional symbol
analysis occurs in `FrameSymbolVisitor.visit_If` and `Symbols.branch_update`
in `src/jinja2/idtracking.py`.

The original branch merge collected names stored by any branch and assigned
newly merged names the load action `('resolve', name)`. Thus `output` became
`('resolve', 'output')` even when every executable branch assigned it.
`TrackingCodeGenerator.enter_frame` then reported it as undeclared.

The investigation also established that normal code generation must retain its
context resolution in some cases: a variable can be read before it is assigned
later in all branches. Therefore, changing the ordinary compiler load action
would break rendering behavior.

## Implemented patch

The patch preserves normal compiler resolution behavior and adds metadata used
by meta introspection:

- `definitely_stored` records names assigned on every logical conditional path.
- `resolved_names` records names that required a direct context lookup before a
  local assignment established them.
- `visit_If` analyzes each `elif` as its own alternate path before merging
  metadata.
- Nested-frame references are propagated to the owning symbol frame, and meta
  collection runs after all frames have been analyzed. This retains real
  dependencies captured by a macro before a later assignment.
- `TrackingCodeGenerator` omits a `resolve` name only when it is definitely
  stored and was not directly resolved before assignment.

This fixes the issue's direct `if` / `elif` / `else` case while retaining
genuine undeclared-variable detection for incomplete branches and ordinary
early reads.

## Modified files

- `src/jinja2/idtracking.py`: records and merges definite-store and direct
  resolution metadata for conditional analysis.
- `src/jinja2/meta.py`: uses that metadata when collecting undeclared names.
- `tests/test_api.py`: contains the issue reproduction and expanded validation
  matrix.
- `README.md`: links to this assignment report. The original Jinja README is
  otherwise preserved.

## Environment setup with uv and Python 3.11

The validated local environment used Python 3.11.15. To create an equivalent
development environment with uv:

```sh
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install -e . -r requirements/tests.txt
```

## Reproduce the original failure

Use a separate worktree so the original tagged checkout is not mixed with the
patched working tree:

```sh
git worktree add ../jinja-hw3-reproduction hw3-reproduction
cd ../jinja-hw3-reproduction
uv venv --python 3.11 .venv
source .venv/bin/activate
uv pip install -e . -r requirements/tests.txt
```

Run the original issue scenario. The assertion is expected to fail on the
tagged baseline and display `{'control', 'output'}`:

```sh
.venv/bin/python - <<'PY'
from jinja2 import Environment, meta

source = """{% if control == 'something' %}
{% set output = 1 %}
{% elif control == 'something else' %}
{% set output = 2 %}
{% else %}
{% set output = 3 %}
{% endif %}
{{ output }}"""

result = meta.find_undeclared_variables(Environment().parse(source))
print(result)
assert result == {'control'}, result
PY
```

## Execute reproduction and validation tests

In the patched checkout, run the direct reproduction test:

```sh
.venv/bin/python -m pytest \
  tests/test_api.py::TestMeta::test_find_undeclared_variables_if_all_branches_assign -q
```

Run all meta validation tests, including exhaustive and incomplete branches,
nested conditionals, right-hand-side loads, macro parameters, and loop scope:

```sh
.venv/bin/python -m pytest tests/test_api.py::TestMeta -v
```

Run the existing rendering regression that constrains early reads before later
assignment:

```sh
.venv/bin/python -m pytest \
  tests/test_regression.py::test_load_parameter_when_set_in_all_if_branches -q
```

Run the complete regression suite:

```sh
.venv/bin/python -m pytest -q
```

## Actual test results

| State or command | Result |
| --- | --- |
| Original baseline from `evidence/01_baseline.txt` | `909 passed in 0.69s` |
| Original issue reproduction before the patch | Failed: expected `{'control'}`, got `{'control', 'output'}` |
| Patched original reproduction test | `3 passed in 0.01s` |
| Patched `TestMeta` suite | `19 passed in 0.02s` |
| Existing `tests/test_regression.py` suite | `54 passed in 0.04s` |
| Final complete suite | `925 passed in 0.68s` |

The 16-test increase from the original baseline is the assignment's added
reproduction and validation coverage. The final full-suite run had zero
failures, skips, and errors.

## Remaining limitations

The macro-ordering case identified during review is now covered by a regression
test. For example:

```jinja
{% macro render() %}{{ output }}{% endmacro %}
{{ render() }}
{% if control %}{% set output = 1 %}{% else %}{% set output = 2 %}{% endif %}
```

Both the original tagged Jinja 3.1.6 implementation and the refined patch
return `{'control', 'output'}` for this template. The patch keeps `output`
because the macro reads it before the later conditional assignment. The
refinement records references from nested frames on the frame that owns the
symbol, then waits until all frames have been analyzed before collecting meta
results. This preserves the dependency without adding a macro-specific rule;
the same mechanism applies to any nested scope that captures an outer name.

No remaining limitation was found in the tested ordinary Jinja syntax. Custom
extensions that construct unusual AST structures are outside this assignment's
coverage and would require separate compatibility testing.
