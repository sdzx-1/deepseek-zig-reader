graph TD
    A[开始] --> B{检查 have_llvm}
    B -->|否| C[返回错误]
    B -->|是| D[初始化 ArenaAllocator]
    D --> E{匹配 in_crt_file 类型}
    E -->|crt1_o| F[初始化参数 args]
    F --> G[添加 -DCRT 参数]
    G --> H[设置 crt1.c 文件]
    H --> I[调用 build_crt_file 生成 crt1.o]
    E -->|rcrt1_o| J[初始化参数 args]
    J --> K[添加 -DCRT 参数]
    K --> L[设置 rcrt1.c 文件]
    L --> M[调用 build_crt_file 生成 rcrt1.o]
    E -->|scrt1_o| N[初始化参数 args]
    N --> O[添加 -DCRT 参数]
    O --> P[设置 Scrt1.c 文件]
    P --> Q[调用 build_crt_file 生成 Scrt1.o]
    E -->|libc_a| R[获取目标架构信息]
    R --> S[构建源文件映射表]
    S --> T[处理架构特定文件]
    T --> U[生成 C 源文件列表]
    U --> V[调用 build_crt_file 生成 libc.a]
    E -->|libc_so| W[解析编译配置]
    W --> X[设置架构宏定义]
    X --> Y[创建根模块]
    Y --> Z[创建子编译]
    Z --> AA[配置子编译参数]
    AA --> AB[更新子编译状态]
    AB --> AC[生成 libc.so]
    AC --> AD[添加至 CRT 文件列表]
    C --> AE[结束]
    I --> AE
    M --> AE
    Q --> AE
    V --> AE
    AD --> AE
