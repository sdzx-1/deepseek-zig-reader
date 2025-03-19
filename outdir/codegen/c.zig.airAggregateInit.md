graph TD
    A[开始] --> B[获取类型和元素信息]
    B --> C[解析元素到resolved_elements数组]
    C --> D[处理大对象iterateBigTomb]
    D --> E[分配局部变量local]
    E --> F{判断聚合类型}
    
    F -->|数组/向量| G[遍历元素初始化每个索引]
    G --> G1[处理哨兵值（如果存在）]
    
    F -->|结构体| H{结构体布局类型}
    H -->|auto/extern| H1[按运行时顺序初始化字段]
    H -->|packed| H2[位操作打包字段到整数]
    
    F -->|元组| I[遍历元组字段初始化非编译时字段]
    
    G1 --> J[返回local]
    H1 --> J
    H2 --> J
    I --> J
    
    J[结束]
    
    style A fill:#90EE90,stroke:#006400
    style J fill:#FFA07A,stroke:#8B0000
