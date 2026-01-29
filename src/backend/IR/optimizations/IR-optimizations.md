# Optimizations performed
<!-- TODO: intro and main optimizing stages -->

## General constructs and definitions used throughout

### Dominator tree
A _dominator tree_ is used in ...


At this level, we define a `Subroutine` as _trivial_ if:
- it has a single basic block,
- it has _at most_ one instruction, and
- no `Phi` nodes are part of any of its `BasicBlock`s.

## Subroutine inlining
> [INFO!] this will only run in optimization level O2. This is because it may tamper with error comment locations.

<!-- TODO: a redundant IF here? (L:157) -->




## Split parallel copies
## Constant replacer
## Copy propagation
## Itxn field calls elision
## Remove unused variables
## Simplify intrinsics
## Replace Itxn Fields
## Replace Compiled references
## Simplify control ops.
## Merge blocks
## Remove linear jumps
## Remove unreachable blocks (DCE)
## Repeated Expression Elimination (RCE)
The main loop of RCE goes like this:
```python
def repeated_expression_elimination(
    context: CompileContext, subroutine: models.Subroutine
) -> bool:
    start, dom_tree = compute_dominator_tree(subroutine)
    modified = False
    while _recursive_rce(dom_tree, start, const_intrinsics={}, asserted=set()):
        modified = True
        copy_propagation(context, subroutine)
    return modified
```
We first start by computing the dominator tree (see annex).
We get as a result an entry basic block `start` (by definition it dominates every other bb),
and a `dom_tree` which is a mapping of basic blocks to lists of basic blocks dominated by them.
In other words, we get:\
<!-- TODO: diagram -->


## Encode-Decode pair elimination
## Merge chained aggregate reads
## Replace aggregate box ops.
## Minimize box exist asserts

## Constant reads and unobserved writes elimination

<!-- this "opt." is only performed after all IR lowerings but BEFORE SSA destructuring -->
## Slot elimination
TODO: explain


## Post-destructuring optimizations