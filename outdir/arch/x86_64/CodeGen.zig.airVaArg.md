flowchart TD
    A[开始airVaArg] --> B[获取上下文信息: ty, promote_ty, unused等]
    B --> C{调用约定?}
    C -->|x86_64_sysv| D[SysV处理]
    C -->|x86_64_win| E[TODO Win64实现]
    C -->|其他| F[报错不支持varargs]
    
    D --> G[分配临时寄存器]
    G --> H{参数类型分类}
    
    H -->|integer| I[处理整数参数]
    I --> J[设置offset_reg]
    J --> K[比较寄存器值与阈值]
    K --> L[生成条件跳转]
    L --> M[寄存器区处理]
    M --> N[更新offset_reg]
    N --> O[跳转到done]
    L --> P[内存区处理]
    P --> Q[更新overflow_arg_area]
    Q --> O
    O --> R[复制结果到promote_mcv]
    
    H -->|sse| S[处理SSE参数]
    S --> T[设置offset_reg]
    T --> U[比较寄存器值与阈值]
    U --> V[生成条件跳转]
    V --> W[寄存器区处理]
    W --> X[更新offset_reg]
    X --> Y[跳转到done]
    V --> Z[内存区处理]
    Z --> AA[更新overflow_arg_area]
    AA --> Y
    Y --> AB[复制结果到promote_mcv]
    
    H -->|memory| AC[触发unreachable]
    
    R --> AD{unused?}
    AB --> AD
    AD -->|是| AE[返回.unreach]
    AD -->|否| AF[类型是否一致?]
    AF -->|是| AG[直接返回promote_mcv]
    AF -->|否| AH[处理类型转换]
    AH --> AI[生成类型转换指令]
    AI --> AG
    
    AG --> AJ[结束指令生成]
    AE --> AJ
    E --> AJ
    F --> AJ
    
    AJ[finishAir返回结果]
