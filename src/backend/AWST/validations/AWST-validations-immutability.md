## Attempt to modify immutable object validation

This checker performs basic validation of potentially mutating operations (i.e. two categories: left hand side of assignments and base of array in-place modifications), ensuring that the underlying modified elements are not immutable in the first place.

> [Link to reference implementation](TODO_LINK)

We traverse the AWST starting at `RootNode`s (module level statements), and visiting the whole tree looking for mutability statements and expressions. There is currently 4 kinds of AWST nodes for this purpose:
- `AssignmentExpression`
- `AssignmentStatement`
- `ArrayPop`
- `ArrayExtend`

In the case of assignments, a check for the left hand side value is performed.\ 
For field and index access expressions, the underlying `Wtype` of the base should not be immutable. For `Arc4Struct`s specifically, we check that they are not marked as frozen.\
<!-- TODO: review this a little more closely -->
<!-- why not use the immutable property? -->
Finally for `TupleExpression`s, we make sure to validate each lvalue that conforms it independantly.

In the case of `ArrayPop` and `ArrayExtend` nodes, we validate that the underlying `Wtype` of the base is mutable (its `immutable` flag unset). Otherwise, an error is emitted and compilation fails.