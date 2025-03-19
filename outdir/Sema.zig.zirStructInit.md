graph TD
    Start[开始] --> A[初始化变量和获取数据]
    A --> B[解析类型]
    B --> C{类型是否已知?}
    C -- 未知 --> D[调用structInitAnon处理匿名结构体]
    C -- 已知 --> E[获取解析后的类型resolved_ty]
    E --> F{类型是结构体还是联合体?}
    F -- 结构体 --> G[分配found_fields, field_inits, field_assign_idxs]
    G --> H[初始化循环变量field_i=0, extra_index=extra.end]
    H --> I{field_i < fields_len?}
    I -- 是 --> J[获取当前字段的item数据]
    J --> K[解析字段名称和索引]
    K --> L[检查字段是否重复/未初始化]
    L --> M[解析并强制转换初始化值]
    M --> N{是packed结构体?}
    N -- 否 --> O[验证默认值与初始化值是否匹配]
    O --> P[更新field_inits和found_fields]
    P --> Q[增加field_i并更新extra_index]
    Q --> I
    I -- 否 --> R[调用finishStructInit完成结构体初始化]
    F -- 联合体 --> S[验证字段数量是否为1]
    S -- 否 --> T[返回字段数量错误]
    S -- 是 --> U[解析字段名称和索引]
    U --> V{字段类型是否为noreturn?}
    V -- 是 --> W[返回noreturn字段错误]
    V -- 否 --> X[解析初始化值并验证]
    X --> Y{是否编译时常量?}
    Y -- 是 --> Z[生成常量联合体值]
    Y -- 否 --> AA[验证运行时值]
    AA --> AB{需要引用?}
    AB -- 是 --> AC[生成指针分配和标签设置]
    AB -- 否 --> AD[生成联合体初始化指令]
    Z --> AE[返回常量值]
    AC --> AE
    AD --> AE
    R --> End[返回结果]
    D --> End
    T --> End
    W --> End
    End[结束]
