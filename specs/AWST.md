# Abstract Wyvern Syntax Tree (entry to compiler)

The first intermediate representation in the compiler is a tree-like structure,
akin to an AST. It is built so that it "normalizes" what is expressible for the compiler backend.

## 

# Optimizations performed

# Validations performed

# Type system (WTypes)

🔹 1. Runtime Representation Kinds (_ValueType)

These describe how a value is physically represented at runtime.

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
🧱 3. Basic WTypes

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
