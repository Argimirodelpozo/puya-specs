# Abstract Wyvern Syntax Tree (entry to compiler)

The first intermediate representation in the compiler is a tree-like structure, akin to an AST. It is built so that it "normalizes" what is expressible for the compiler backend.
<!-- TODO: improve general description -->


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

## Labels validation
This validator centers around all `Label`s in an AWST, from now on refered to as "labels".\
It checks for label existance (`Goto` nodes can't target inexistant labels) and for label uniqueness (in the context of a given module-level `Subroutine` or an externally callable contract function, `ContractMethod`).

> [!NOTE]
> Implementation-wise, a `Label` in this layer is just a type rename of a python native `str`.

<!-- > [!NOTE] in TEAL, labels must be unique for a single contract file. However, the compiler will inject 'function context' into labels down the pipeline. TODO: improve explanation, is it correct? -->

We traverse the AWST, instantiating independant visitors for both every module level subroutine (`Subroutine` nodes found outside of any classes as module statements) and for every method inside of a contract.\

The function body is visited at the `Block` level, keeping track of labels associated to each block. If they are not unique (i.e. have been seen before in the same function context), an `error` is logged and compilation fails.\

Furthermore, the validator visits all `Goto` nodes at the block statement level. If a `Goto` node is found whose target label is not present in any block in the current function, an `error` is logged and compilation fails.

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


# Type system (WTypes)

## Runtime Representation Kinds (`_ValueType`)

These describe how a value is physically represented at runtime. The enumeration is:
- uint64
- bytes
- reference
- composite
- none

Two of these have a correspondence with AVM types (uint64 and bytes) while the others do not.
TODO: link to definition



ValueType	Meaning
uint64	Value is a single AVM uint64 stack element.
bytes	Value is a single AVM byte slice.
reference	Represents a pointer-like reference (arrays, structs, etc.).
composite	Value consists of multiple underlying runtime values.
none	No runtime value (e.g., void or static type).

value_type.avm_type controls whether the type maps to AVM int or bytes stack slots.

🔹 2. Persistability Semantics (_TypeSemantics)

Describes where the type may live (stack only, ARC-4 / state, composite, reference).

Semantics	ValueType	Persistable	Notes
persistable_uint64	uint64	yes	e.g., uint64, asset, application
persistable_bytes	bytes	yes	e.g., string, account, biguint
maybe_persistable_composite	composite	depends	Composite types may become persistable if all fields are.
ephemeral_uint64	uint64	no	Used for ephemeral runtime-only values.
ephemeral_composite	composite	no	Temporary composite values.
ephemeral_reference	reference	no	Reference-type containers.
static_type	none	no	Types with no runtime value (void, type-only).


## Basic WTypes

Primitive, single-slot, and simple persistable types.

Type	Description
void	No value; used for statements or uninhabited types.
bool	Boolean stored as AVM uint64 (0/1).
uint64	AVM’s native integer type.
biguint	Arbitrarily large unsigned integer (byte-encoded).
string	UTF-8 string (persistable bytes).
asset	Asset ID (uint64).
account	Account address (bytes).
application	Application ID (uint64).
state_key	App state key (bytes).
box_key	Box key (bytes).
uint64_range	Static-only type used for range iteration.

All basic WTypes have immutable = True.

🧩 4. Bytes Types
BytesWType(length: int | None)

Represents a byte string with optional fixed length.

Always persistable and immutable.

Examples:

bytes
bytes[16]
bytes[32]

🔃 5. Enumeration
WEnumeration(sequence_type)

Represents enumerate(x) types from the language.

Static-only type (no runtime value).

Example:

enumerate_uint64
enumerate_bytes

🚦 6. Transaction-Related Types

Types referring to transactions in the group or inner transactions.

WGroupTransaction(transaction_type)

Represents gtxn references.

WInnerTransaction(transaction_type)

Represents an inner transaction object.

WInnerTransactionFields(transaction_type)

Represents the inner transaction field dictionary.

All three:

Are ephemeral

Composite or uint64 depending on subtype

Have names like:

group_transaction_pay

inner_transaction_fields_axfer

inner_transaction_app

📦 7. Reference Types
ReferenceArray(element_type)

An array of unknown size, implemented as a reference.

Not persistable.

Elements must be immutable and not void.

Example:

ref_array<uint64>
ref_array<bytes[32]>

🧱 8. Tuple Types
WTuple(types: tuple[WType], names: Optional[tuple[str]])

Fixed-size heterogeneous product type.

Aggregate type.

Persistable iff all fields persistable.

Immutable iff all fields immutable.

Supports:

Named fields (struct-like)

Unnamed tuples (positional)

Example names:

tuple<uint64,bytes[16]>
tuple<x:uint64,y:uint64>

🔐 9. ARC-4 Type System

All ARC-4 types are persistable and encode to strict ARC-4 ABI formats.

9.1 Basic ARC-4 Types
ARC4Type(name="arc4.bool")

ARC-4 booleans (0x00 / 0x01).

9.2 ARC-4 Integers
ARC4UIntN(n)

Unsigned integer of exactly n bits, where:

n divisible by 8

8 ≤ n ≤ 512

Examples:

arc4.uint8
arc4.uint256
arc4.uint512

9.3 ARC-4 Fixed-point
ARC4UFixedNxM(n, m)

n bits for integer portion

m decimal precision bits

n must be multiple of 8

1 ≤ m ≤ 160

Example:

arc4.ufixed128x10

9.4 ARC-4 Tuples
ARC4Tuple(types)

Aggregate of persistable types.

Example:

arc4.tuple<uint64,bytes[32]>


All members must be persistable.

9.5 ARC-4 Arrays
ARC4DynamicArray(element_type)

Dynamic array of persistable elements.

ARC4StaticArray(element_type, array_size)

Fixed-length array.

Examples:

arc4.dynamic_array<uint64>
arc4.static_array<bytes[32], 4>

9.6 ARC-4 Structs
ARC4Struct(fields, frozen)

Named fields, like a proper struct.

Must be persistable (all fields persistable).

frozen=True makes it immutable if all fields are immutable.

Example:

arc4.struct { a:uint64, b:bytes[16] }

9.7 ARC-4 Aliases
Alias	Expands to
arc4.byte	arc4.uint8
arc4.string	dynamic array of bytes
arc4.address	static array of 32 bytes


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