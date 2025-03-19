graph TD
    A[开始: genInst] --> B{检查air_tags}
    B -->|add| C[调用airBinOp(add)]
    B -->|add_sat| D[调用airSatBinOp(add)]
    B -->|sub| E[调用airBinOp(sub)]
    B -->|mul| F[调用airBinOp(mul)]
    B -->|div_float| G[调用airDiv]
    B -->|cmp_eq| H[调用airCmp(eq)]
    B -->|sqrt| I[调用airUnaryFloatOp(sqrt)]
    B -->|load| J[调用airLoad]
    B -->|store| K[调用airStore]
    B -->|call| L[调用airCall]
    B -->|is_null| M[调用airIsNull]
    B -->|atomic_load| N[调用airAtomicLoad]
    B -->|.inferred_alloc| O[触发unreachable]
    B -->|未实现标签| P[返回cg.fail]
    B -->|其他已知分支| Q[调用对应处理方法]
    C --> R[结束]
    D --> R
    E --> R
    F --> R
    G --> R
    H --> R
    I --> R
    J --> R
    K --> R
    L --> R
    M --> R
    N --> R
    O --> R
    P --> R
    Q --> R

    style A fill:#4CAF50,stroke:#388E3C
    style B fill:#FFC107,stroke:#FFA000
    style O fill:#F44336,stroke:#D32F2F
    style P fill:#F44336,stroke:#D32F2F
    style R fill:#9E9E9E,stroke:#616161
