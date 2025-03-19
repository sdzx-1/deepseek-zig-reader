flowchart TD
    A[开始] --> B[初始化变量和解析指令数据]
    B --> C[处理outputs和inputs的解析]
    C --> D{inst_ty有运行时位?}
    D --> |是| E[分配inst_local并安全初始化]
    D --> |否| F[inst_local设为.none]
    E --> G[记录locals起始位置]
    F --> G
    G --> H[遍历outputs]
    H --> I{约束是否合法?}
    I --> |否| J[返回错误]
    I --> |是| K{是寄存器约束?}
    K --> |是| L[生成寄存器变量和约束]
    K --> |否| M[处理直接输出]
    L --> H
    M --> H
    H --> |完成| N[遍历inputs]
    N --> O{约束是否合法?}
    O --> |否| J
    O --> |是| P{需要本地存储?}
    P --> |是| Q[生成输入变量并赋值]
    P --> |否| R[直接使用输入值]
    Q --> N
    R --> N
    N --> |完成| S[处理clobbers]
    S --> T[处理内联汇编源码]
    T --> U[生成__asm__语句]
    U --> V[生成输出约束部分]
    V --> W[生成输入约束部分]
    W --> X[生成clobber部分]
    X --> Y[遍历outputs赋值寄存器]
    Y --> Z[处理未使用指令]
    Z --> AA[返回结果]
    J --> AA
