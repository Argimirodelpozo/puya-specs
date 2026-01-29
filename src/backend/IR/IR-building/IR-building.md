## Lowering from AWST

TODO: IR main until we get to the building main
(logicsig vs. contract build, embedded subroutines, etc.)


The following is a schematic full IR diagram, that shows the process from AWST to the final IR stage.


<!-- TODO: turn into diagram! -->
AWST => build IR (unbound) subroutines (embedded, user) => 
build contract / logic_sig IR => 
transform IR:
Pipeline goes optimize program => lower aggregate IR => optimize program => slot elimination => destructuring => post-destructure validations (validate module artifact)

<!-- TODO: completar con todo lo de IR 2 TEAL -->


## Function Builder and the build body "entrypoint"


## Braun et. al. SSA construction

<!-- TODO: link to reference impl. BraunSSA -->
<!-- TODO: cite with footnotes -->
In order to build a control flow graph and SSA form directly from the AWST, we employ the Braun SSA construction algorithm.\
It differs from the classical SSA form construction in that it does not need pre-processing, `Phi` nodes are inserted lazily as needed, and by the end of the build pass we have both a control flow graph (with `BasicBlock`s as nodes) and a program in SSA form, both ready to be optimized before the [destructuring pass](#ssa-destructuring).

To achieve this, we need to keep track of:
- the set $sB$ of sealed blocks (basic blocks whose instructions have been visited, that have already been constructed and finished with a terminator).
- for each variable found, a set to its `Register`s (see below) mapped to the `BasicBlock` in which the variable version was defined. These are all grouped under the variable _Identifier_, which may be modeled as a simple unique string.
<!-- TODO: explain -->
- the set of "incomplete" `Phi` nodes per `BasicBlock`, $phi_B$
- a set of variable versions, mappping variable identifiers to simple unsigned integers representing versioning, $v_n$ (where $n$ is an unsigned integer, these will be assigned sequentially).

We make extensive use here of the `Register` construct. A register is an abstraction that models local variable storage, and will be resolved further down the pipeline to a [stack location](TODO_LINK_Algospecs) or [scratch space](TODO_LINK_Algospecs) (see the [MIR](MIR.md) section for more details on materialising abstract register allocation).\
`Register` nodes have an associated `name` and `version` number, and are usually represented in output as `{name}#{version}` (using the `#` symbol as a separator).




### Trivial `Phi`
A `Phi` node is said to be trivial if it references _at most_ one value other than self.

### Builder full reference

###


#### `BytesBinaryOperation`
add => concat

bit_and => 

## Array lowering




## SSA construction: Braun Algorithm


## Implementation details

The following diagram shows the transformation pipeline schematically.

```mermaid
flowchart TD
    A[Start: get_transform_pipeline]

    B[_optimize_program_ir<br/>qualifier="ssa.opt"<br/>artifact_ir]
    C[_lower_aggregate_ir<br/>ref]
    D[_optimize_program_ir<br/>qualifier="ssa.array.opt"<br/>artifact_ir]
    E[slot_elimination<br/>ref]
    F[destructure_ssa]

    A --> B --> C --> D --> E --> F
```


After building the IR, the _transform pipeline_ is as follows:
```python
def get_transform_pipeline(
    artifact_ir: ModuleArtifact,
) -> list[Callable[[ArtifactCompileContext, Program], None]]:
    ref = artifact_ir.metadata.ref
    return [
        functools.partial(_optimize_program_ir, artifact_ir=artifact_ir, qualifier="ssa.opt"),
        functools.partial(_lower_aggregate_ir, ref=ref),
        functools.partial(
            _optimize_program_ir, artifact_ir=artifact_ir, qualifier="ssa.array.opt"
        ),
        functools.partial(slot_elimination, ref=ref),
        destructure_ssa,
    ]
```
In other words, the pipeline does IR level optimization, aggregate lowering, 
another full optimization pass (same opts.), then specifically slot elimination,
and finally destructuring.


## Aggregate node lowering
Consider now an IR [`Program`](#full-models-reference) $P$. An aggregate replacement visitor is instantiated for each subroutine in $P$ (i.e. `main` and member subroutines).\
The IR nodes visited by this stage are:
- `BytesEncode` 
- `DecodeBytes`
- `ArrayLength`
- `ExtractValue`
- `ReplaceValue`
- `BoxRead`
- `BoxWrite`

We now delve into the building process for each of these aggregate related primitives.
### `BytesEncode` aggregate lowering
### `DecodeBytes` aggregate lowering
### `ArrayLength` aggregate lowering
### `ExtractValue` aggregate lowering
### `ReplaceValue` aggregate lowering
### `BoxWrite` aggregate lowering
### `BoxRead` aggregate lowering

At the end of this build subprocess for each `Subroutine`, an in-SSA validation is performed. It checks that a fundamental property of SSA form is preserved; that is, all registers are assigned _once_ in the context of the `Subroutine` under analysis.\
Afterwards, a pass is performed to call individual validators for each attribute in the `Subroutine`.
> [Link to reference implementation](LINK_validate_with_ssa in Subroutine(Context), models.py)



<!-- TODO: commentary on encodings.py -->