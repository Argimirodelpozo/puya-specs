# Dead store removal (x-stack, f-stack cases)

> [!Note] l-stack dead store removal occurs during l-stack allocation. The other two stack region cases are handled here.

```py
    if vla.is_dead_store(b):
        return a, mir.Pop(n=1, source_location=b.source_location)
```
If `b` constitutes a *dead store* (see [Variable Lifetime Analysis section](#variable-lifetime-analysis) above), then it may be replaced by a `MIR.Pop` operation to pop the top value of the stack, as the last product of `a` is stored but never used again, and can therefore be safely discarded.
The resulting pattern is then `(a, MIR.Pop(n=1))`.\

TODO: illustrative example