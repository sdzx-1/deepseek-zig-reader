graph TD
    A[开始 writeRelocs] --> B[初始化 tracy 跟踪]
    B --> C[记录调试信息: 地址和名称]
    C --> D[获取 cpu_arch 和 relocs]
    D --> E[初始化 i = 0]
    E --> F{遍历 relocs 中的每个 rel?}
    F --> |是| G[计算 rel_offset 和 r_address]
    G --> H[确定 r_symbolnum 和 r_extern]
    H --> I[计算 addend]
    I --> J[记录详细调试信息]
    J --> K{cpu_arch 类型?}
    
    K --> |aarch64| L[aarch64 分支]
    L --> M{rel.type 类型?}
    M --> |unsigned| N[处理长度和 addend]
    M --> |其他类型| O[设置 r_type]
    N --> P[写入 buffer[i]]
    O --> P
    P --> Q[i += 1]
    
    K --> |x86_64| R[x86_64 分支]
    R --> S[处理 PC 相对地址调整]
    S --> T{rel.type 类型?}
    T --> |signed 系列| U[设置 r_type]
    T --> |其他类型| U
    U --> V[写入 buffer[i]]
    V --> Q
    
    Q --> F
    F --> |否| W[断言 i == buffer.len]
    W --> X[结束 tracy 跟踪]
    X --> Y[函数结束]
