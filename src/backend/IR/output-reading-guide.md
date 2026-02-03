# IR Output reading guide

The IR layer is the compiler stage responsible for the heaviest optimization and code transformation in the pipeline. As such, it presents a rich variety of intermediate inputs.\
Most of them follow a similar syntax, and their main difference is at which point of the overarching IR pipeline we find ourselves in.

There are 4 main sub-stages to the IR stage that may generate intermediate output:
- The `build` stage, posfixed with the `"ssa"` qualifier. This is generated right after the IR building process has taken place, but before any optimizations are performed.
- The first `optimized` stage, posfixed with the `ssa.opt` qualifier. This is a family of outputs that are generated after each modifying optimization pass.
- The `array` and aggregates stage, posfixed with the `ssa.array` qualifier. This is output generated right after aggregate lowering.
- The second `optimized` stage, posfixed with the `ssa.array.opt` qualifier. A family of outputs that result from optimization passes being run after aggregate lowering.
- The final `destructuring` stage, posfixed with the `destructured` qualifier. This is a single output after we destructure the IR code (meaning, we come "out" of SSA form and resolve `Phi` nodes).\
This also constitutes the final output of the IR stage, and the input state for the [`MIR` stage](../MIR/MIR.md) of the pipeline.

## Some general considerations

## Main difference between `.ssa`/`.ssa.opt` and `array`

## Main difference between `.ssa`/`.array` (and their `.opt` versions) and `.destructured`