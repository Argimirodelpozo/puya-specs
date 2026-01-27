## ABI method name validation
> [!INFO]
> The regular expression for an ARC4 compliant method names is
>```regexp
>"^[_A-Za-z][A-Za-z0-9_]*$"
>```
>you may refer to the [ARC4 specification](TODO_LINK) for more details.

We traverse the AWST, looking for `ContractMethod` nodes and `MethodConstant` nodes.\

For the first one, if it constitues an ARC4 method, then it must have an `ARC4ABIMethodConfig` node associated.\
We validate its name against the ARC4 regular expresion.

For the second one, we consider the `MethodSignature` node associated to it, and validate the name in this signature against the aforementioned regular expression.

Note that the compiler emits a `warning` log for each instance found of a method name that is not ARC4 compliant, but it will not fail compilation solely because of this (unless compiling with the `--treat-warnings-as-errors` option enabled).