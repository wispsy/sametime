# sametime

Bounded synchronous thread concurrency for independent I/O operations.

`sametime` provides structured thread management built directly on Python's standard library `concurrent.futures`. Requires Python 3.14+.

## Highlights

- **Strict Resource Bounds**: Controls both worker count (`max_workers`) and queued futures (`max_pending`), preventing memory ballooning on large/infinite input iterables.
- **Context-Managed Lifecycle**: Automatic cancellation of pending tasks and graceful thread joins on context exit.
- **Ordered & Unordered Execution**: Preserve input order or stream results as soon as they complete (`ordered=False`).
- **Flexible Failure Strategies**: Choose between fail-fast behavior (`thread_map`, `thread_iter`) or indexed outcomes with error capture (`thread_outcomes`).
- **Consumer-Thread Callbacks**: `on_result` callbacks execute safely on the caller's thread before results are yielded.

## Installation

```sh
pip install sametime
```

For development:

```sh
git clone https://github.com/wispsy/sametime.git
cd sametime
uv sync
```

## Quick Start

### Basic Parallel Mapping

```python
from sametime import thread_map

results = thread_map(len, ['hello', 'world', 'concurrency'], max_workers=3)
assert results == [5, 5, 11]
```

### Streaming Results As They Complete

```python
from sametime import thread_iter

with thread_iter(fetch, urls, max_workers=4, ordered=False) as stream:
    for item in stream:
        handle(item)
```

### Tolerating Per-Item Errors with `Outcome`

```python
from sametime import thread_outcomes

with thread_outcomes(int, ['10', 'invalid', '30'], max_pending=2) as results:
    for outcome in results:
        if outcome.ok:
            print(f"Item #{outcome.index} = {outcome.value}")
        else:
            print(f"Item #{outcome.index} error: {outcome.error}")
```

## Documentation

See [docs/api.md](docs/api.md) for full API details, threading guarantees, and design considerations.

## Running Tests

```sh
python -m unittest discover -s tests -v
```

## License

MIT
