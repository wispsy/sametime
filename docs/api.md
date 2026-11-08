# sametime API Reference

`sametime` provides bounded synchronous thread concurrency for independent I/O operations, built strictly on Python's standard library (`concurrent.futures.ThreadPoolExecutor`). Requires Python 3.14+.

---

## Design Principles

1. **Explicit Bounds**: `max_workers` controls active threads (defaults to CPU count + 4, capped at 32). `max_pending` bounds the maximum number of submitted, unconsumed futures (defaults to `max_workers * 2`).
2. **Context Manager Lifetime**: `thread_iter` and `thread_outcomes` are context managers. Exiting the `with` block cancels unstarted futures and joins active threads immediately.
3. **Consumer-Thread Callbacks**: `on_result` callbacks execute strictly on the consuming thread as results are ready or consumed, avoiding threading races in user state.
4. **Clean Error Handling**:
   - `thread_map` / `thread_iter`: Fail fast, re-raising the first exception encountered.
   - `thread_outcomes`: Captures worker exceptions inside `Outcome` records while letting keyboard interrupts and consumer errors propagate.

---

## API Reference

### `Outcome[R]`

An indexed execution result:

```python
@dataclass(frozen=True, slots=True)
class Outcome[R]:
    index: int
    value: R | None = None
    error: Exception | None = field(default=None, repr=False)
```

- `index: int`: The original 0-based position of the item in the input iterable.
- `ok: bool`: True if the call succeeded without exception.
- `result() -> R`: Returns `value` (including `None`), or re-raises `error`.

---

### `thread_map(function, iterable, *, max_workers=None, max_pending=None, ordered=True, on_result=None) -> list[R]`

Runs `function` concurrently over `iterable` using a bounded thread pool, collecting all results into a list.
- If `ordered=True` (default), preserves the input sequence order in the returned list.
- Fails fast on the first exception.

---

### `thread_iter(function, iterable, *, max_workers=None, max_pending=None, ordered=True, on_result=None) -> Iterator[Iterator[R]]`

Context manager yielding an iterator of results.
- `ordered=True` yields results in input order.
- `ordered=False` yields whichever result finishes first.
- Bounded input consumption: does not pull infinitely from lazy generators before downstream processing consumes results.

```python
with thread_iter(fetch_data, urls, max_workers=8, ordered=False) as results:
    for data in results:
        process(data)
```

---

### `thread_outcomes(function, iterable, *, max_workers=None, max_pending=None, ordered=True, on_result=None) -> Iterator[Iterator[Outcome[R]]]`

Context manager yielding `Outcome` objects for both successes and worker failures without interrupting processing of remaining items.

```python
with thread_outcomes(process_record, records, max_pending=16) as outcomes:
    for outcome in outcomes:
        if outcome.ok:
            print(f"Record {outcome.index} succeeded: {outcome.value}")
        else:
            print(f"Record {outcome.index} failed: {outcome.error}")
```
