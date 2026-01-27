<!-- DRAFT -->

# Validations performed

The following is a series of validations performed on a finished AWST. They are generally simple in nature, and implemented as visitors for specific [nodes](#awst-node-reference). They are run singularly, sequentially, and in the order in which they are presented in this specification.

> You may find the function that performs AWST validations [here](TODO_LINK) in the Puya implementation.

> [!NOTE] there are also several validators for specific fields in AWST node constructions. For the sake of clarity, we include those small, specific subroutines in each relevant node's section in the [full node reference](#awst-node-reference).

## Inner transactions (general) validation
<!-- TODO: explain -->

## Inner transactions in loop validation
<!-- TODO: explain -->

## Stale inner transactions validation
<!-- TODO: explain -->

## Base invoker validation
We traverse `SubroutineCallExpressions`, both inside and outside contracts, and keeping track of the contract class being currently visited.\ 

To validate each of these, we skip `SubroutineID` target calls, as these are always valid (consider how a "free" module-level subroutine may be called from inside or outside a contract).\

for targets that are instance methods of either a contract or any base class of a contract (`InstanceMethodTarget` and `SuperInstanceMethodTarget` respectively), if they are used outside of a contract method, then the compiler emits and `error` and compilation fails.\

Finally, for call targets that are contract methods (`ContractMethodTarget`) we check that the call happens inside the context of a caller contract class and that the target base contract is either the caller contract class, or some parent contract in the current method resolution order.\
Otherwise, this is either another instance of a contract method being invoked outside of the context of a contract, or an invocation outside of the current hierarchy, and therefore is invalid (failing compilation with an `error`).

## Storage types validation
<!-- TODO: explain -->

## Immutability validation
<!-- TODO: explain -->

## ABI method name validation
> [!INFO]
> The regular expression for ARC4 compliant method names is
>```regexp
>"^[_A-Za-z][A-Za-z0-9_]*$"
>```
>you may refer to the [ARC4 specification](TODO_LINK) for more details.

We traverse the AWST, looking for `ContractMethod` nodes and `MethodConstant` nodes.\

For the first one, if it constitues an ARC4 method, then it must have an `ARC4ABIMethodConfig` node associated.\
We validate its name against the ARC4 regular expresion.

For the second one, we consider the `MethodSignature` node associated to it, and validate the name in this signature against the aforementioned regular expression.

Note that the compiler emits a `warning` log for each instance found of a method name that is not ARC4 compliant, but it will not fail compilation solely because of this (unless compiling with the `--treat-warnings-as-errors` option).


# AWST Node reference

The following section covers every single node expressable in the AWST.\
For each, we also give its representation in the `awst` human readable output file, as well as any specific field validations performed.


## Base

- `Node`
- `Statement` – base class for all statement nodes
- `Expression` – base class for all expression nodes
- `RootNode` – base class for top-level compilation units
- `Function` – base class for subroutines / methods
- `ContractMemberNode` – base class for contract-scoped members

---

## Statement Nodes

- `ExpressionStatement`
- `Block`
- `Goto`
- `IfElse`
- `Switch`
- `WhileLoop`
- `LoopExit`
- `LoopContinue`
- `ReturnStatement`
- `AssignmentStatement`
- `UInt64AugmentedAssignment`
- `BigUIntAugmentedAssignment`
- `BytesAugmentedAssignment`
- `ForInLoop`

---

## Literal / Constant Expression Nodes

- `AssertExpression`
- `IntegerConstant`
- `DecimalConstant`
- `BoolConstant`
- `BytesConstant`
- `StringConstant`
- `VoidConstant`
- `TemplateVar`
- `MethodConstant`
- `AddressConstant`

(Plus alias type: `CompileTimeConstantExpression`)

---

## ARC4 Encoding / Conversion Expressions

- `ARC4Encode`
- `ARC4Decode`
- `ARC4FromBytes`
- `ConvertArray`
- `NewArray`
- `NewStruct`
- `ARC4Router`

---

## Array / Tuple / Struct Expressions

- `Copy`
- `ArrayConcat`
- `ArrayExtend`
- `ArrayPop`
- `ArrayReplace`
- `ArrayLength`
- `TupleExpression`
- `TupleItemExpression`
- `NamedTupleExpression`
- `FieldExpression`
- `IndexExpression`
- `SliceExpression`
- `IntersectionSliceExpression`

---

## Storage & State Expressions

- `AppStateExpression`
- `AppAccountStateExpression`
- `BoxPrefixedKeyExpression`
- `BoxValueExpression`
- `StorageExpression` (alias:
  `AppStateExpression | AppAccountStateExpression | BoxValueExpression`)
- `StateGet`
- `StateGetEx`
- `StateExists`
- `StateDelete`

---

## Inner-Transaction Expressions

- `CreateInnerTransaction`
- `UpdateInnerTransaction`
- `SetInnerTransactionFields`
- `SubmitInnerTransaction`
- `InnerTransactionField`
- `GroupTransactionReference`

---

## Misc Core Expressions

- `SizeOf`
- `IntrinsicCall`
- `VarExpression`
- `CheckedMaybe`
- `SingleEvaluation`
- `ReinterpretCast`
- `ConditionalExpression`
- `AssignmentExpression`
- `CommaExpression`
- `Emit`
- `Range`
- `Enumeration`
- `Reversed`
- `StateGet`
- `StateGetEx`
- `StateExists`
- `StateDelete`
- `CompiledContract`
- `CompiledLogicSig`

---

## Comparison Expressions

- `NumericComparisonExpression`
- `BytesComparisonExpression`
- `BooleanBinaryOperation`
- `Not`

---

## Integer / Bytes Unary & Binary Expressions

- `UInt64UnaryOperation`
- `UInt64PostfixUnaryOperation`
- `BigUIntPostfixUnaryOperation`
- `BytesUnaryOperation`
- `UInt64BinaryOperation`
- `BigUIntBinaryOperation`
- `BytesBinaryOperation`

---

## Call / Library Expressions

- `SubroutineCallExpression`
- `PuyaLibCall`

---

## Subroutines & Contracts (Root Nodes)

- `Subroutine` (implements `Function`, `RootNode`)
- `ContractMethod` (implements `Function`, `ContractMemberNode`)
- `AppStorageDefinition` (implements `ContractMemberNode`)
- `LogicSignature` (implements `RootNode`)
- `Contract` (implements `RootNode`)

---

# Appendix: Output reading guide
<!-- TODO: complete with non-json AWST output reading pointers -->