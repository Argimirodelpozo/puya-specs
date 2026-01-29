<!-- DRAFT -->

# Optimizations performed
There is a single optimization function, a peephole optimizer with a window of size 2 (i.e. optimizing instruction pairs), however this optimization pass is performed after resolving allocation of each of the three regions (for `O1` and `O2` opt. levels). Note that `O0` opt. level performs no optimizations in this layer, and only allocates [L-stack](#the-l-stack) and [F-stack](#the-f-stack) (leaving the [X-stack](#the-x-stack) region empty).

For the optimization pass, it's crucial to first perform a subroutine-wide variable lifetime analysis.

## Variable lifetime analysis [^1]

[^1]: this section is heavily based on https://www.classes.cs.uchicago.edu/archive/2004/spring/22620-1/docs/liveness.pdf

We say that a variable is _live_ if it holds a value that will/might be used in the future.

In the subsequent analysis, a node `n` represents an [MIR instruction](#full-models-reference). The relevant nodes for this analysis are `AbstractStore` and `AbstractLoad`.

We define the following sets:

`use[n]`: the set of variable uses in the node `n` of the control flow graph. A variable use is a right hand side appearence of a variable's identifier.

`def[n]`: the set of variable definitions in the node `n` of the control flow graph. A variable definition is a left hand side appearence of a variable's identifier.

`in[n]`: the set of variables that are live-in at node `n`. A variable (temp) `a` is live-in at node `n` if it is used at `n` (`a` $\in$ `use[n]`), or if there is a path from `n` to a node that uses `a` that does not contain a definition of (assignment to) `a`. We write `a` $\in$ `in[n]`.

`out[n]`: the set of variables that are live-out at node `n`. A variable `a` is live-out at node `n` if it is live-in at one of the successors of `n`. We write `a` $\in$ `out[n]`.

The sets satisfy the following two equations:

`in[n]` $=$ `use[n]` $\cup$ (`out[n]` $-$ `def[n]`)

`out[n]` $=$ $\cup$ \{ `in[s]` $|$ `s` $\in$ `succ[n]` \}

Furthermore, a node `n'` that is an `AbstractStore` is a *dead store* if the variable to which it stores `v` (identified solely by its local id) is such that:

`v` $\notin$ `out[n']`.

In other words, the target store variable `v` is not live-out for `n'`, which means that there is no control flow path to a node that uses this variable from this point of the program on.

### Iterative construction of variable lifetime analysis sets
TODO: explain

## Peephole optimizations
We define a peephole window of size 2, meaning the peephole optimizer will consider _pairs_ of operations inside each [basic block](#memorybasicblock).
For all these, we consider a pair of ops. `(a, b)`.\
The optimizer will perform the following passes in the order in which they are declared:

<!-- TODO: all the following are not independant. Should they be in one big optimization all together? -->
### Move `Store{L,X,F}Stack` id's into products of previous op.

```py
# move local_ids to produces of previous op where possible
    if (
        isinstance(b, mir.StoreLStack | mir.StoreXStack | mir.StoreFStack)
        and a.produces
        and a.produces[-1] != b.local_id
    ):
        a = attrs.evolve(a, produces=(*a.produces[:-1], b.local_id))
        return a, b
```
> [Reference implementation](TODO_LINK)

Consider a case where the first element of the pair, `a`, is an op. that produces a non-empty list of `n` local ids `a.prod = [a_1, a_2, ..., a_n]` (`n > 0`). Now, assume the second element, `b`, is a *materialised* store operation (one of MIR.StoreLStack, MIR.StoreXStack or MIR.StoreFStack). Furthermore, consider that the target local id of `b` is such that `b.local_id != a_n`. In this case, we have an implicit aliasing of the local id `a_n` into `b.local_id`. We can thus get rid of `a_n` by modifying the products of `a` in place.\
The resulting pair is then `(a', b)` where `a'.prod = [a_1, a_2, ..., b.local_id]`.

TODO: illustrative example

Note that, since this optimization is run after each stack region's allocation, each subsequent run unlocks new potential cases for `b`.

### Eliminate top-of-stack renamings

```py
    # remove redundant stores and loads
    if a.produces and a.produces[-1] == _get_local_id_alias(b):
        return (a,)
```
> [Reference implementation](TODO_LINK)

Consider now a pair where the last product of `a` is just a local id alias in `b`.
This is, if `b` is a `MIR.StoreLStack` or a `MIR.LoadLStack` without copy, and the `depth` is 0. In other words, `b` takes the top of the stack and, through a store/load operation, renames it to `b.local_id`.\
Then, `b` may be safely removed, and the resulting pattern is just `(a, )`.

### Fold `store=>load` chains inside the same region (non-rotational)

```py
    if isinstance(a, mir.LoadOp) and isinstance(b, mir.StoreOp) and a.local_id == b.local_id:
        match a, b:
            case mir.LoadXStack(), mir.StoreXStack():
                return ()
            case mir.LoadFStack(), mir.StoreFStack():
                return ()
            case mir.AbstractLoad(), mir.AbstractStore():
                # this is used see test_cases/bug_load_store_load_store
                return ()
```

If `a` is a load operation, `b` is a store operation, and the local ids match, we have a `store=>load` chain. If, furthermore, these are both either:
- x-stack operations,
- f-stack operations, or

<!-- TODO: analyze THIS case. Where is this used? check the example provided -->
- abstract (i.e. unmaterialized yet),\

we can safely remove them without breaking dataflow, as the load means the variable is already stored in the correct region of the stack.\
The resulting pattern is then an empty tuple `()`, as both elements of the pair are removed.

TODO: illustrative example