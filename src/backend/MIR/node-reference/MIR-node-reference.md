
# Full models reference
In this section we provide the [full set of nodes expressible in MIR](https://github.com/algorandfoundation/puya/blob/main/src/puya/mir/models.py).

## Top-Level Concepts

### **`BaseOp`**
Abstract base class for all MIR operations. Tracks:
- `consumes`: how many values are popped from the l-stack
- `produces`: local IDs pushed to the l-stack
- `error_message`, `source_location`
Defines the visitor dispatch via `accept()`.

---

## Literal & Special Value Ops

### **`Int`**
Pushes an integer literal onto the stack.

### **`Byte`**
Pushes a byte literal using a given AVM byte encoding.

### **`Undefined`**
Represents an undefined value of a specific AVM type.

### **`TemplateVar`**
Represents a compile-time template variable (`int` or `byte`).

### **`Address`**
Pushes a literal Algorand address.

### **`Method`**
Pushes a method selector/string literal for ABI dispatch.

---

## Abstract Memory Ops

### **`StoreOp`**
Abstract superclass for all memory store operations.

### **`LoadOp`**
Abstract superclass for all memory load operations.

---

## Abstract Storage Ops
These ops. are the drivers of transformation in this compiler layer, as they will be replaced by specific region storage ops. as a result of the MIR processing stages (or removed by any of the optimization passes in between).

### **`AbstractStore`**
Consumes stack top and stores into a local variable.

### **`AbstractLoad`**
Loads a local variable by ID and pushes it onto the stack.

---

## Local Variable (L-Stack) Ops

### **`StoreLStack`**
Stores a value into the L-stack at a given depth.
May optionally push a copy to the stack.\
If producing a copy (copy flag set to true), then the
`produce` value must be equal to 1, else it must be equal to 0.

### **`LoadLStack`**
Loads a value from the L-stack at a computed depth.
May optionally produce a copy to the stack.\
If producing a copy (copy flag set to true), then the
`produce` value must be equal to 1, else it must be equal to 0.

---

## X-Stack (Expression Stack) Ops

### **`StoreXStack`**
Stores a value into the X-stack at a given depth.

### **`LoadXStack`**
Loads a value from the X-stack.

---

## F-Stack (Frame Stack) Ops

### **`StoreFStack`**
Stores into a frame stack slot (`frame_index`).

### **`LoadFStack`**
Loads a value from a frame stack slot.

---

## Parameter Load/Store

### **`LoadParam`**
Loads a subroutine parameter, pushing its value.

### **`StoreParam`**
Stores a value into a parameter slot.

---

## Stack Manipulation

### **`Pop`**
Removes `n` values from the stack.

---

## Subroutine Calls

### **`CallSub`**
Calls a subroutine:
- consumes parameter count
- produces return count
- tracks target name

### **`IntrinsicOp`**
Represents a TEAL intrinsic operation (non-memory, non-control).  
Holds opcode + immediates.

---

## Control Flow Ops (Terminators)

### **`ControlOp`**
Base class for all block terminators (branches, exits, returns).

### **`RetSub`**
Return from a subroutine.

### **`ProgramExit`**
Return from the main program.

### **`Err`**
Program exit with an error.

### **`Goto`**
Unconditional branch.

### **`ConditionalBranch`**
Branch on zero/nonzero: bz <zero_target> ; b <nonzero_target>


### **`Switch`**
Switch-table branch with default target.

### **`Match`**
Pattern-match branching.  
Consumes N+1 stack values.

---

## IR Structural Nodes

### **`MemoryBasicBlock`**
A basic block containing:
- memory ops
- a terminator
- predecessors
- stack-shape metadata (`x_stack_in/out`, `f_stack_in/out`)
- name and ID

### **`Parameter`**
Represents a parameter of a subroutine: name, local ID, type.

### **`Signature`**
Full subroutine signature (parameters + return types).

### **`FStackPreAllocation`**
Determines which local variables must be preallocated in the F-stack.

### **`MemorySubroutine`**
A subroutine in Memory IR. It contains the following fields:
- `id`: a unique string identifying this subroutine in the context of the whole program.
- `is_main`: a boolean flag indicating if this is the main subroutine or not.
- `signature`: a structured signature field composed by the function name, its input parameters, and the return type/s (in AVM types).
- `body`: a sequence of basic blocks modeled as [`MemoryBasicBlock`](#memorybasicblock).
- Optional fields:
    - `pre_alloc`: an *optional* field containing the f-stack allocation (see the [relevant section above](#the-f-stack)).
    - `source_location`: an *optional* field containing a representation of the source location for this subroutine (line and column number for start and end of the code block in the source file).

### **`SlotAllocation`**
Describes scratch slot allocation for TEAL lowering.

### **`Program`**
Represents a full MIR program:
- kind (approval/clear/global/etc.)
- main subroutine
- user subroutines
- AVM version
- slot allocation

---


## AbstractStore
Store operations for a `local id` identified variable, which have not yet been materialized to a specific register (or in this case, a stack location). After the stack passes in this layer, no abstract stores should remain before TEAL lowering (see validation procedures). Resolving these is one half of the main objective of this layer.

## AbstractLoad
Load operations for a `local id` identified variable, which have not yet been materialized. After the stack passes in this layer, no abstract loads should remain before TEAL lowering (see validation procedures). Resolving these is the other half of the main objective of this layer.

