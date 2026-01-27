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