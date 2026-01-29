<!-- DRAFT -->

# Output reading guide
The following is a reference to aid understanding of intermediate compiler outputs at this layer.

> [!NOTE] these are produced as output of the frontends (see the [puya-ts frontend](./../../frontends/puya-ts/puya-ts.md) and the [puyapy frontend](TODO_LINK)) but included here for ease of reading

There are two forms of output representation of the AWST:
- a human readable format, useful for quick debugging and understanding how nodes are set up, nested and interacting with each other; and
- a serialized json format, which is used as input for the Puya compiler by the [puya-ts frontend](../../frontends/puya-ts/puya-ts.md), and contains all explicit information present in all fields in [node classes](./node-reference/node-reference.md).

We provide some general reading guidelines, and then a full reference of textual representation of each node.

>[!NOTE] the textual representation is based on the `ToCodeVisitor` found [here](TODO_LINK) in the reference implementation

## Human readable AWST output
Generally speaking, contracts start with a field `method_resolution_order` (from now on, _mro_).\
The _mro_ is a list of parent classes, denoting the lookup order of methods (see the python [mro](TODO_LINK) for more details on the C3 algorithm and how this works).
<!-- TODO: improve expl. -->
An example in a contract inheriting from an `ARC4Contract`:
```
  method_resolution_order: (
    algopy.arc4.ARC4Contract,
    algopy._contract.Contract,
  )
```

Then, we'll have an object called `globals` which will have a list of [key]:type pairs, where keys are valid AVM global state keys and types are valid AVM types 

> [!NOTE] AVM types are i.e. one of `uint64` or `bytes`, and type aliases that boil down to one of these. E.g. an `asset` type is an alias of `uint64`, or an `account` type is an alias of `address` which is in turn an alias of `bytes`.



## AWST output in Json format
<!-- TODO: complete with commentary on json output. Maybe: scheme? -->