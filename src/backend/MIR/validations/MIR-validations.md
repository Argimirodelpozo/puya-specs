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