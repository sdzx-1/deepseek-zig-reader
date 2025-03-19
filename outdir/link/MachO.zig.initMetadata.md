flowchart TD
    A[开始] --> B{是否非可重定位?}
    B -->|是| C[初始化段: __TEXT_ZIG]
    C --> D[初始化段: __CONST_ZIG]
    D --> E[初始化段: __DATA_ZIG]
    E --> F[初始化段: __BSS_ZIG]
    F --> G{是否有调试选项?}
    G -->|是| H[创建dSYM并初始化调试信息]
    G -->|否| I[跳过调试初始化]
    H --> I
    B -->|否| I
    I --> J[定义辅助函数: appendSect和allocSect]
    J --> K[添加节: __text_zig]
    K --> L{是否可重定位?}
    L -->|是| M[调用allocSect分配空间]
    L -->|否| N[调用appendSect附加到段]
    M --> O[添加节: __const_zig]
    N --> O
    O --> P{是否可重定位?}
    P -->|是| Q[调用allocSect分配空间]
    P -->|否| R[调用appendSect附加到段]
    Q --> S[添加节: __data_zig]
    R --> S
    S --> T{是否可重定位?}
    T -->|是| U[调用allocSect分配空间]
    T -->|否| V[调用appendSect附加到段]
    U --> W[添加节: __bss_zig]
    V --> W
    W --> X{是否可重定位且需调试?}
    X -->|是| Y[添加所有调试节并初始化]
    X -->|否| Z[结束]
    Y --> Z
