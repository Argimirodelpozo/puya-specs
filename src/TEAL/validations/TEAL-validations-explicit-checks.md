<!-- DRAFT -->

# _Explicit check_ invariance validation
After lowering and before running optimization passes, an initial set of explicit checks is collected (see above in the build section). An explicit check is an `Assert` or `Err` TEAL model that has been marked as such from its start in the pipeline because it comes explicitly from the user code, and thus will have an internal flag set to `True` when built.
The collection algorithm simply tallies the amount of explicit checks by subroutine.\
After all [optimization passes](#optimizations-performed) are performed, explicit checks are collected again.
A decrease in explicit checks for a given subroutine means an optimization has been semantically destructive for the purpose of this validation, and will thus fail compilation.
> [!NOTE] the word _decrease_ hides a subtlety here; consider that ops may be duplicated during [inlining](#optimizations-performed), and thus there could be _more_ explicit checks after optimization than before.