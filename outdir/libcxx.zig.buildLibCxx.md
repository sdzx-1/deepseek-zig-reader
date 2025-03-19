flowchart TD
    A[开始] --> B{检查LLVM扩展是否启用?}
    B -->|否| C[返回错误: ZigCompilerNotBuiltWithLLVMExtensions]
    B -->|是| D[初始化跟踪和内存分配器]
    D --> E[设置基本参数\n(根名称、输出模式、目标平台等)]
    E --> F[处理路径和ABI参数\n(包含路径、版本参数等)]
    F --> G[配置编译参数\n(优化模式、strip选项等)]
    G --> H[创建根模块]
    H --> I{是否有非单线程配置?}
    I -->|是| J[选择libcxx_base_files + libcxx_thread_files]
    I -->|否| K[选择libcxx_base_files]
    J --> L[遍历源文件并处理]
    K --> L
    L --> M{当前文件是否需要排除?}
    M -->|是| L
    M -->|否| N[添加平台相关编译标志]
    N --> O[添加通用编译标志\n(-DNDEBUG, -std=c++23等)]
    O --> P[添加包含路径到缓存豁免标志]
    P --> Q[将处理后的文件加入源文件列表]
    Q --> R{是否遍历完所有文件?}
    R -->|否| L
    R -->|是| S[创建子编译实例]
    S --> T{子编译是否成功?}
    T -->|否| U[设置错误并返回SubCompilationFailed]
    T -->|是| V[生成静态库文件]
    V --> W[将静态库加入链接队列]
    W --> X[结束]
    
    style A fill:#f9f,stroke:#333
    style B fill:#f96,stroke:#333
    style C fill:#f99,stroke:#333
    style U fill:#f99,stroke:#333
    style X fill:#f9f,stroke:#333
