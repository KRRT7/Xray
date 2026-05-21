---
name: using-danom
description: >-
  Uses the danom tile documentation as on-demand reference (progressive disclosure).
  Use when you need to look up danom API signatures, draft correct usage examples, or refactor code to danom patterns
  (Result/Either/Stream/@safe/new_type/compose) without inventing APIs.
---

# Using danom

## When to load references

- If you need imports, quick examples, or method names, read `../../docs/index.md`.
- If you need code patterns Tessl enforces for danom usage in this repo, read `../../rules/danom-patterns.md`.

## Minimal example (copy/paste)

```python
from danom import safe

@safe
def parse_int(text: str) -> int:
    return int(text)

result = parse_int("123")  # Ok(...)
```