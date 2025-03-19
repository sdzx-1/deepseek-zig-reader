flowchart TD
    A[开始 airAggregateInit] --> B[获取 pt, zcu, result_ty, len, elements]
    B --> C{判断 result_ty 类型}
    
    C --> |struct| D[分配 frame_index]
    D --> E{是否 packed 布局?}
    E --> |是| F[调用 genInlineMemset 清零内存]
    F --> G[遍历 elements]
    G --> H[检查 comptime 字段]
    H --> |非 comptime| I[处理位偏移计算]
    I --> J{字段大小 >64?}
    J --> |是| K[返回 TODO 错误]
    J --> |否| L[寄存器分配与位操作]
    L --> M[生成 OR 指令合并字段]
    G --> N[继续下一个元素]
    N --> G
    
    E --> |否| O[遍历 elements]
    O --> P[检查 comptime 字段]
    P --> |非 comptime| Q[按字段偏移写入内存]
    Q --> R[继续下一个元素]
    R --> O
    
    C --> |array/vector| S{是否是布尔向量?}
    S --> |是| T[分配寄存器并初始化清零]
    T --> U[遍历 elements]
    U --> V[生成位掩码和移位操作]
    V --> W[合并到寄存器]
    W --> X[继续下一个元素]
    X --> U
    
    S --> |否| Y[分配 frame_index]
    Y --> Z[遍历 elements 写入内存]
    Z --> AA[处理哨兵值]
    
    C --> |其他类型| AB[unreachable]
    
    D & Y & T --> AC[构建 result MCValue]
    AC --> AD{元素数量 ≤ Liveness.bpi-1?}
    AD --> |是| AE[复制到小缓冲区]
    AD --> |否| AF[使用 BigTomb 处理]
    AE & AF --> AG[finishAir 返回结果]
    AG --> HALT[结束]
    
    K --> AG
