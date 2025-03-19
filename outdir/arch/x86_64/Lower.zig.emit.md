graph TD
    A[开始] --> B[初始化emit_prefix, emit_mnemonic, emit_ops_storage]
    B --> C[遍历操作数 ops]
    C --> D{操作数类型是 .mem?}
    D --> |否| E[直接复制操作数]
    D --> |是| F[检查 base 类型]
    F --> |.reloc| G[处理重定位符号]
    G --> H{bin_file 类型?}
    H --> |ELF| I[处理 ELF 的 TLS 和重定位]
    H --> |Mach-O| J[处理 Mach-O 的 TLS 和重定位]
    H --> |其他| K[报错: 不支持格式]
    
    I --> L{是否是 TLS 符号?}
    L --> |是| M{lower.pic?}
    M --> |是| N[生成 LD 模型指令]
    M --> |否| O[生成 LE 模型指令]
    L --> |否| P{助记符类型?}
    P --> |.lea| Q[生成 RIP 相对地址]
    P --> |.mov| R[处理外部指针或 RIP 重定位]
    P --> |.call| S[生成基址寄存器指令]
    
    J --> T{是否是 TLS 符号?}
    T --> |是| U[生成 Mach-O TLS 指令]
    T --> |否| V{助记符类型?}
    V --> |.lea| W[处理 RIP 或外部指针]
    V --> |.mov| X[处理外部指针或 RIP 重定位]
    
    E --> Y[操作数处理完成]
    N --> Y
    O --> Y
    Q --> Y
    R --> Y
    S --> Y
    U --> Y
    W --> Y
    X --> Y
    K --> Y
    
    Y --> Z[循环结束?]
    Z --> |否| C
    Z --> |是| AA[生成最终指令到结果列表]
    AA --> AB[结束]
