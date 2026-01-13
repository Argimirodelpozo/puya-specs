# Intermediate Representation (IR) layer

## Lowering from AWST

TODO: IR main until we get to the building main
(logicsig vs. contract build, embedded subroutines, etc.)



### Builder



## SSA construction: Braun Algorithm


## Implementation details

The following diagram shows the transformation pipeline schematically.

```mermaid
flowchart TD
    A[Start: get_transform_pipeline]

    B[_optimize_program_ir<br/>qualifier="ssa.opt"<br/>artifact_ir]
    C[_lower_aggregate_ir<br/>ref]
    D[_optimize_program_ir<br/>qualifier="ssa.array.opt"<br/>artifact_ir]
    E[slot_elimination<br/>ref]
    F[destructure_ssa]

    A --> B --> C --> D --> E --> F
```


After building the IR, the _transform pipeline_ is as follows:
```python
def get_transform_pipeline(
    artifact_ir: ModuleArtifact,
) -> list[Callable[[ArtifactCompileContext, Program], None]]:
    ref = artifact_ir.metadata.ref
    return [
        functools.partial(_optimize_program_ir, artifact_ir=artifact_ir, qualifier="ssa.opt"),
        functools.partial(_lower_aggregate_ir, ref=ref),
        functools.partial(
            _optimize_program_ir, artifact_ir=artifact_ir, qualifier="ssa.array.opt"
        ),
        functools.partial(slot_elimination, ref=ref),
        destructure_ssa,
    ]
```
In other words, the pipeline does IR level optimization, aggregate lowering, 
another full optimization pass (same opts.), then specifically slot elimination,
and finally destructuring.


# Optimizations performed

At this level, we define a subroutine as "trivial" if:
- it has a single basic block,
- it has _at most_ one instruction, and
- no phi instructions

## Subroutine inlining

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


## SSA Destructuring


# Validations performed
TODO: intro text

## After [destructuring](#ssa-destructuring)
TODO: all validators here (from IR/validate)

### Infinite loop detector

```py
class NoInfiniteLoopsValidator(DestructuredIRValidator):
    @typing.override
    def visit_block(self, block: models.BasicBlock) -> None:
        assert block.terminator is not None, "unterminated block found during IR validation"
        if block.terminator.unique_targets == [block]:
            logger.error(
                "infinite loop detected",
                location=block.terminator.source_location or block.source_location,
            )
```
This validation pass checks that each basic block has a terminator, and that the target block for the terminator is _not_ uniquely itself, as that would constitute a very simple case of an infinite loop.
> [!Note]
> Infinite loops don't make sense in the AVM architecture, as they would just consistently run out of budget, reverting any state changes they may attempt and wasting computation.
TODO: what about an use case for programs not meant to be run but only simulated (e.g. Algoland Tasos example)


# Full models reference
