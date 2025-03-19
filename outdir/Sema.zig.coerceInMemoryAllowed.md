flowchart TD
    A[开始] --> B{dest_ty == src_ty?}
    B -- 是 --> C[返回 .ok]
    B -- 否 --> D[获取dest_tag和src_tag]
    D --> E{类型判断分支}

    E -->|整数类型| F{都是int类型?}
    F -- 是 --> G{符号和位数相同?}
    G -- 是 --> C
    G -- 否 --> H[返回int_not_coercible错误]
    
    E -->|浮点类型| I{都是float类型?}
    I -- 是 --> J{位数相同?}
    J -- 是 --> C
    J -- 否 --> K[继续后续判断]
    
    E -->|指针/可选指针| L{存在指针类型?}
    L -- 是 --> M[调用coerceInMemoryAllowedPtrs]
    M --> C或错误
    
    E -->|切片类型| N{都是切片类型?}
    N -- 是 --> O[调用coerceInMemoryAllowedPtrs]
    O --> C或错误
    
    E -->|函数类型| P{都是函数类型?}
    P -- 是 --> Q[调用coerceInMemoryAllowedFns]
    Q --> C或错误
    
    E -->|错误联合| R{都是error_union?}
    R -- 是 --> S[递归检查payload和error set]
    S --> C或error_union_payload错误
    
    E -->|错误集合| T{都是error_set?}
    T -- 是 --> U[双向检查错误集兼容性]
    U --> C或错误
    
    E -->|数组类型| V{都是数组类型?}
    V -- 是 --> W{长度相同?}
    W -- 否 --> X[返回array_len错误]
    W -- 是 --> Y[递归检查元素类型和sentinel]
    Y --> C或array_elem/array_sentinel错误
    
    E -->|向量类型| Z{都是向量类型?}
    Z -- 是 --> AA{长度相同?}
    AA -- 否 --> AB[返回vector_len错误]
    AA -- 是 --> AC[递归检查元素类型]
    AC --> C或vector_elem错误
    
    E -->|数组↔向量| AD{类型互换?}
    AD -- 是 --> AE{长度和元素类型兼容?}
    AE -- 否 --> AF[返回array_len或array_elem错误]
    AE -- 是 --> AG[检查padding位布局]
    AG -- 兼容 --> C
    AG -- 不兼容 --> 继续后续判断
    
    E -->|可选类型| AH{都是optional类型?}
    AH -- 是 --> AI[检查指针类型一致性]
    AI -- 不一致 --> AJ[返回optional_shape错误]
    AI -- 一致 --> AK[递归检查子类型]
    AK --> C或optional_child错误
    
    E -->|元组类型| AL{都是元组类型?}
    AL -- 是 --> AM{字段数量/对齐/类型匹配?}
    AM -- 否 --> 继续后续判断
    AM -- 是 --> AN[递归检查每个字段类型]
    AN --> C或继续判断
    
    E -->|其他类型| AO[返回.no_match错误]
    K --> AO
    AJ --> AO
    AF --> AO
    AB --> AO
    X --> AO
    H --> AO
