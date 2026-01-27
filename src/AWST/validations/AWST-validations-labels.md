# Labels validation
This validator centers around all `Label`s in an AWST, from now on refered to as "labels".\
It checks for label existance (`Goto` nodes can't target inexistant labels) and for label uniqueness (in the context of a given module-level `Subroutine` or an externally callable contract function, `ContractMethod`).

> [!NOTE]
> Implementation-wise, a `Label` in this layer is just a type rename of a python native `str`.

<!-- > [!NOTE] in TEAL, labels must be unique for a single contract file. However, the compiler will inject 'function context' into labels down the pipeline. TODO: improve explanation, is it correct? -->

We traverse the AWST, instantiating independant visitors for both every module level subroutine (`Subroutine` nodes found outside of any classes as module statements) and for every method inside of a contract.\

The function body is visited at the `Block` level, keeping track of labels associated to each block. If they are not unique (i.e. have been seen before in the same function context), an `error` is logged and compilation fails.\

Furthermore, the validator visits all `Goto` nodes at the block statement level. If a `Goto` node is found whose target label is not present in any block in the current function, an `error` is logged and compilation fails.