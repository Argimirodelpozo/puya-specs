# MIR outputs

There are a series of intermediate outputs written in human readable format throughout this stage.
The first output is tagged `"build"`, and is output right after lowering has happened, but before any of the stack regions have been allocated.

[Link to output in reference implementation](TODO_LINK)

```
TODO: pick example
```

This representation is one step closer to valid TEAL code than the last representation in IR. Some crucial changes in between these stages:

- Register assignments (`let` statements in the textual representation of the IR) have been materialized, being split into an instruction pushing the right hand side to the stack (lets call this value `n`), followed by an `AbstractStore` (represented here as `v-store n`).\
An example of this could be:
```c
//In the destructured IR
main algopy.arc4.ARC4Contract.approval_program:
    block@0: // L1
        let tmp%0#0: bool = (txn ApplicationID)
        goto tmp%0#0 ? block@2 : block@1
```

```c
//In the MIR (.build)
// Op                                                                                             Stack (out)
// algopy.arc4.ARC4Contract.approval_program() -> uint64:
subroutine main:
    main_block@0:
        txn ApplicationID                                                                         tmp%0#0
        v-store tmp%0#0
        v-load tmp%0#0                                                                            tmp%0#0
        bz main_call___init__@1 ; b main_after_if_else@2
```
Note also the presence of a `Stack (out)` column, used to show the state of the virtual/abstract (as in, not yet materialized) stack _after_ the execution of every opcode.\
Finally, IR [`Goto` nodes](TODO_LINK) are lowered into MIR as `ConditionalBranch` instructions followed by `Branch` instructions.
<!-- TODO: review these model names and reword accordingly -->

Something similar happens with uses in the original IR, now transformed in `AbstractLoad` nodes. These are represented as `v-load k` in the `.build` output (where `k` is the value being loaded).


Besides the usage of these new primitives, the first MIR representation is very similar to the last IR representation (`.destructured`, see the [IR output reading guide](TODO_LINK) or the last step in the [IR building pipeline](TODO_LINK)).


<!-- TODO: link every location where the output is made. -->

500. => stack operations get a stack description.
Stack description for l-stack operations is just as is, with comma separated values, where the last values are the top of the stack and the first values are the bottom of the stack.

Stack description for P, F, X come after (...)
<!-- TODO: complete -->