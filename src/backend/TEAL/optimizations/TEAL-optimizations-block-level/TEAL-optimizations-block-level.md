# Subroutine Block optimizations
> [!INFO] this set of optimizations is only performed on optimization levels `O1` and `O2`.

Most of the subroutines in this pass deal with getting rid of jumps by inlining when possible. We simplify chains of single unconditional jump instruction blocks,...
<!-- TODO: complete intro -->


### Inline optimizations: jump chains
Consider now the set of all blocks $b$ that are a single unconditional `Branch` to a target, have no stack manipulations associated, and are not the entry block.

We remove these from the subroutine blocks, keeping track of the link between original block label and their target.

Now, for every jump chain $b_0 => b_1 => ... => b_n$, we backpropagate in order to have every $b_0, b_1...b_{n-1} => b_n$, mapping their unique identifying labels to the last on the chain.

After we have simplified all possible chains, we traverse all instructions inside the subroutine. For every jump instruction (`Branch`, `BranchNonZero`, `BranchZero`, `Switch` and `Match`) targetting any element in a chain, we replace the target by the last element.


### Inline optimizations: single instruction blocks
TODO: complete

### Inline optimizations: singly referenced blocks
TODO: complete

### Replacement of subroutine invocations for branches
For a given subroutine, we iterate over all instructions. If a given `callsub` op. is found, and the jump target subroutine is determined to be "branchable" (see above for the definition), the operation is replaced by a hard branch (`b`, represented in this layer by the [`Branch`](#TODO_branch) node) to the same jump target.


### Inline jump chains
TODO: this one is repeated, but is the same as above. Should we keep it here?


### Remove jump fallthroughs
As per the building process, [MIR Conditionals](#TODO_LINK_MIR), as well as [Match] and [Switch] MIR nodes, generate explicit fallthrough hard branches (`b`, represented as `Branch` in this layer). These fallthroughs are useful for some analysis, but have no impact in control flow as the natural opcode evaluation order would trivially flow into the next line. Therefore, they are trivially removable at this stage.\
Consider every pair of consecutive blocks $b0, b1$. whenever the last instruction in $b0$ is a `Branch` (unconditional branch) and the jump target is the unique identifying label of $b1$, we remove the branch operation from $b0$. 

TODO: what happens if I have an explicit branch to the label right next to it? Should we protect against this by making sure the branch is a fallthrough (e.g. check that the instruction right before is a bz or bnz)?

TODO: understand stack manipulations guard case for O0


## Constant gathering
> [!INFO] this optimization is performed on all optimization levels (`O0`, `O1` and `O2`).

> [!NOTE] this optimization is needed in `O0` because, due to ARC56, template variables need to be gathered into constant blocks for pc offset calculations. See the [ARC56 standard](TODO_LINK) for further details.




## Combine pushes

> [!INFO] this is only done for optimization levels `O1` and `O2`.

If there is any sequence of consecutive `pushint` or `pushbytes` operations present in any block, in any subroutine, 
they are compressed into a `pushints` or `pushbytess` respectively. In the case of `pushbytess`, byte encodings are preserved for each value. Comments are comma-concatenated accordingly.
TODO: example


<!-- def optimize_teal_program(
    context: ArtifactCompileContext, teal_program: models.TealProgram
) -> None:
    # O0 will still remove some redundant ops to ensure a feasible program size
    for teal_sub in teal_program.all_subroutines:
        _optimize_subroutine_ops(context, teal_sub)
    maybe_output_intermediate_teal(context, teal_program, qualifier="peephole")

    if context.options.optimization_level > 0:
        branchable_subroutine_entry_blocks = {
            sub.blocks[0].label
            for sub in teal_program.all_subroutines
            if _is_branchable_subroutine(sub)
        }

        for teal_sub in teal_program.all_subroutines:
            _optimize_subroutine_blocks(context, teal_sub, branchable_subroutine_entry_blocks)
        maybe_output_intermediate_teal(context, teal_program, qualifier="block")

    gather_program_constants(teal_program)
    if context.options.optimization_level > 0:
        combine_pushes(teal_program) -->