flowchart TD
    A[开始: abiAlignmentInner] --> B{检查ty.toIntern()}
    B -->|empty_tuple_type| C[返回scalar=1]
    B -->|其他类型| D[访问ip.indexToKey(ty.toIntern())]
    
    D -->|int_type| E{int_type.bits是否为0?}
    E -->|是| C
    E -->|否| F[返回intAbiAlignment(bits, target)]
    
    D -->|ptr_type/anyframe_type| G[返回ptrAbiAlignment(target)]
    
    D -->|array_type| H[递归调用子类型的abiAlignmentInner]
    
    D -->|vector_type| I{检查Zig后端}
    I -->|stage2_c| J[递归调用子类型的abiAlignmentInner]
    I -->|stage2_x86_64| K{处理特定向量逻辑}
    K -->|elem_bytes=0| C
    K -->|根据bytes计算对齐| L[返回对应对齐值]
    I -->|其他后端| M[计算bytes和对齐值]
    
    D -->|opt_type| N[调用abiAlignmentInnerOptional]
    
    D -->|error_union_type| O[调用abiAlignmentInnerErrorUnion]
    
    D -->|error_set_type/inferred_error_set| P[根据errorSetBits计算对齐]
    
    D -->|func_type| Q[返回target最小函数对齐]
    
    D -->|simple_type| R{匹配具体simple类型}
    R -->|bool/anyopaque等| C
    R -->|usize/isize| S[返回指针位宽对齐]
    R -->|c_xxx类型| T[调用cTypeAlign]
    R -->|浮点类型| U[根据目标平台返回对齐]
    R -->|void/comptime_int等| C
    
    D -->|struct_type| V{检查struct布局}
    V -->|packed| W[处理packed结构体逻辑]
    V -->|非packed| X{检查对齐是否已解析}
    X -->|未解析| Y[根据strat处理]
    X -->|已解析| Z[返回结构体对齐]
    
    D -->|tuple_type| AA[遍历字段计算最大对齐]
    
    D -->|union_type| AB{检查对齐是否已解析}
    AB -->|未解析| AC[根据strat处理]
    AB -->|已解析| AD[返回联合体对齐]
    
    D -->|enum_type| AE[返回枚举标签类型的对齐]
    
    D -->|其他无效类型| AF[unreachable]
    
    C & F & G & H & J & L & M & N & O & P & Q & S & T & U & W & Y & Z & AA & AC & AD & AE --> AG[返回结果]
