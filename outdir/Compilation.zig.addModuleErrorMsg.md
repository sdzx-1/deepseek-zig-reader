graph TD
    Start[开始] --> A[获取gpa和ip]
    A --> B[升级错误源位置err_src_loc]
    B --> C{能否加载错误源文件?}
    C --> |失败| D[构造文件路径错误消息]
    D --> E[添加根错误消息到eb]
    E --> Return[返回]
    C --> |成功| F[获取错误span和位置信息]
    F --> G{存在reference_trace_root?}
    G --> |否| H[直接处理主错误]
    G --> |是| I[解析所有引用]
    I --> J[初始化seen哈希表和ref_traces列表]
    J --> K[遍历引用链]
    K --> L{是否达到最大引用数或循环?}
    L --> |是| M[记录未显示的引用数量]
    L --> |否| N[获取引用位置和名称]
    N --> O[添加引用到ref_traces]
    O --> K
    K --> |循环结束| P[处理剩余引用计数]
    P --> H
    H --> Q[构造主错误源位置信息]
    Q --> R[添加引用跟踪到eb]
    R --> S[初始化notes哈希表]
    S --> T[遍历所有note]
    T --> U{是否与主错误位置重复?}
    U --> |是| V[标记不显示源码行]
    U --> |否| W[保留源码行]
    W --> X[插入/更新哈希表]
    X --> Y{遍历结束?}
    Y --> |否| T
    Y --> |是| Z[添加根错误到eb]
    Z --> AA[分配notes存储空间]
    AA --> AB[填充去重后的notes]
    AB --> End[结束]
    Return --> End
