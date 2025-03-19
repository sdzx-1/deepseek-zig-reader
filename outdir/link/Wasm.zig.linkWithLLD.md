graph TD
    A[开始: linkWithLLD] --> B[检查LLD链接器]
    B --> C[初始化配置参数]
    C --> D{是否有Zig代码?}
    D -- 是 --> E[生成模块对象路径]
    D -- 否 --> F[模块对象路径为null]
    E --> G[创建子进度节点]
    F --> G
    G --> H{是否为Obj模式?}
    H -- 是 --> I[直接复制对象文件]
    H -- 否 --> J[构造LLD命令行参数]
    J --> K[添加基本参数]
    K --> L[添加内存相关参数]
    L --> M[添加符号导出参数]
    M --> N[添加入口点参数]
    N --> O[添加目标平台参数]
    O --> P[添加库文件参数]
    P --> Q[处理链接输入项]
    Q --> R[调用LLD子进程]
    R --> S{子进程成功?}
    S -- 是 --> T[设置可执行权限]
    S -- 否 --> U[处理错误输出]
    U --> V[返回链接失败]
    T --> W{是否禁用缓存?}
    W -- 否 --> X[写入缓存摘要]
    W -- 是 --> Y[结束]
    X --> Z[更新缓存清单]
    Z --> Y
    I --> Y
    V --> Y
    Y[结束]
    
    subgraph 错误处理
        U --> V
    end
    
    subgraph 缓存处理
        W --> X
        X --> Z
    end
    
    style A fill:#4CAF50,stroke:#388E3C
    style Y fill:#FF5722,stroke:#E64A19
    style V fill:#F44336,stroke:#D32F2F
