graph TD
    A[Start doTask] --> B[Task Type?]
    B -->|load_explicitly_provided| C{comp.bin_file exists?}
    C -->|Yes| D[Decrement remaining_prelink_tasks]
    D --> E[Create prog_node]
    E --> F[Loop through link_inputs]
    F --> G[base.loadInput(input)]
    G --> H{Error?}
    H -->|Yes| I[Handle Error via diags]
    H -->|No| J[prog_node.completeOne]
    J --> F
    F -->|End loop| K[prog_node.end]

    B -->|load_host_libc| L{comp.bin_file exists?}
    L -->|Yes| M[Decrement remaining_prelink_tasks]
    M --> N[Create prog_node]
    N --> O[Loop through flags]
    O --> P{Link Mode?}
    P -->|dynamic| Q[Try load DSO]
    Q --> R{FileNotFound?}
    R -->|Yes| S[Try load Archive]
    R -->|No| T[Handle Error via diags]
    P -->|static| U[Load Archive]
    U --> V{Error?}
    V -->|Yes| W[Handle Error via diags]
    O -->|End loop| X[prog_node.end]

    B -->|load_object| Y{comp.bin_file exists?}
    Y -->|Yes| Z[Decrement remaining_prelink_tasks]
    Z --> AA[Create prog_node]
    AA --> AB[base.openLoadObject]
    AB --> AC{Error?}
    AC -->|Yes| AD[Handle Error via diags]

    B -->|load_archive| AE{comp.bin_file exists?}
    AE -->|Yes| AF[Decrement remaining_prelink_tasks]
    AF --> AG[Create prog_node]
    AG --> AH[base.openLoadArchive]
    AH --> AI{Error?}
    AI -->|Yes| AJ[Handle Error via diags]

    B -->|load_dso| AK{comp.bin_file exists?}
    AK -->|Yes| AL[Decrement remaining_prelink_tasks]
    AL --> AM[Create prog_node]
    AM --> AN[base.openLoadDso]
    AN --> AO{Error?}
    AO -->|Yes| AP[Handle Error via diags]

    B -->|load_input| AQ{comp.bin_file exists?}
    AQ -->|Yes| AR[Decrement remaining_prelink_tasks]
    AR --> AS[Create prog_node]
    AS --> AT[base.loadInput]
    AT --> AU{Error?}
    AU -->|Yes| AV[Handle Error via diags]

    B -->|codegen_nav| AW[remaining_prelink_tasks == 0?]
    AW -->|Yes| AX[Activate PerThread]
    AX --> AY[linkerUpdateNav]
    AY --> AZ[Handle Error?]
    AZ -->|OutOfMemory| BA[diags.setAllocFailure]
    AW -->|No| BB[Postpone task]

    B -->|codegen_func| BC[remaining_prelink_tasks == 0?]
    BC -->|Yes| BD[Activate PerThread]
    BD --> BE[linkerUpdateFunc]
    BE --> BF[Handle Error?]
    BF -->|OutOfMemory| BG[diags.setAllocFailure]
    BC -->|No| BH[Postpone task]

    B -->|codegen_type| BI[remaining_prelink_tasks == 0?]
    BI -->|Yes| BJ[Activate PerThread]
    BJ --> BK[linkerUpdateContainerType]
    BK --> BL[Handle Error?]
    BL -->|OutOfMemory| BM[diags.setAllocFailure]
    BI -->|No| BN[Postpone task]

    B -->|update_line_number| BO[Activate PerThread]
    BO --> BP[linkerUpdateLineNumber]
    BP --> BQ[Handle Error?]
    BQ -->|OutOfMemory| BR[diags.setAllocFailure]

    style A stroke:#333,stroke-width:2px
    style B stroke:#666,stroke-dasharray: 5 5
