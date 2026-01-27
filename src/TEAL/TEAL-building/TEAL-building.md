## Building
Most MIR nodes are lowered as a single TEAL node, which in turn almost always represent a single TEAL op.\
Notable exceptions to this rule are MIR `ConditionalBranch` nodes (which get lowered as 2 ops. to make the fallthrough case explicit), similarly to `Switch` and `Match` nodes.

> [!NOTE] after building, `TEALBlocks` are no longer assured to be strict basic blocks with a single exit point.
<!-- TODO: explain exceptions -->

<!-- Furthermore, as the `MIR` stage output is very close to a one to one mapping of TEAL code, many  -->

### `MIR.MemorySubroutine` to `TealSubroutine`
The main driver of the building process, this process is carried out for each subroutine in the `MIRProgram`.
In pseudocode, for a given `MemorySubroutine` `ms`:

<!-- TODO: finish pseudocode -->
```python
    TealSubroutine out
    if needsProto(ms):
        out.blocks = [protoBlock]
```
The first stage 


TODO: all other built models (any interesting parts)


### Conditional Branch
A [MIR conditional branch node](../specs/MIR.md) gets lowered as either a `BranchZero` TEAL node or a `BranchNotZero` TEAL node, followed by a `Branch` TEAL node targeting the next block.

> [!INFO] Right after building but before any optimizations, an output may be obtained. The output at this stage is tagged "lowered" (for example `my_contract.lowered.teal`), and is governed by the `--output-intermediate-teal` flag.