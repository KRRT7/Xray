# danom

Functional programming primitives for Python: `Result` (Ok/Err), `Either` (Right/Left), `Stream` pipelines, `@safe` decorators, `new_type`, function composition, and predicate combinators.

- Package: `danom` (PyPI) — `danom==0.14.1`
- Upstream: <https://github.com/second-ed/danom>
- Requires: Python >= 3.12

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

## `@safe` decorator

Use `@safe` on functions that may fail. Never construct `Ok()`/`Err()` inside a `@safe` function — just return or raise.

```python
@safe
def load(path: str) -> dict:
    return json.loads(Path(path).read_text())

load("data.json")  # Ok({"key": "val"}) or Err(error=FileNotFoundError(...))
```

Use `@safe_method` for instance methods (forwards `self`).

- On success: `Ok(return_value)`
- On exception: `Err(error=exc, input_args=(args, kwargs), traceback=formatted_str)`

## Composing Results

Inside a `@safe` function, `.unwrap()` inner Results to propagate failures without branching:

```python
@safe
def pipeline(path: str) -> Report:
    data = load(path).unwrap()       # Err propagates as exception
    return process(data).unwrap()
```

For explicit chaining outside `@safe`, use `.and_then` / `.or_else`:

```python
parse_float("2.0").and_then(reciprocal)  # Ok(0.5)
parse_float("0").and_then(reciprocal)    # Err(ZeroDivisionError(...))
```

## Result API

### Constructors

```python
Ok(inner=None)
Err(error=None, input_args=(), traceback="")
```

### Methods

| Method | Ok behavior | Err behavior |
| ------ | ----------- | ------------ |
| `.is_ok()` | `True` | `False` |
| `.unwrap()` | returns `inner` | raises `error` |
| `.map(fn, *a, **kw)` | `Ok(fn(inner, *a, **kw))` | returns self |
| `.map_err(fn, *a, **kw)` | returns self | `Err(fn(error, *a, **kw))` |
| `.and_then(fn, *a, **kw)` | `fn(inner, *a, **kw)` | returns self |
| `.or_else(fn, *a, **kw)` | returns self | `fn(error, *a, **kw)` |

### Class helpers (useful in Streams)

- `Result.unit(x)` → `Ok(x)`
- `Result.result_is_ok(result)` → `bool` (predicate)
- `Result.result_unwrap(result)` → unwrap Ok values

### Pattern matching

```python
match result:
    case Ok(inner=value):
        handle(value)
    case _:
        log_error(result.error)
```

## Either monad

Prefer `Either` when both branches are "normal values". Prefer `Result` when failures are exceptions.

- `Right(inner=None)` / `Left(inner=None)`
- `.unwrap()` returns the inner value for both sides (no exception raising)
- Helpers: `Either.either_is_ok`, `Either.either_unwrap`

## Stream pipelines

Use `Stream.from_iterable()` for multi-step transforms. List comprehensions for single-step.

```python
Stream.from_iterable(items).map(transform).filter(predicate).collect()
```

Common operations:

- Build: `Stream.from_iterable(iterable)`
- Lazy: `.map(fn, *args, **kwargs)`, `.filter(fn, *args, **kwargs)`, `.tap(fn)`
- Terminal: `.collect()`, `.fold(initial, fn)`, `.partition(fn)`, `.sequence()`

When filtering/unwrapping `Stream[Result[T]]`, use the class helpers:

```python
(
    Stream.from_iterable(["1", "2", "nope", "3"])
    .map(parse_int)                 # Stream[Result[int]]
    .filter(Result.result_is_ok)    # keep Ok(...)
    .map(Result.result_unwrap)      # unwrap Ok values
    .collect()
)
```

If you need to "fail fast" a stream of results, prefer `Stream.sequence()` — it converts `Stream[Result[T]]` into `Result[Stream[T]]` (first error wins) — instead of collecting and branching manually.

## Predicates and composition

- `compose(fn1, fn2, ...)` — pipe left-to-right
- `identity(x)` — return x unchanged
- `invert(fn)` — boolean negation wrapper
- `all_of(fn1, fn2, ...)` / `any_of(...)` / `none_of(...)` — predicate combinators

## Validated wrapper types

```python
Email = new_type("Email", str, validators=[lambda s: "@" in s])
email = Email("user@example.com")
```

## Notable behaviors

- `Err.details` extracts traceback frame info including `locals` (be careful logging/serializing it).
- `Result.flatten()` / `Either.flatten()` collapse nested monads and return the first failing branch.
- `Stream.sequence()` converts `Stream[Result[T] | Either[T]]` into `Result[Stream[T]]` / `Either[Stream[T]]`.
