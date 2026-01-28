# MIR outputs

There are a series of intermediate outputs written in human readable format throughout this stage.
The first output is tagged `"build"`, and is output right after lowering has happened, but before any of the stack regions have been allocated.\

<!-- TODO: link every location where the output is made. -->

500. => stack operations get a stack description.
Stack description for l-stack operations is just as is, with comma separated values, where the last values are the top of the stack and the first values are the bottom of the stack.

Stack description for P, F, X come after (...)
<!-- TODO: complete -->