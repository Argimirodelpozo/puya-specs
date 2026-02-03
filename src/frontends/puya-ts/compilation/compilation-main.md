# Main compilation pipeline: from AST to AWST

<!-- TODO: intro text -->
<!-- TODO: quick high level overview of each step -->

The following is a schematic view of the main compilation procedure.

```mermaid
flowchart
flowchart 
  A[Setup Logging Context] --> B[Register PTypes]
  B --> C[Create TypeScript Program]
  C --> D[Build AWST from TS Program]
  D --> E[Validate AWST]
  E --> F{Dry Run?}
  F -- No --> G[Puya Compile AWST]
  F -- Yes --> H[Skip Backend Compilation]
  G --> I[Return Results]
  H --> I

  I --> J[[CompileResult<br/>AST + AWST + Compilation Set]]
```

> [Link to implementation](TODO_LINK)