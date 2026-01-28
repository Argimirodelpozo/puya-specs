<!-- DRAFT -->

# Optimizations performed

<!-- TODO: DIAGRAMA_O0 -->

<!-- TODO: DIAGRAMA_O1 -->

<!-- TODO: DIAGRAMA_O2 -->

The main optimization loop (in [main.py](../puya/src/puya/teal/optimize/main.py)) executes
for each subroutine (inlcuding `main`), already lowered into TEAL after [MIR => TEAL lowering](MIR.md),
the following set of optimizations is performed in the order in which they are presented, dependant on optimization level.

There are two overarching subcategories of `TEAL` optimizations performed: a set at [instruction level](TODO_LINK), and a set at [`Block` level](TODO_LINK).\
The first kind are intra-block optimizations where the constituting ops. are eliminated or replaced according to several passes of analysis.\
The second kind are optimizations performed in blocks as a whole.