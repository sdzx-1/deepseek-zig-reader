flowchart TD
    A[开始 buildSharedObjects] --> B{检查是否有 LLVM}
    B -->|否| C[返回错误 ZigCompilerNotBuiltWithLLVMExtensions]
    B -->|是| D[初始化 ArenaAllocator 和缓存]
    D --> E[获取目标信息并设置缓存前缀]
    E --> F[创建缓存清单并添加哈希项]
    F --> G[加载 abilists 文件到缓存]
    G --> H{检查缓存命中?}
    H -->|是| I[调用 queueSharedObjects 并返回]
    H -->|否| J[生成新缓存目录和元数据]
    J --> K[查找目标架构和 ABI 匹配的索引]
    K --> L[确定目标 glibc 版本索引]
    L --> M{版本是否有效?}
    M -->|否| N[记录警告并返回错误]
    M -->|是| O[生成版本映射文件]
    O --> P[初始化汇编缓冲区]
    P --> Q[遍历所有库]
    Q --> R{库是否被移除?}
    R -->|是| Q
    R -->|否| S[处理函数符号包含]
    S --> T[处理符号名称和版本]
    T --> U[生成函数符号汇编代码]
    U --> V[处理数据对象包含]
    V --> W[生成数据对象汇编代码]
    W --> X[写入 .s 文件并构建共享库]
    X --> Q
    Q -->|循环结束| Y[写入缓存清单]
    Y --> Z[调用 queueSharedObjects 并返回]
