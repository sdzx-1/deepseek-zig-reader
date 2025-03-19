flowchart TD
    A[开始] --> B[初始化变量: pt, zcu, ip, air_tags]
    B --> C[遍历body中的每个指令inst]
    C --> D{inst未被使用且无需处理?}
    D -- 是 --> C
    D -- 否 --> E[记录调试信息]
    E --> F[处理旧的书签记录, 确保容量]
    F --> G[根据指令标签执行对应操作]
    G --> H{{switch(tag)}}
    
    %% 主要处理分支
    H -->|算术/位运算| I[调用airBinOp等函数]
    H -->|指针运算| J[调用airPtrArithmetic]
    H -->|未实现操作| K[返回TODO错误]
    H -->|数学函数| L[调用airUnaryMath]
    H -->|溢出操作| M[调用airXXXWithOverflow]
    H -->|比较操作| N[调用airCmp]
    H -->|内存操作| O[调用airLoad/Store等]
    H -->|控制流| P[处理分支/循环/返回]
    H -->|其他特例| Q[调用对应处理函数]
    
    %% 公共后续流程
    I --> R
    J --> R
    K --> R[断言检查寄存器状态]
    L --> R
    M --> R
    N --> R
    O --> R
    P --> R
    Q --> R
    
    R --> S{调试模式检查?}
    S -- 是 --> T[执行寄存器跟踪一致性检查]
    S -- 否 --> U
    T --> U[继续下一个指令]
    U --> C
    
    C -->|遍历完成| V[输出跟踪日志]
    V --> W[函数结束]
    
    %% 错误处理分支
    K -.-> X[触发panic/返回错误]
    X --> W
