# Memory Intermediate Representation (MIR) layer

Up to this point, all allocations are _abstract_ (see the [relevant nodes in IR](IR.md#full-models-reference)) (not counting allocations explicitly defined by the user TODO: example).
In this layer, these _abstract_ allocations are made concrete by simulation and analysis of stack usage. 
<!-- In order to make the best possible usage of the native AVM stack [TODO: LINK to research] is used. -->

The general idea of this layer is the introduction of 3 logical stack regions, according to their relation to the overall control flow structure of the program.

The `l-stack` is, conceptually, the stack region used for intra-block allocations.\
The `x-stack` is the stack region used for inter-block allocations.\
Finally, the `f-stack` is used for the frame region ([explicit in the AVM](TODO_link_to_specs_AVM_section)). It is created in the entry block for each subroutine and maintained constant throughout the rest of basic blocks.

The following document attempts an exhaustive explanation of lowering from IR to MIR, stack allocation building, optimizations and validations performed at this step up to the output representation, that will be the input of the following [TEAL](TEAL.md) layer.
<!-- Having this region preserved in the stack is useful for most allocations that don't fit any of the other two regions.
TODO: improve explanation of f-stack -->

# MIR main algorithm

The following diagram shows the MIR pipeline schematically.
```mermaid
flowchart TD
    A[Start] --> B[lower_main_to_mir(IRProgram main)]
    B --> C[Iterate IRProgram subroutines]
    C --> D[lower_subroutine_to_mir(s)]
    D --> C

    C --> E{slot_allocation_strategy == dynamic?}
    E -- Yes --> F[Add special slot allocation subroutine]
    E -- No --> G[Skip]

    F --> H[Construct MIR.Program(kind, main, subroutines, slot_alloc)]
    G --> H

    H --> I[Apply globalStackAllocation(result)]
    I --> J[Return result]
```
> Link to reference implementation [here](TODO_LINK).

In the [building phase](#ir--mir-lowering), each subroutine in the IR [`Program`](IR.md#full-models-reference) input is lowered, starting with main as a special case.\
Then each non-main subroutine is built.\
At the end of the building process, and only in the case of `slot allocation strategy` set to `dynamic`, we build the slot allocation and append a special slot building subroutine at the end of the subroutine list.

Finally, the `global stack allocation` algorithm is performed, constructing each stack region according to the required optimization level, and materialising all `AbstractLoad` and `AbstractStore` operations in the process.

After this, we get the final output ready to continue down the pipeline to the [TEAL](TEAL.md) stage.

# IR => MIR lowering

## Slot allocation strategy
TODO: slot allocation expl. goes here


# The Stack visitor
The stack visitor is a visitor that processes ops and subsequently builds a stack with its regions explicitly. It applies the effects of the visiting op into the l-stack.

TODO: explain

# Global Stack Allocation
TODO: pseudocode
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

## Dead store removal (l-stack case)
<!-- TODO: isolate when this happens in l-stack construction and describe -->

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


### Dead store removal (x-stack, f-stack cases)

> [!Note] l-stack dead store removal occurs during l-stack allocation. The other two stack region cases are handled here.

```py
    if vla.is_dead_store(b):
        return a, mir.Pop(n=1, source_location=b.source_location)
```
If `b` constitutes a *dead store* (see [Variable Lifetime Analysis section](#variable-lifetime-analysis) above), then it may be replaced by a `MIR.Pop` operation to pop the top value of the stack, as the last product of `a` is stored but never used again, and can therefore be safely discarded.
The resulting pattern is then `(a, MIR.Pop(n=1))`.\

TODO: illustrative example

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

# Validations performed

## Unexpected nodes
<!-- TODO: for this section, explain briefly what each of these should have turned into at this stage -->
There is a series of [IR](IR.md#full-models-reference) nodes that should not be found during construction of the MIR program. This constitutes an implicit validation: by finding one of these at this stage, we know that the pipeline has failed at some point during the previous stage, and thus emit an `error` and fail compilation upon attempted visitation.

- `CompiledContractReference` and `CompiledLogicSigReference`
<!-- TODO: explain what they turned into and link -->

- `ValueTuple` nodes should have been split into their constituting values
<!-- TODO: link and double check -->

- `ItxnConstant`, `SlotConstant`, and `InnerTransactionField`

- `BytesEncode` and `DecodeBytes` should have been resolved in the prior stage

- Box write and read operations (`BoxWrite` and `BoxRead`)

- `ArrayLength`, `ExtractValue` and `ReplaceValue`

- [`Phi`](IR.md#full-models-reference) and `PhiArgument` nodes. These should have been either resolved during IR building by virtue of being [trivial](IR.md#trivial-phi), or after getting out of SSA during [destructuring](IR.md#ssa-destructuring) for non-trivial ones.

<!-- TODO: complete all validations, implicit and explicit -->

## F-Stack pre-allocation AVM Type
The f-stack pre-allocation variables should all be either `uint64` or `bytes` typed. `any` AVM types will raise an error at this stage.
<!-- TODO: link to where this validation happen and explain a bit better why this is -->
<!-- TODO: after construction? -->

# Full models reference
In this section we provide the [full set of nodes expressible in MIR](https://github.com/algorandfoundation/puya/blob/main/src/puya/mir/models.py).

## Top-Level Concepts

### **`BaseOp`**
Abstract base class for all MIR operations. Tracks:
- `consumes`: how many values are popped from the l-stack
- `produces`: local IDs pushed to the l-stack
- `error_message`, `source_location`
Defines the visitor dispatch via `accept()`.

---

## Literal & Special Value Ops

### **`Int`**
Pushes an integer literal onto the stack.

### **`Byte`**
Pushes a byte literal using a given AVM byte encoding.

### **`Undefined`**
Represents an undefined value of a specific AVM type.

### **`TemplateVar`**
Represents a compile-time template variable (`int` or `byte`).

### **`Address`**
Pushes a literal Algorand address.

### **`Method`**
Pushes a method selector/string literal for ABI dispatch.

---

## Abstract Memory Ops

### **`StoreOp`**
Abstract superclass for all memory store operations.

### **`LoadOp`**
Abstract superclass for all memory load operations.

---

## Abstract Storage Ops
These ops. are the drivers of transformation in this compiler layer, as they will be replaced by specific region storage ops. as a result of the MIR processing stages (or removed by any of the optimization passes in between).

### **`AbstractStore`**
Consumes stack top and stores into a local variable.

### **`AbstractLoad`**
Loads a local variable by ID and pushes it onto the stack.

---

## Local Variable (L-Stack) Ops

### **`StoreLStack`**
Stores a value into the L-stack at a given depth.
May optionally push a copy to the stack.\
If producing a copy (copy flag set to true), then the
`produce` value must be equal to 1, else it must be equal to 0.

### **`LoadLStack`**
Loads a value from the L-stack at a computed depth.
May optionally produce a copy to the stack.\
If producing a copy (copy flag set to true), then the
`produce` value must be equal to 1, else it must be equal to 0.

---

## X-Stack (Expression Stack) Ops

### **`StoreXStack`**
Stores a value into the X-stack at a given depth.

### **`LoadXStack`**
Loads a value from the X-stack.

---

## F-Stack (Frame Stack) Ops

### **`StoreFStack`**
Stores into a frame stack slot (`frame_index`).

### **`LoadFStack`**
Loads a value from a frame stack slot.

---

## Parameter Load/Store

### **`LoadParam`**
Loads a subroutine parameter, pushing its value.

### **`StoreParam`**
Stores a value into a parameter slot.

---

## Stack Manipulation

### **`Pop`**
Removes `n` values from the stack.

---

## Subroutine Calls

### **`CallSub`**
Calls a subroutine:
- consumes parameter count
- produces return count
- tracks target name

### **`IntrinsicOp`**
Represents a TEAL intrinsic operation (non-memory, non-control).  
Holds opcode + immediates.

---

## Control Flow Ops (Terminators)

### **`ControlOp`**
Base class for all block terminators (branches, exits, returns).

### **`RetSub`**
Return from a subroutine.

### **`ProgramExit`**
Return from the main program.

### **`Err`**
Program exit with an error.

### **`Goto`**
Unconditional branch.

### **`ConditionalBranch`**
Branch on zero/nonzero: bz <zero_target> ; b <nonzero_target>


### **`Switch`**
Switch-table branch with default target.

### **`Match`**
Pattern-match branching.  
Consumes N+1 stack values.

---

## IR Structural Nodes

### **`MemoryBasicBlock`**
A basic block containing:
- memory ops
- a terminator
- predecessors
- stack-shape metadata (`x_stack_in/out`, `f_stack_in/out`)
- name and ID

### **`Parameter`**
Represents a parameter of a subroutine: name, local ID, type.

### **`Signature`**
Full subroutine signature (parameters + return types).

### **`FStackPreAllocation`**
Determines which local variables must be preallocated in the F-stack.

### **`MemorySubroutine`**
A subroutine in Memory IR. It contains the following fields:
- `id`: a unique string identifying this subroutine in the context of the whole program.
- `is_main`: a boolean flag indicating if this is the main subroutine or not.
- `signature`: a structured signature field composed by the function name, its input parameters, and the return type/s (in AVM types).
- `body`: a sequence of basic blocks modeled as [`MemoryBasicBlock`](#memorybasicblock).
- Optional fields:
    - `pre_alloc`: an *optional* field containing the f-stack allocation (see the [relevant section above](#the-f-stack)).
    - `source_location`: an *optional* field containing a representation of the source location for this subroutine (line and column number for start and end of the code block in the source file).

### **`SlotAllocation`**
Describes scratch slot allocation for TEAL lowering.

### **`Program`**
Represents a full MIR program:
- kind (approval/clear/global/etc.)
- main subroutine
- user subroutines
- AVM version
- slot allocation

---


## AbstractStore
Store operations for a `local id` identified variable, which have not yet been materialized to a specific register (or in this case, a stack location). After the stack passes in this layer, no abstract stores should remain before TEAL lowering (see validation procedures). Resolving these is one half of the main objective of this layer.

## AbstractLoad
Load operations for a `local id` identified variable, which have not yet been materialized. After the stack passes in this layer, no abstract loads should remain before TEAL lowering (see validation procedures). Resolving these is the other half of the main objective of this layer.


# Appendix 

## MIR syntax


## MIR outputs

There are a series of intermediate outputs written in human readable format throughout this stage.
The first output is tagged `"build"`, and is output right after lowering has happened, but before any of the stack regions have been allocated.\


TODO: link every location where the output is made.

500. => stack operations get a stack description.
Stack description for l-stack operations is just as is, with comma separated values, where the last values are the top of the stack and the first values are the bottom of the stack.

Stack description for P, F, X come after (...)