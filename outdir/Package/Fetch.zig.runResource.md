graph TD
    A[开始] --> B[初始化资源 (defer resource.deinit())]
    B --> C[创建临时目录]
    C --> D{临时目录创建成功?}
    D -- 是 --> E[解压资源到临时目录 (unpackResource)]
    D -- 否 --> E1[记录错误并返回 FetchFailed]
    E --> F{是否为Linux且需要Btrfs处理?}
    F -- 是 --> G[重新打开临时目录]
    F -- 否 --> H[加载清单文件 (loadManifest)]
    G --> H
    H --> I[应用过滤规则并验证文件]
    I --> J[计算包哈希 (computeHash)]
    J --> K[重命名临时目录到缓存]
    K --> L{重命名成功?}
    L -- 是 --> M[清理临时目录残留]
    L -- 否 --> L1[记录错误并返回 FetchFailed]
    M --> N[验证哈希是否匹配远程哈希?]
    N -- 匹配 --> O{是否启用递归?}
    N -- 不匹配 --> N1[记录哈希错误并返回 FetchFailed]
    O -- 是 --> P[为依赖项创建新任务 (queueJobsForDeps)]
    O -- 否 --> Q[结束]
    P --> Q
    E1 --> Q
    L1 --> Q
    N1 --> Q
