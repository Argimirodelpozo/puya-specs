<!-- DRAFT -->

# Intermediate Representation (IR) layer

The IR layer does a lot of the heavier lifting in the Puya compiler. It transforms the AWST into an SSA form and builds a control flow graph at the same time (where nodes in the graph are `BasicBlock`s). It then performs an extensive set of varyingly complex optimizations, before going through [destructuring](#ssa-destructuring) ("coming out" of SSA form, where `PhiNode`s are resolved), and finally performing another set of pos-destructure optimizations.