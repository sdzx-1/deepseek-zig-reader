flowchart TD
    A[开始] --> B[初始化llvm_triple动态数组]
    B --> C{根据target.cpu.arch选择llvm_arch}
    C --> |arm/armeb/thumb等| D[处理ARM子架构]
    C --> |mips/mipsel等| E[检查MIPS特性决定子架构]
    C --> |其他架构| F[直接映射架构名]
    D --> G[调用subArchName获取子架构]
    E --> F
    F --> H{是否存在子架构?}
    H --> |是| I[追加子架构到llvm_triple]
    H --> |否| J[跳过]
    I --> K[追加'-'分隔符]
    J --> K
    K --> L{根据target.os.tag选择供应商}
    L --> |apple/scei等| M[追加供应商名]
    L --> |其他| N[追加'unknown']
    M --> O[追加'-'分隔符]
    N --> O
    O --> P{根据target.os.tag选择llvm_os}
    P --> |linux/windows等| Q[追加操作系统名]
    P --> |其他| R[追加'unknown']
    Q --> S[处理操作系统版本]
    R --> S
    S --> T{是否需要附加版本号?}
    T --> |是| U[格式化并追加版本号]
    T --> |否| V[跳过]
    U --> W[追加'-'分隔符]
    V --> W
    W --> X{根据target.abi选择llvm_abi}
    X --> |gnu/musl等| Y[追加ABI名]
    X --> |其他| Z[追加'unknown']
    Y --> AA[处理ABI相关版本信息]
    Z --> AA
    AA --> AB[返回llvm_triple字符串]
    AB --> AC[结束]

    style A fill:#f9f,stroke:#333
    style AC fill:#f9f,stroke:#333
    style C,F,G,E,D,Q,P,X,Y,Z,L,M,N,K,O,S,T,U,V,W,AA,AB stroke:#333,stroke-width:2px
