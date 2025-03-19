graph TD
    Start[开始] --> A{ty.toIntern()}
    A -->|empty_tuple_type| B[返回 false]
    A -->|其他类型| C[indexToKey 分支]
    
    C -->|int_type| D[int_type.bits !=0?]
    D -->|是| E[返回 true]
    D -->|否| F[返回 false]
    
    C -->|ptr_type| G{ignore_comptime_only?}
    G -->|是| H[返回 true]
    G -->|否| I[strat 分支]
    I -->|sema| J[调用 comptimeOnlySema]
    J -->|结果| K[返回 !结果]
    I -->|eager| L[返回 !comptimeOnly]
    I -->|lazy| M[抛出 NeedLazy]
    
    C -->|array_type/vector_type| N[长度>0?]
    N -->|否| O[返回 false]
    N -->|是| P[递归检查子类型]
    P --> Q[返回子类型结果]
    
    C -->|opt_type| R{child_ty 是 noreturn?}
    R -->|是| S[返回 false]
    R -->|否| T{ignore_comptime_only?}
    T -->|是| U[返回 true]
    T -->|否| V[strat 分支]
    V -->|sema/eager| W[检查 comptimeOnly]
    V -->|lazy| X[抛出 NeedLazy]
    
    C -->|error_union/set| Y[返回 true]
    C -->|func_type| Z[返回 false]
    
    C -->|simple_type| AA[根据具体类型判断]
    AA -->|f16/bool等| AB[返回 true]
    AA -->|void/comptime_int等| AC[返回 false]
    
    C -->|struct_type| AD{是否假设有运行时位}
    AD -->|是| AE[返回 true]
    AD -->|否| AF[遍历所有字段]
    AF --> AG[递归检查字段类型]
    AG -->|任一字段为true| AH[返回 true]
    AG -->|全部字段为false| AI[返回 false]
    
    C -->|tuple_type| AJ[遍历元组元素]
    AJ --> AK[存在运行时字段?]
    AK -->|是| AL[返回 true]
    AK -->|否| AM[返回 false]
    
    C -->|union_type| AN[检查运行时标签]
    AN --> AO[递归检查标签类型]
    AO -->|标签有运行时位| AP[返回 true]
    AN --> AQ[遍历联合体字段]
    AQ --> AR[任一字段为true?]
    AR -->|是| AS[返回 true]
    AR -->|否| AT[返回 false]
    
    C -->|enum_type| AU[递归检查标签类型]
    AU --> AV[返回子结果]
    
    C -->|其他类型| AW[触发unreachable]
    style Start fill:#90EE90
    style B fill:#FFB6C1
    style E fill:#FFB6C1
    style F fill:#FFB6C1
    style H fill:#FFB6C1
    style K fill:#FFB6C1
    style L fill:#FFB6C1
    style M fill:#FFD700
    style O fill:#FFB6C1
    style Q fill:#FFB6C1
    style S fill:#FFB6C1
    style U fill:#FFB6C1
    style W fill:#FFB6C1
    style X fill:#FFD700
    style Y fill:#FFB6C1
    style Z fill:#FFB6C1
    style AB fill:#FFB6C1
    style AC fill:#FFB6C1
    style AE fill:#FFB6C1
    style AH fill:#FFB6C1
    style AI fill:#FFB6C1
    style AL fill:#FFB6C1
    style AM fill:#FFB6C1
    style AP fill:#FFB6C1
    style AS fill:#FFB6C1
    style AT fill:#FFB6C1
    style AV fill:#FFB6C1
    style AW fill:#FF4500
