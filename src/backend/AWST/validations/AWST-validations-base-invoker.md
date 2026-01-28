<!-- DRAFT -->

# Base invoker validation
We traverse `SubroutineCallExpressions`, both inside and outside contracts, and keeping track of the contract class being currently visited.

To validate each of these, we skip `SubroutineID` target calls, as these are always valid (consider how a "free" module-level subroutine may be called from inside or outside a contract).

for targets that are instance methods of either a contract or any base class of a contract (`InstanceMethodTarget` and `SuperInstanceMethodTarget` respectively), if they are used outside of a contract method, then the compiler emits and `error` and compilation fails.

Finally, for call targets that are contract methods (`ContractMethodTarget`) we check that the call happens inside the context of a caller contract class and that the target base contract is either the caller contract class, or some parent contract in the current method resolution order.\
Otherwise, this is either another instance of a contract method being invoked outside of the context of a contract, or an invocation outside of the current hierarchy, and therefore is invalid (failing compilation with an `error`).