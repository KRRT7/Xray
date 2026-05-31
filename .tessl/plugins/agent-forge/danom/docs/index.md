# danom (v0.14.1)

Functional programming primitives for Python: `Result` (Ok/Err), `Either` (Right/Left), `Stream` pipelines, `@safe` decorators, `new_type`, function composition, and predicate combinators.

- Package: `danom` (PyPI)
- Language: Python
- Upstream: https://github.com/second-ed/danom
- API docs: https://second-ed.github.io/danom/
- Requires: Python >= 3.12

## Installation

```bash
pip install danom
```

If your project pins versions, align with the tile: `danom==0.14.1`.

## Core imports

```python
from danom import (
    Ok, Err, Result,        # Result monad
    Right, Left, Either,    # Either monad
    Stream,                 # Lazy pipeline
    safe, safe_method,      # Exception-catching decorators
    new_type,               # Validated wrapper types
    compose, identity, invert,
    all_of, any_of, none_of,
)
```

## Quickstart examples

### `@safe`: turn exceptions into `Result`

```python
from danom import safe

@safe
def parse_int(text: str) -> int:
    return int(text)

parse_int("42")    # Ok(inner=42)
parse_int("nope")  # Err(error=ValueError(...))
```

Guideline: inside a `@safe` function, return normal values and raise exceptions as usual — don’t manually construct `Ok(...)`/`Err(...)`.

### Chaining results (`and_then`, `or_else`)

```python
from danom import safe

@safe
def parse_float(text: str) -> float:
    return float(text)

@safe
def reciprocal(x: float) -> float:
    return 1.0 / x

result = parse_float("2.0").and_then(reciprocal)  # Ok(0.5)
result = parse_float("0").and_then(reciprocal)    # Err(ZeroDivisionError(...))
```

### Pattern matching on `Ok(...)`

```python
from danom import Ok

match result:
    case Ok(inner=value):
        handle(value)
    case _:
        log_error(result.error)
```

### Stream pipeline with `Result` helpers

```python
from danom import Stream, Result

values = (
    Stream.from_iterable(["1", "2", "nope", "3"])
    .map(parse_int)                 # Stream[Result[int]]
    .filter(Result.result_is_ok)    # keep Ok(...)
    .map(Result.result_unwrap)      # unwrap Ok values
    .collect()
)
```

### Validated wrapper types with `new_type`

```python
from danom import new_type

Email = new_type("Email", str, validators=[lambda s: "@" in s])
email = Email("user@example.com")
```

## API reference (high-signal)

### Result: `Ok` / `Err` / `Result`

#### Constructors

```python
Ok(inner=None)
Err(error=None, input_args=(), traceback="")
```

- `Ok(value)` wraps a successful value (default `None`)
- `Err(error=exc)` wraps an exception as the error

#### Core methods

| Method | Ok behavior | Err behavior |
|--------|-------------|--------------|
| `.is_ok()` | `True` | `False` |
| `.unwrap()` | returns `inner` | raises `error` |
| `.map(fn, *a, **kw)` | `Ok(fn(inner, *a, **kw))` | returns self |
| `.map_err(fn, *a, **kw)` | returns self | `Err(fn(error, *a, **kw))` |
| `.and_then(fn, *a, **kw)` | `fn(inner, *a, **kw)` | returns self |
| `.or_else(fn, *a, **kw)` | returns self | `fn(error, *a, **kw)` |

#### Class helpers (useful in Streams)

- `Result.unit(x)` → `Ok(x)`
- `Result.result_is_ok(result)` → `bool` (predicate)
- `Result.result_unwrap(result)` → unwrap Ok values

### Either: `Right` / `Left` / `Either`

Same interface shape as `Result`, but intended for “success vs alternative value”.

- `Right(inner=None)` / `Left(inner=None)`
- `.unwrap()` returns the inner value for both sides (no exception raising)
- Helpers: `Either.either_is_ok`, `Either.either_unwrap`

### `safe` / `safe_method`

`@safe` wraps a callable so it returns a `Result` instead of raising.

- On success: `Ok(return_value)`
- On exception: `Err(error=exc, input_args=(args, kwargs), traceback=formatted_str)`

`@safe_method` is the same idea for instance methods (forwards `self` correctly).

### Stream

Immutable lazy iterator with functional operations; everything is deferred until a terminal method.

```python
from danom import Stream

Stream.from_iterable([1, 2, 3]).map(lambda x: x + 1).filter(lambda x: x % 2 == 0).collect()
```

Common operations:

- Build: `Stream.from_iterable(iterable)`
- Lazy: `.map(fn, *args, **kwargs)`, `.filter(fn, *args, **kwargs)`, `.tap(fn)`
- Terminal: `.collect()`, `.fold(initial, fn)`, `.partition(fn)`, `.sequence()`

### Notable behaviors (useful to know)

- `Err.details` extracts traceback frame info including `locals` (be careful logging/serializing it).
- `Result.flatten()` / `Either.flatten()` collapse nested monads and return the first failing branch.
- `Stream.sequence()` converts `Stream[Result[T] | Either[T]]` into `Result[Stream[T]]` / `Either[Stream[T]]` (first error wins).

### Predicates and composition

- `compose(fn1, fn2, ...)` — pipe left-to-right
- `identity(x)` — return x unchanged
- `invert(fn)` — boolean negation wrapper
- `all_of(fn1, fn2, ...)` / `any_of(...)` / `none_of(...)` — predicate combinators

## Best practices

- Use `@safe` at boundaries (I/O, parsing, external calls), and propagate with `and_then`.
- Use `Result` when the failure mode is “exception”; use `Either` when both sides are normal values.
- Keep Stream pipelines small and readable (2–6 steps); extract named helpers if it grows.
- Don’t unwrap early: keep `Result` values until the end, then unwrap once.
