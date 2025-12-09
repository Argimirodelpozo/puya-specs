# TEAL layer (AVM code or "final" lowering)

The MIR program is consumed by mir_to_teal(.), where the program "main" subroutine is built first.
Then, each of the other "subroutines" are built.

Optionally, the TEAL at this intermediate, unoptimized stage is output, tagged as "lowered".

Then optimizations are run on the lowered TEAL. Note that there are many optimizations here
that are run regardless of optimization level.

Finally the transformed full TEAL program is returned.


# Lowering from MIR
The [main algorithm](TODO_LINK) for this lowering does the following:
(TODO:pseudocode)
```python
def mir_to_teal(
    context: ArtifactCompileContext, program_mir: mir.Program
) -> teal_models.TealProgram:
    main = TealBuilder.build_subroutine(
        program_mir.main, slot_allocation=program_mir.slot_allocation
    )
    subroutines = [TealBuilder.build_subroutine(mir_sub) for mir_sub in program_mir.subroutines]
    teal = teal_models.TealProgram(
        kind=program_mir.kind,
        avm_version=program_mir.avm_version,
        main=main,
        subroutines=subroutines,
    )
    maybe_output_intermediate_teal(context, teal, qualifier="lowered")
    initial_check_set = _collect_explicit_checks(teal)
    optimize_teal_program(context, teal)
    post_allocation_check_set = _collect_explicit_checks(teal)
    for check_data, initial_count in initial_check_set.items():
        # less than rather than != since we can duplicate ops for inlining
        if post_allocation_check_set.get(check_data, 0) < initial_count:
            raise InternalError("explicit condition check(s) removed during TEAL optimization")
    return teal
```
That is, the special `main` subroutine is built first, then each subroutine in the program.
A `TealProgram` structure is created with these, with the corresponding avm version and program kind (wether a stateful application or a logic signature).\
Explicit checks ([`Assert`](TODO_LINK) and [`Err`](TODO_LINK) instructions) are collected.\
[Optimizations](#optimizations-performed) are performed, and post-optimization explicit checks are collected again, and compared to those collected pre-optimization (see the [validations performed](#validations-performed) section below).\
Finally, the optimized TEAL program is output.

## Intra-layer transformations


# Optimizations performed

The main optimization loop (in [main.py](../puya/src/puya/teal/optimize/main.py)) executes
for each subroutine, already lowered into TEAL after [MIR => TEAL lowering](MIR.md),
the following set of optimizations is performed (reliant on )


## Peephole optimizations
These are optimizations 
Four windows of one, two, three and four opcodes respectively will be used.
They will appear sequentially in order of window size, and are 

Some preliminary definitions:
- `{COMM_OP}` is a commutative op. One of 
`[
        "+",
        "*",
        "&",
        "&&",
        "|",
        "||",
        "^",
        "==",
        "!=",
        "b*",
        "b+",
        "b&",
        "b|",
        "b^",
        "b==",
        "b!=",
        "addw",
        "mulw"
]`
- `{SWAP_OP}` is a stack swap op. One of 
`[
        "swap",
        "cover 1",
        "uncover 1"
]`

By size of the peephole window defined, we have the following optimizations performed.

### Singles:
Note that all of these are performed at all opt. levels (even -O0).
- `arg n` -> `arg_n` (for `n <= 3`).
- `cover 0` -> []. Also `uncover 0` -> [].
- `dig 0` -> `dup`.
- `popn 1` -> `pop`.

### Pairs:
Note that all of these are performed even at -O0.
- Redundant rotation removal. This encompasses:
        1) `cover n; uncover n;` -> []. 
        2) `uncover n; cover n;` -> [].
        3) `swap; swap;` -> [].
        4) `swap; cover 1;` -> [].
        5) `swap; uncover 1;` -> [].
        6) `cover 1; swap;` -> [].
        7) `cover 1; cover 1;` -> [].
        8) `cover 1; uncover 1;` -> []. (already transformed in 1st rule).
        9) `uncover 1; swap;` -> [].
        10) `uncover 1; cover 1;` -> []. (already transformed in 1st rule).
        11) `uncover 1; uncover 1;` -> [].
- Any stack swap followed by a `pop` is replaced by a `bury 1`:
        1) `swap; pop` -> `bury 1`.
        2) `cover 1; pop` -> `bury 1`.
        3) `uncover 1; pop` -> `bury 1`.
- Any stack swap followed by a commutative op is replaced by just the op by itself. Like this:
`{SWAP_OP}; {COMM_OP}` -> `{COMM_OP}`.
- `frame_dig n; frame_bury n` -> [].
- `frame_bury n; frame_dig n` -> `dup; frame_bury_n;`.
- `dup; {SWAP_OP}` -> `dup`. Also `dupn; {SWAP_OP}` -> `dupn`.
- `dup; pop` -> [].
(TODO: complete!)

- `dig 1; dig 1` -> `dup2`.
- `int 0; return -> err`.
- Any stack swap operations followed by binary ordering operations are reversed. Like this:
`{SWAP_OP}; {BIN_ORD_OP}` -> `{flip(BIN_ORD_OP)}`, where `flip()` finds usage of `>` or `<` and
inverts them (e.g. `>=` becomes `<=`).


### Triplets:
If any of the ops in the analysis triplet is a `frame_dig` operation, a special analysis is performed. 
<!-- TODO explain frame_dig analysis and why its relevant-->
- `'cover 3; cover 3; {SWAP_OP}` -> `uncover 2; uncover 3`
Proof: ...

- `uncover 2; {SWAP_OP}; uncover 2` -> `swap`
Proof: ...

TODO: complete

### Quadruplets:
TODO

## Subroutine Block optimizations

## Constant gathering

## Combine pushes




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



# Validations performed
...TODO

## Explicit checks should remain
Before 


# TEAL Layer nodes
The IR instructions in this layer are a quite close model of AVM TEAL ops, save for structural containers and template variables which are not part of AVM intrinsics.

Consider a stack, modelled as a list of local ids, which are strings that represent local variables.

We define 5 kinds of stack manipulations:
- `StackConsume`: takes `n` elements off the top of the stack.
- `StackExtend`: adds a sequence of local id's to the top of the stack.
- `StackDefine`: considering the set of unique variables in the stack. It performs a set intersection with the given elements, returning an extended set.
- `StackInsert`: inserts a local id at a given stack `depth`. For a given stack $s$, the insertion index is computed as $idx = |s| - depth$.
- `StackPop`: pops the stack to the specified `depth`. The index of the eliminated element is computed as $idx = |s| - depth - 1$



Note that in this layer, the nodes/models of the resulting language represent TEAL opcodes of the latest AVM version. An abstract opcode is modelled using the following fields:
(link to TealOp class)

- a string containing the opcode name
- a pair of unsigned integers representing the consumption and production of vals from stack

and optionally:
- a source location in code (see common reference)
- a comment to be emmitted after the op in the resulting TEAL
- an error message to be emmited when/if the program fails trying to execute this op
- a sequence of stack manipulations (...)

<!-- ```python
class TealOp:
    op_code: str
    consumes: int
    produces: int
    source_location: SourceLocation | None = attrs.field(eq=False)
    comment: str | None = None
    """A comment that is always emitted after the op in TEAL"""
    error_message: str | None = None
    """Error message to display if program fails at this op"""
    stack_manipulations: Sequence[StackManipulation] = attrs.field(
        default=(),
        converter=tuple[StackManipulation, ...],
        eq=False,
    )

    @property
    def immediates(self) -> Sequence[int | str]:
        return ()

    def teal(self) -> str:
        teal_args = [self.op_code, *map(str, self.immediates)]
        if self.comment or self.error_message:
            error_message = (
                format_error_comment(self.op_code, self.error_message)
                if self.error_message
                else ""
            )
            comment_lines = error_message.splitlines()
            comment_lines += (self.comment or "").splitlines()
            comment = "\n//".join(comment_lines)
            teal_args.append(f"// {comment}")
        return " ".join(teal_args)

    @property
    def stack_height_delta(self) -> int:
        return self.produces - self.consumes
``` -->


