<!-- DRAFT -->

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

## Storage types validation
<!-- TODO: explain -->

## Immutability validation
<!-- TODO: explain -->

