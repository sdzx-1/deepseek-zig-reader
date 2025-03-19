graph TD
    A[开始flushModule] --> B[初始化跟踪器和进度节点]
    B --> C[激活PerThread上下文]
    C --> D[循环更新所有UAV]
    D --> E[初始化Flush结构]
    E --> F[生成ABI定义和zig.h内容]
    F --> G[处理全局汇编代码]
    G --> H[初始化延迟类型池和错误声明]
    H --> I[收集所有导出名称]
    I --> J{遍历uavs/navs?}
    J -->|是| K[处理AV块和导出状态]
    K --> J
    J -->|否| L[刷新C类型池]
    L --> M[填充类型缓冲区]
    M --> N[填充延迟声明缓冲区]
    N --> O[处理所有代码块]
    O --> P[写入文件]
    P --> Q[处理文件错误]
    Q --> R[结束]

    style A stroke:#333,stroke-width:2px
    style R stroke:#0f0,stroke-width:2px
    style Q stroke:#f00,stroke-width:2px

    subgraph 循环结构
        D -->|i < count| D
        J -->|遍历uavs/navs| J
    end

    subgraph 错误处理
        P -->|捕获错误| Q
    end
