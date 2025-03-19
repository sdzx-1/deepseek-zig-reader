graph TD
    A[开始genBinOp] --> B{是否浮点且需要libcall?}
    B -- 是 --> C[生成libcall函数名]
    C --> D[调用genCall生成结果]
    D --> E{操作是mod或div_trunc/floor?}
    E -- 是 --> F[处理调整结果]
    E -- 否 --> G[返回结果]
    F --> H[生成加法调用或寄存器调整]
    H --> G

    B -- 否 --> I{是否是SSE操作?}
    I -- 是 --> J[处理向量/浮点操作]
    J --> K{确定SSE指令类型}
    K --> L[处理16位浮点转换]
    K --> M[处理32/64位浮点]
    K --> N[处理整型向量操作]
    L --> O[生成ph2ps/ps2ph指令]
    M --> P[生成ss/sd指令]
    N --> Q[生成padd/pmul等指令]

    I -- 否 --> R[处理整型/指针操作]
    R --> S{操作类型判断}
    S -- 算术运算 --> T[生成add/sub/mul指令]
    S -- 位运算 --> U[生成and/or/xor指令]
    S -- min/max --> V[生成比较和cmov指令]
    S -- 指针运算 --> W[处理元素大小调整]

    T --> X[处理溢出和回写]
    U --> X
    V --> Y[生成条件移动]
    W --> Z[生成乘法和加减]

    J --> AA{是否需要混合指令?}
    AA -- 是 --> AB[生成blendv指令]
    AA -- 否 --> AC[直接写结果]

    R --> AD{是否需要特殊处理?}
    AD -- 大整型 --> AE[使用多寄存器处理]
    AD -- 比较操作 --> AF[生成条件码和掩码]

    AE --> AG[处理高低位比较]
    AF --> AH[生成movmsk转换]

    AG --> AI[回写结果寄存器]
    AH --> AI

    AI --> AJ[返回MCValue]
    AC --> AJ
    G --> AJ
