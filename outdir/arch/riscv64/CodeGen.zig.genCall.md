graph TD
    A[开始 genCall] --> B[获取函数类型 fn_ty]
    B -->|info 类型| C{是 air 还是 lib?}
    C -->|air| D[解析 air 的函数类型]
    C -->|lib| E[生成 lib 的函数类型]
    D --> F[解析调用约定 call_info]
    E --> F
    F --> G[分配调用栈帧并调整大小/对齐]
    G --> H[初始化寄存器锁列表 reg_locks]
    H --> I[处理返回值位置]
    I --> J{返回值类型}
    J -->|间接返回| K[分配帧索引并设置寄存器]
    J -->|其他| L[跳过]
    K --> M
    L --> M[遍历所有参数]
    M --> N[处理参数传递方式]
    N --> O{参数类型}
    O -->|寄存器| P[锁定寄存器并记录]
    O -->|寄存器对| Q[锁定双寄存器]
    O -->|间接| R[分配帧索引并设置基址]
    O -->|其他| S[报错]
    P --> T
    Q --> T
    R --> T
    S --> T[继续循环]
    T -->|完成所有参数| U[生成参数传递代码]
    U --> V{调用类型}
    V -->|air| W{是否为已知函数值?}
    W -->|是| X[处理ELF符号生成调用]
    W -->|否| Y[动态地址调用 (jalr)]
    X --> Z[生成 jalr 指令]
    Y --> Z
    V -->|lib| AA[报错 TODO]
    Z --> AB[重置向量设置 avl/vtype]
    AA --> AB
    AB --> AC[返回 call_info.return_value.short]
