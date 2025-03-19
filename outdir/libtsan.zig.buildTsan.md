flowchart TD
    A[开始] --> B{检查LLVM支持}
    B -- 不支持 --> C[返回错误]
    B -- 支持 --> D[初始化ArenaAllocator]
    D --> E[确定目标系统和根名称]
    E --> F[生成basename和emit_bin]
    F --> G[配置优化模式、strip等参数]
    G --> H{配置解析成功?}
    H -- 失败 --> I[设置错误并返回]
    H -- 成功 --> J[创建根模块root_mod]
    J --> K{模块创建成功?}
    K -- 失败 --> I
    K -- 成功 --> L[添加tsan_sources到编译列表]
    L --> M[根据OS添加平台特定源文件]
    M --> N[添加汇编源文件]
    N --> O[添加sanitizer_common源文件]
    O --> P[添加libc依赖/无依赖源文件]
    P --> Q[添加符号化工具源文件]
    Q --> R[添加拦截器源文件]
    R --> S[配置链接器参数]
    S --> T[创建子编译sub_compilation]
    T --> U{子编译创建成功?}
    U -- 失败 --> I
    U -- 成功 --> V[更新子编译状态]
    V --> W{编译成功?}
    W -- 失败 --> I
    W -- 成功 --> X[将结果加入链接队列]
    X --> Y[结束]
    
    style A fill:#9f9,stroke:#333
    style C fill:#f99,stroke:#333
    style I fill:#f99,stroke:#333
    style Y fill:#9f9,stroke:#333
