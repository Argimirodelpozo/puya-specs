# Validations performed
<!-- TODO: intro text -->

## `Subroutine` validation (with SSA)

## Validate with SSA
Basic SSA validation. For each basic block in the subroutine, we collect all assigned registers into a set. This encompasses registers that are target of an `Assignment`, and registers assigned to a `phi` node.\

If at any point we find a repeat, that means a basic SSA property is broken (a register is assigned twice) and therefore an `InternalError` is raised as the IR is in an invalid state.


## After [destructuring](#ssa-destructuring)
<!-- TODO: all validators here (from IR/validate) -->



# Validate module artifact (pos-destructuring validation)
<!-- for validator_cls in (
        OpRunModeValidator,
        MinAvmVersionValidator,
        ITxnResultFieldValidator,
        CompileReferenceValidator,
        SlotReservationValidator,
        NoInfiniteLoopsValidator, -->
<!-- TODO: link to code for this -->

## Op run mode validator
Very simple instruction-by-instruction visitor based validator. Given an `artifact`, be it a `LogicSignature` or a `Contract`.

Every single op is visited (note that, by virtue of our node structure, only `Intrinsic` operations need to be considered in this analysis), and a validation of the run mode the instruction is allowed to run in is performed.    

Any conflict between the artifact type and the run mode of an instruction used in it here causes a failed validation and compilation error.
<!-- (TODO: link code) -->

## min avm version validator
Instruction by instruction visitor based validator.\
We take the program avm version (version declared at the beginning of the program with a `#pragma version` instruction, see TODOLINKSPECS).\
Similarly, each visited instruction has a version in which it was introduced. If this minimum AVM version that supports the given op. is higher than the program declared version...
<!-- TODO: finish
TODO: comment on variants!
TODO: link code -->

## itxn result field validator
TODO: how do we know its not constant here?

## compile reference validator


## slot reservation validator