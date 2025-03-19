flowchart TD
    A[开始 reifyStruct] --> B[初始化变量: pt, zcu, gpa, ip]
    B --> C[计算字段长度 fields_len]
    C --> D[初始化哈希器 hasher]
    D --> E[遍历每个字段]
    E --> F[提取字段信息: 名称、类型、默认值等]
    F --> G[哈希字段信息]
    G --> H[检查字段属性: comptime/默认值/对齐]
    H --> E
    E --> I[完成字段遍历]
    I --> J[尝试获取现有结构类型]
    J --> K{类型是否存在？}
    K --> |是| L[返回现有类型引用]
    K --> |否| M[创建WIP结构类型]
    M --> N[设置结构类型名称]
    N --> O[再次遍历每个字段]
    O --> P[检查字段名重复]
    P --> Q[处理字段对齐]
    Q --> R[处理comptime字段限制]
    R --> S[验证字段类型有效性]
    S --> T[设置字段类型和默认值]
    T --> O
    O --> U[处理packed布局的特殊逻辑]
    U --> V[计算总位宽并检查backing int]
    V --> W[创建类型命名空间]
    W --> X[队列类型解析和代码生成作业]
    X --> Y[完成结构类型构建]
    Y --> Z[返回新类型引用]

    P --> |字段名重复| P1[返回错误]
    Q --> |无效对齐| Q1[返回错误]
    R --> |extern/packed + comptime| R1[返回错误]
    S --> |无效类型（如opaque/noreturn）| S1[返回错误]
    U --> |packed布局验证失败| U1[返回错误]
    V --> |backing int类型不匹配| V1[返回错误]
    
    style A fill:#4CAF50,stroke:#388E3C
    style L fill:#2196F3,stroke:#1976D2
    style Z fill:#4CAF50,stroke:#388E3C
    style P1 fill:#FF5722,stroke:#E64A19
    style Q1 fill:#FF5722,stroke:#E64A19
    style R1 fill:#FF5722,stroke:#E64A19
    style S1 fill:#FF5722,stroke:#E64A19
    style U1 fill:#FF5722,stroke:#E64A19
    style V1 fill:#FF5722,stroke:#E64A19
