<!-- DRAFT -->

# `IR` => `MIR` lowering

## Slot allocation strategy
<!-- TODO: slot allocation expl. goes here -->


# The Stack visitor
The stack visitor is a visitor that processes ops and subsequently builds a stack with its regions explicitly. It applies the effects of the visiting op into the l-stack.

<!-- TODO: explain -->

# Global Stack Allocation
<!-- TODO: pseudocode -->
```python
GlobalStackAllocation(Program P)
    for Subroutine s in P: #note that this includes main as well as every other subroutine
        if opt level is O0:
            lstack(s)
            fstack(s)
        else:
            lstack(s)
            peepholeOptimizationPass(s)
            xstack(s)
            peepholeOptimizationPass(s)
            fstack(s)
            peepholeOptimizationPass(s)
```
<!-- 
def global_stack_allocation(ctx: ProgramMIRContext, program: models.Program) -> None:
    for desc, (method, min_opt_level) in {
        "lstack": (l_stack_allocation, 0),
        "lstack.opt": (peephole_optimization_single_pass, 1),
        "xstack": (x_stack_allocation, 1),
        "xstack.opt": (peephole_optimization_single_pass, 1),
        "fstack": (f_stack_allocation, 0),
        "fstack.opt": (peephole_optimization_single_pass, 1),
    }.items():
        if ctx.options.optimization_level < min_opt_level:
            continue
        for mir_sub in program.all_subroutines:
            method(ctx, mir_sub)
        if ctx.options.output_memory_ir:
            output_memory_ir(ctx, program, qualifier=desc)
    if ctx.options.output_memory_ir:
        output_memory_ir(ctx, program, qualifier="") -->
(...)
After MIR building has taken place and slot allocations have been solved, global stack allocation is performed. This processing step performs several optimizations and builds L-, X- and F- stacks (according to optimization flags).\
In the `O0` level, only L-Stack and F-Stack are built, and no optimization passes are performed.
In any other optimization level (`O1` or `O2`), every stack region allocations are built in the order seen in the algorithm, which is the order in which the regions "stack" on top of each other in the final model of the Stack (note a pass of peephole optimizations is performed after each one).

## The L-stack
The L-stack allocations are built by applying [Koopman's algorithm](TODO:LINK).
At a high level, this is the region of the stack responsible for intra-block allocation, and will attempt to materialize some abstract storage operations as [`LStackLoad`](#local-variable-l-stack-ops) and `LStackStore` ops.

(TODO: pseudocode impl -very similar to python impl)

- For each basic block in every subroutine, we find usage pairs (see definition below).
Then, for each usage pair, we copy the first element (a define or a use) to the bottom of the l-stack. 

insert_index = a_index if isinstance(a, mir.AbstractStore) else a_index + 1
Copy usage pairs...
Then dead store removal pass.

## Deadstore removal
TODO: pseudocode deadstore removal

TODO: high level desc.

We define a dead store as a store for a variable (identified by local id $ID$) that is not alive at the exit of the block.
TODO: EXAMPLE

Firstly, we perform Variable Lifetime Analysis for the given subroutine. Then, for each basic block in the subroutine body, we loop through every op.
We take an analysis window of two ops (i.e. we analyze them in consecutive pairs) for every LStack Store.

<!-- def _dead_store_removal(sub: mir.MemorySubroutine) -> None:
    vla = VariableLifetimeAnalysis(sub)
    for block in sub.body:
        ops = block.mem_ops
        op_idx = 0
        while op_idx < len(ops) - 1:
            window = slice(op_idx, op_idx + 2)
            a, b = ops[window]
            # StoreLStack is used to:
            #   1.) create copy of the value to be immediately stored via virtual store
            #   2.) rotate the value to the bottom of the stack for use in a later op in this block
            # If it is a dead store, then the 1st scenario is no longer needed
            # and instead just need to ensure the value is moved to the bottom of the stack
            if isinstance(a, mir.StoreLStack):
                if a.copy and vla.is_dead_store(b):
                    a = attrs.evolve(a, copy=False, produces=())
                    ops[window] = (a,)
            elif (
                (isinstance(a, mir.LoadLStack) and not a.copy)
                and (isinstance(b, mir.StoreLStack) and b.copy)
                and a.local_id == b.local_id
            ):
                a = attrs.evolve(
                    a,
                    copy=True,
                    produces=(f"{a.local_id} (copy)",),
                )
                ops[window] = (a,)
            op_idx += 1 -->

If opt level >= 1, implicit store removal.\

## Implicit store removal optimization

In IR terms, an operation is assigned to a temp, which then is immediately assigned to another variable.
We may eliminate the temp intermediate step.

This is carried away in a peephole fashion. For each op in a basic block that produces any L-stack output, we take a window equal to the ops. output production size (length of the `produces` field).
If every one of these subsequent ops. inside the window immediately following the selected op.: 
1. are L-stack load no-copy operations (i.e. stack rotations), and
2. are exactly equal to the list of produced local ids _reversed_.
Then, all the ops in the analysis window are eliminated.



Finally, _calculate_load_depths (once l-stack allocations are finished)

## Load depths calculation
Now that all L-stack allocations are finished, we may compute load depths.
First instantiate a [Stack visitor](TODO:LINK).
Then, iterate on each op of each basic block in the given subroutine.
The StackVisitor will keep track of l-stack effects for each visited op. At this stage, we don't have to worry about any other stack regions but L.
                <!-- local_id_index = stack.l_stack.index(op.local_id)
                block.mem_ops[idx] = attrs.evolve(
                    op, depth=len(stack.l_stack) - local_id_index - 1
                ) -->
Whenever we iterate over a LoadLStack, and before visiting the operation with the StackVisitor, we
set its depth. Given the current length of the l-stack being constructed by the visitor, and the index at which the local id being stored to appears on the l-stack, we set the depth feld in the `LoadLStack` to be:

$depth = |partial \ l-stack| - index - 1$.

TODO: explain WHY


TODO: explain this better but main algo is these steps
    <!-- # the following is basically koopmans algorithm
    # done as part of http://www.euroforth.org/ef06/shannon-bailey06.pdf
    # see also https://users.ece.cmu.edu/~koopman/stack_compiler/stack_co.html#appendix
    for block in sub.body:
        usage_pairs = _find_usage_pairs(block)
        _copy_usage_pairs(block, usage_pairs)
    _dead_store_removal(sub)
    if ctx.options.optimization_level:
        _implicit_store_removal(sub)
    # calculate load depths now that l-stack allocations are done
    _calculate_load_depths(sub) -->


In order to build the l-stack, we define a `UsagePair` as a tuple where the first element is an `AbstractLoad` or an `AbstractStore`, and the second element is always an `AbstractLoad` referencing the same variable.\
In other words, a `UsagePair` models, in terms of the MIR, a `use->use` or `def->use` chain.
`(mir.AbstractLoad | mir.AbstractStore, mir.AbstractLoad)`.

> [!NOTE]
> See `/mnt/c/Users/x/Desktop/Puya_learning/puya/src/puya/mir/stack_allocation/l_stack.py`
> In this implementation, we first construct a mapping of local_ids (variable identifier at this stage) to (index, op) tuples, where op is an abstract load or store. Then we take the tuples for a given variable in consecutive pairs and compute their sorting keys.

For each basic block in the program, consider an ordered list of usage pairs, $L_{u}$.
Note that for any entry (u_1, u_2) on this list, u_1 comes strictly before u_2, and therefore the intra-basic block index of u_1, i(u_1) will always be such that $i(u_1) < i(u_2)$.
We compute this list using the following sorting criteria:
- for each pair u_1=>u_2, we compute the amount of instructions between u_1 and u_2. Then we sort the pairs in ascending order of this number;
- when tied, we break the tie by the intra-basic block index of the u_1 instruction;
- when tied, we break the tie by the intra-basic block index of the u_2 instruction.

A visitor is defined, LStackHeight, in order to compute the L-stack height in between ops of a usage pair. Note that stack depths are "simple" to compute here, as the L-stack is the first region to be allocated, therefore no x- or f- stacks are present.

When applying this visitor to a basic block, a couple validations are performed:
- any x or f stack load or store operation should fail,
- for any ops consuming values, the depth of the stack should be greater or equal than the amount of consumed vals. otherwise this basic block is in an unavoidable invalid stack state.\
The visitor consumes keeps track of depth, doing depth += #consumed - #produced for each visited op (save for load_l_stack() as an exception, which when copying produces an extra value).

TODO: this is an impl. detail


```py
@typing.override
    def visit_load_l_stack(self, load: mir.LoadLStack) -> None:
        if load.copy:
            self.depth += 1
```
WHY is there not the same for store_l_stack? Can I run into a load when visiting but not a store?


TODO: WHAT IS the local_id in this pair? check _find_usage_pairs()

TODO: explain `_copy_usage_pairs`


# The x-stack

Consider a given basic block, all of its successors, and all of the predecessors of its successors.
We define an `EdgeSet` as all edges who share a parent, a child, a sibling or a co-parent.

TODO: example of edge set

As an auxiliary way of collecting all of the information pertaining a block for the construction of the x-stack, we define a `BlockRecord`, that contains:
- a basic block (in this case MIR basic block)
- a list of all local references (abstract loads and stores)
- live-in vars (see VLA)
- live-out vars (see VLA)
- children, parents, co-parents and co-siblings




# The f-stack

The f-stack is initialized in the entry block and remains constant throughout a whole subroutine.
For f-stack building, we consider now a subroutine divided in (MIR) basic blocks.
Now, consider the set of all variables referenced in this subroutine (i.e. all instances of `AbstractStore` and `AbstractLoad`), sorted by their local id.
Note that, at the point of its construction, all L-stack and X-stack allocations have been allocated in code.
> [!NOTE] In case no variable defs or uses are found, that is, the whole subroutine has no instances of `AbstractStore` or `AbstractLoad` at this point, the f-stack is empty and no attempt to build it is performed.

In pseudocode:
```python
    for subroutine s:
        vars <= collect all variables in s
        if vars is empty:
            return ()
        block_0 = s.body[0]
        first...
        TODO: complete
```

```python
def f_stack_allocation(_ctx: ProgramMIRContext, subroutine: mir.MemorySubroutine) -> None:
    all_variables = _VariableCollector.collect(subroutine)
    if not all_variables:
        subroutine.pre_alloc = FStackPreAllocation.empty()
        return

    entry_block = subroutine.body[0]
    first_store_ops = _get_lazy_fstack(entry_block)
    unsorted_pre_allocate = [x for x in all_variables if x not in first_store_ops]
    subroutine.pre_alloc = _get_pre_alloc(subroutine, unsorted_pre_allocate)
    logger.debug(
        f"{subroutine.signature.name} f-stack entry: {subroutine.pre_alloc.allocate_on_entry}"
    )
    logger.debug(f"{subroutine.signature.name} f-stack on first store: {list(first_store_ops)}")

    entry_block.f_stack_in = subroutine.pre_alloc.allocate_on_entry
    entry_block.f_stack_out = [*entry_block.f_stack_in, *first_store_ops]
    # f-stack is initialized in the entry block and doesn't change after that
    for block in subroutine.body[1:]:
        block.f_stack_in = block.f_stack_out = entry_block.f_stack_out

    for block in subroutine.body:
        stack = Stack.begin_block(subroutine, block)
        for index, op in enumerate(block.mem_ops):
            match op:
                case mir.AbstractStore() as store:
                    insert = op in first_store_ops.values()
                    if insert:
                        assert block is entry_block
                        depth = stack.xl_height - 1
                    else:
                        depth = stack.fxl_height - stack.f_stack.index(store.local_id) - 1

                    block.mem_ops[index] = op = attrs_extend(
                        mir.StoreFStack,
                        store,
                        depth=depth,
                        frame_index=stack.fxl_height - depth - 1,
                        insert=insert,
                    )
                case mir.AbstractLoad() as load:
                    depth = stack.fxl_height - stack.f_stack.index(load.local_id) - 1
                    block.mem_ops[index] = op = attrs_extend(
                        mir.LoadFStack,
                        load,
                        depth=depth,
                        frame_index=stack.fxl_height - depth - 1,
                    )
            op.accept(stack)
        match block.terminator:
            case mir.RetSub() as retsub:
                block.terminator = attrs.evolve(
                    retsub, fx_height=len(stack.f_stack) + len(stack.x_stack)
                )
```

# Context
The context object in this layer provides very limited functionality. It is solely a store of the IR program and a list of all subroutine names and their associated IR code.
> [!NOTE]
> The implementation attempts to get a list of names that are the shortest possible.
 <!--TODO: WHEN is that not the case? example -->