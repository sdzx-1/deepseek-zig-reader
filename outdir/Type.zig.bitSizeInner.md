graph TD
    A[开始] --> B[获取target和ip]
    B --> C[转换strat为strat_lazy]
    C --> D{switch ip.indexToKey(ty)}
    D -->|int_type| E[返回int_type.bits]
    D -->|ptr_type| F{flags.size}
    F -->|slice| G[返回ptrBitWidth*2]
    F -->|其他| H[返回ptrBitWidth]
    D -->|anyframe_type| I[返回ptrBitWidth]
    D -->|array_type| J[计算len和elem_size]
    J --> K{elem_size == 0?}
    K -->|是| L[返回0]
    K -->|否| M[计算elem_bit_size]
    M --> N[返回(len-1)*8*elem_size + elem_bit_size]
    D -->|vector_type| O[计算child_ty和elem_bit_size]
    O --> P[返回elem_bit_size * len]
    D -->|opt_type| Q[返回abiSize *8]
    D -->|error_set_type等| R[返回errorSetBits]
    D -->|error_union_type| Q
    D -->|simple_type| S{匹配具体类型}
    S -->|f16/f32等| T[返回对应位数]
    S -->|usize/isize| U[返回ptrBitWidth]
    S -->|c_char等| V[返回对应c类型位数]
    S -->|bool| W[返回1]
    S -->|void| X[返回0]
    D -->|struct_type| Y{是否为packed?}
    Y -->|是| Z[使用backingIntType递归计算]
    Y -->|否| AA[返回abiSize*8]
    D -->|tuple_type| AB[返回abiSize*8]
    D -->|union_type| AC{是否为packed?}
    AC -->|是| AD[遍历字段计算最大bit_size]
    AC -->|否| AE[返回abiSize*8]
    D -->|enum_type| AF[递归计算tag_ty的bitSize]
    D -->|其他类型| AG[unreachable]
    E --> ZA[结束]
    G --> ZA
    H --> ZA
    I --> ZA
    L --> ZA
    N --> ZA
    P --> ZA
    Q --> ZA
    R --> ZA
    T --> ZA
    U --> ZA
    V --> ZA
    W --> ZA
    X --> ZA
    Z --> ZA
    AA --> ZA
    AB --> ZA
    AD --> ZA
    AE --> ZA
    AF --> ZA
    AG --> ZA
