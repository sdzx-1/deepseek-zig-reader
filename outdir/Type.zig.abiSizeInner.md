graph TD
    A[Start: abiSizeInner] --> B{ty.toIntern()?}
    B --> |empty_tuple_type| C[Return scalar=0]
    B --> |Other Types| D[ip.indexToKey(ty.toIntern())]
    
    D --> |int_type| E[Check bits]
    E --> |bits=0| C
    E --> |bits>0| F[Return intAbiSize(bits, target)]
    
    D --> |ptr_type| G{flags.size?}
    G --> |slice| H[Return ptr_size*2]
    G --> |other| I[Return ptr_size]
    
    D --> |array_type| J[Get len and child type]
    J --> K{len=0?}
    K --> |Yes| C
    K --> |No| L[Recursively get elem_size]
    L --> M{Result type?}
    M --> |scalar| N[Return len*elem_size]
    M --> |val| O[Check strat]
    O --> |lazy| P[Return .val with lazy_size]
    
    D --> |vector_type| Q[Determine sub_strat]
    Q --> R[Calculate alignment]
    R --> S{Backend?}
    S --> |Default| T[Calculate total_bits/8]
    S --> |stage2_c| U[elem_bytes * len]
    S --> |stage2_x86_64| V[Special handling for bool]
    T --> W[Return aligned total_bytes]
    U --> W
    V --> W
    
    D --> |error_union_type| X[Calculate code_size]
    X --> Y{Payload has bits?}
    Y --> |No| Z[Return code_size]
    Y --> |Yes| AA[Calculate alignment and size]
    AA --> AB[Combine payload and code_size]
    AB --> AC[Return final size]
    
    D --> |struct_type| AD{strat?}
    AD --> |sema| AE[Resolve layout]
    AD --> |lazy| AF[Check layout completeness]
    AF --> AG[Return .val if incomplete]
    AE --> AH[Return packed/extern size]
    
    D --> |union_type| AI[Check layout status]
    AI --> AJ[Return resolved size]
    
    D --> |simple_type| AK[Switch on simple type]
    AK --> AL[Return predefined sizes]
    
    D --> |enum_type| AM[Get tag_ty size]
    
    D --> |other types| AN[Unreachable cases]
    
    style A fill:#f9f,stroke:#333,stroke-width:4px
    style C fill:#bbf,stroke:#333
    style F fill:#bbf,stroke:#333
    style H fill:#bbf,stroke:#333
    style I fill:#bbf,stroke:#333
    style N fill:#bbf,stroke:#333
    style P fill:#bbf,stroke:#333
    style W fill:#bbf,stroke:#333
    style Z fill:#bbf,stroke:#333
    style AC fill:#bbf,stroke:#333
    style AH fill:#bbf,stroke:#333
    style AG fill:#bbf,stroke:#333
    style AJ fill:#bbf,stroke:#333
    style AL fill:#bbf,stroke:#333
    style AM fill:#bbf,stroke:#333
