---
name: solara-state-management
description: "Patterns for reactive state in Solara: reactive variables, hooks, cross-component sharing, and scoping"
version: 1.0.0
---

# Solara State Management

Actionable guidance for reactive state in Solara: reactive variables, hooks, cross-component sharing, and scoping.

---

## Reactive Variables (Module-Level State)

- `solara.reactive(initial_value)` creates a reactive variable at module level
- Access value with `.value`, set with `.value = new_val`
- Changes trigger re-renders in all components that read the value
- This is the primary way to share state across components

```python
import solara

counter = solara.reactive(0)

@solara.component
def IncrementButton():
    solara.Button(f"Count: {counter.value}", on_click=lambda: setattr(counter, 'value', counter.value + 1))

@solara.component
def CounterDisplay():
    solara.Text(f"Current count: {counter.value}")
```

---

## Component-Local State with use_state

- `value, set_value = solara.use_state(initial)` — local to the component instance
- Calling `set_value` triggers only this component to re-render
- Use for UI-local state (form inputs, toggle states, selected tabs)
- Initial value is only used on first render

```python
@solara.component
def ToggleExample():
    is_on, set_is_on = solara.use_state(False)
    solara.Switch(value=is_on, on_value=set_is_on)
    solara.Text(f"State: {'On' if is_on else 'Off'}")
```

---

## use_reactive Hook

- `state = solara.use_reactive(initial_or_reactive)` — can accept either an initial value OR a reactive variable
- Returns a reactive-like object with `.value`
- Useful when a component should work with both passed-in reactive vars and standalone

```python
@solara.component
def FlexibleCounter(initial_count: int = 0):
    state = solara.use_reactive(initial_count)
    solara.Button(f"Count: {state.value}", on_click=lambda: setattr(state, 'value', state.value + 1))
```

---

## Computed/Derived State

- `solara.use_memo(fn, dependencies)` — recompute only when dependencies change
- Avoid recomputing expensive derived data on every render

```python
import pandas as pd

@solara.component
def FilteredView(df: pd.DataFrame, search: str):
    filtered = solara.use_memo(lambda: df[df["name"].str.contains(search, case=False)], [df, search])
    solara.DataFrame(filtered)
```

---

## Side Effects

- `solara.use_effect(fn, dependencies)` — run after render when deps change
- Return a cleanup function for teardown
- Pass `[]` as dependencies to run only on mount

```python
@solara.component
def Timer():
    elapsed, set_elapsed = solara.use_state(0)

    def start_timer():
        import threading
        stop = threading.Event()

        def tick():
            while not stop.is_set():
                stop.wait(1)
                set_elapsed(lambda e: e + 1)

        t = threading.Thread(target=tick, daemon=True)
        t.start()
        return stop.set  # cleanup

    solara.use_effect(start_timer, [])
    solara.Text(f"Elapsed: {elapsed}s")
```

---

## Async Tasks with use_task

- `solara.lab.use_task(async_fn, dependencies)` — manages async operations
- Returns a `Result` object with `.value`, `.error`, `.pending`, `.latest`
- Automatically cancels previous invocation when deps change

```python
@solara.component
def DataLoader(url: str):
    result = solara.lab.use_task(lambda: fetch_data(url), dependencies=[url])
    if result.pending:
        solara.SpinnerSolara()
    elif result.error:
        solara.Error(f"Failed: {result.error}")
    else:
        solara.DataFrame(result.value)
```

---

## State Scoping

- **Module-level `solara.reactive()`** — shared across ALL users/sessions (use for app config, not user data)
- To create per-user/per-session state, use `solara.reactive()` inside `on_kernel_start`:

```python
user_data = solara.reactive(None)

def initialize():
    user_data.value = load_user_defaults()

solara.on_kernel_start(initialize)
```

- **`solara.use_state()`** — scoped to a single component instance in a single session

---

## Cross-Filter State

- `solara.use_cross_filter(id, filter_type)` for coordinated filtering across multiple visualization components
- `solara.CrossFilterDataFrame`, `solara.CrossFilterSelect`, `solara.CrossFilterSlider`

```python
@solara.component
def CoordinatedFilters():
    df = solara.reactive(pd.DataFrame(...))
    solara.CrossFilterSelect(df.value, "category", label="Category")
    solara.CrossFilterSlider(df.value, "amount", label="Amount")
    solara.CrossFilterDataFrame(df.value)
```

---

## Best Practices

| Situation | Use |
|-----------|-----|
| State shared between components | `solara.reactive()` |
| Purely local UI state | `use_state()` |
| Expensive derived data | `use_memo()` |
| IO/async work | `use_task()` — handles loading/error states |

- Never mutate state directly — always assign a new value to `.value` or call the setter
- Keep state as close to where it's used as possible
