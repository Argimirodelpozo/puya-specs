# Memory Intermediate Representation (MIR) layer

Up to this point, all allocations are _abstract_ (see [TODO: LINK to IR reference]) (not counting allocations explicitly defined by the user TODO: example).
In this layer, these _abstract_ allocations are made concrete by the usage of the stack. In order to make the best possible usage of the native AVM stack [TODO: LINK to research] is used.

The general idea of this layer is the introduction of 3 logical stack regions, according to their relation to the overall control flow structure of the program./
The `l-stack` is, conceptually, the stack region used for intra-block allocations.\
The `x-stack` is the stack region used for inter-block allocations.\
Finally, the `f-stack` is used for the frame region (explicit in the AVM). It is created in the entry block and maintained constant throughout the rest of basic blocks.
Having this region preserved in the stack is useful for most allocations that don't fit any of the other two regions.
TODO: improve explanation of f-stack

# main building algorithm
```py
def program_ir_to_mir(context: ArtifactCompileContext, program_ir: ir.Program) -> models.Program:
   ctx = attrs_extend(ProgramMIRContext, context, program=program_ir)

    mir_main = _lower_subroutine_to_mir(ctx, program_ir.main, is_main=True)
    mir_subroutines = [
        _lower_subroutine_to_mir(ctx, ir_sub, is_main=False) for ir_sub in program_ir.subroutines
    ]
    match program_ir.slot_allocation.strategy:
        case SlotAllocationStrategy.none:
            mir_allocation = None
        case SlotAllocationStrategy.dynamic:
            mir_allocation = _build_slot_allocation(program_ir)
            mir_subroutines.append(build_new_slot_sub(mir_allocation.allocation_slot))
        case unexpected:
            typing.assert_never(unexpected)
    result = models.Program(
        kind=program_ir.kind,
        main=mir_main,
        subroutines=mir_subroutines,
        avm_version=program_ir.avm_version,
        slot_allocation=mir_allocation,
    )
    if ctx.options.output_memory_ir:
        output_memory_ir(ctx, result, qualifier="build")
    global_stack_allocation(ctx, result)
    return result
```
Each subroutine in the ir program is lowered, starting with main as a special case.
Then for each subroutine, each basic block is lowered.
Afterwards, in the case of `slot allocation strategy` set to `dynamic`, we build the slot allocation and append a special slot building subroutine.
Finally, the `global stack allocation` is performed.

There is a series of IR nodes that are unexpected at this stage and will thus fail upon attempted compilation:
- Phi Nodes (TODO: link)
- TODO: complete


# The Stack visitor
The stack visitor is a visitor that processes ops and subsequently builds a stack with its regions explicitly. It applies the effects of the visiting op into the l-stack.

TODO: explain


# The l-stack
The L-stack allocations are built by applying [Koopman's algorithm](TODO:LINK).

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

TODO: question: 

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

> [!NOTE] In case no variable defs or uses are found, that is, the whole subroutine has no AS or ALs, the F-stack will be empty.




> Divergence from the original paper:
> No usage of p-stack and e-stack

# Context
The context object in this layer provides very limited functionality. It is solely a store of the IR program and a list of all subroutine names and their associated IR code.
> [!NOTE]
> The implementation attempts to get a list of names that are the shortest possible.
 <!--TODO: WHEN is that not the case? example -->


## Lowering from (destructured) IR

### Builder

## Koopman's Algorithm

## Bailey's Algorithm

## Global stack allocator (not implemented)


# Optimizations performed

We say that a variable is _live_ if it holds a value that will/might be used in the future.
(https://www.classes.cs.uchicago.edu/archive/2004/spring/22620-1/docs/liveness.pdf)
VLA (Variable Lifetime Analysis) is used throughout many of the optimizations performed.
Consider now the following sets:

`use[n]`: the set of variable uses in the node `n` of the control flow graph. A variable use is a right hand side appearence of a variable's identifier.

`def[n]`: the set of variable definitions in the node `n` of the control flow graph. A variable definition is a left hand side appearence of a variable's identifier.

`in[n]`: the set of variables that are live-in at node `n`. A variable (temp) `a` is live-in at node `n` if it is used at `n` (`a` $\in$ `use[n]`), or if there is a path from `n` to a node that uses `a` that does not contain a definition of (assignment to) `a`. We write `a` $\in$ `in[n]`.

`out[n]`: the set of variables that are live-out at node `n`. A variable `a` is live-out at node `n` if it is live-in at one of the successors of `n`. We write `a` $\in$ `out[n]`.

The sets satisfy the following two equations:

`in[n]` $=$ `use[n]` $\cup$ (`out[n]` $-$ `def[n]`)

`out[n]` $=$ $\cup$ \{ `in[s]` $|$ `s` $\in$ `succ[n]` \}

The relevant nodes for this analysis are `AbstractStore` and `AbstractLoad`.


## Peephole optimizations
We define a peephole window of size 2, meaning the peephole optimizer will consider _pairs_ of operations inside each basic block (see `MemoryBasicBlock` below).
The optimizer attempts to:
- eliminate redundant loads/stores
- fold store=>load patterns
- rename produced local IDs
- remove unnecessary stores
- replace dead stores with pops
- remove load/store pairs that cancel out




# Validations performed

TODO: proofread (AI assisted)

# Full models reference
In this section we provide the full set of nodes expressable in MIR.

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

### **`Comment`**
A no-op representing an inline comment (debug/pretty-print only).

---

## Abstract Memory Ops

### **`StoreOp`**
Abstract superclass for all memory store operations.

### **`LoadOp`**
Abstract superclass for all memory load operations.

---

## Local Variable (L-Stack) Ops

### **`AbstractStore`**
Consumes stack top and stores into a local variable.

### **`AbstractLoad`**
Loads a local variable by ID and pushes it onto the stack.

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
A subroutine in Memory IR:
- id
- signature
- blocks
- pre-allocation info

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

## Utilities

### **`produces_from_desc`**
Given a description and arity, builds a tuple of produced local IDs:
- size > 1 → `{desc}.0`, `{desc}.1`, ...
- size = 1 → `{desc}`
- size = 0 → `()`

---



## AbstractStore
Store operations for a `local id` identified variable, which have not yet been materialized to a specific register (or in this case, a stack location). After the stack passes in this layer, no abstract stores should remain before TEAL lowering (see validation procedures). Resolving these is one half of the main objective of this layer.

## AbstractLoad
Load operations for a `local id` identified variable, which have not yet been materialized. After the stack passes in this layer, no abstract loads should remain before TEAL lowering (see validation procedures). Resolving these is the other half of the main objective of this layer.

# Appendix: MIR outputs and reading guide

There are a series of outputs. TODO: link every location where the output is made.

500. => stack operations get a stack description.
Stack description for l-stack operations is just as is, with comma separated values, where the last values are the top of the stack and the first values are the bottom of the stack.

Stack description for P, F, X come after (...)