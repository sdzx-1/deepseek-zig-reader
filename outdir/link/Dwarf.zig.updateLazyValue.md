flowchart TD
    A[开始 updateLazyValue] --> B[初始化 wip_nav 结构]
    B --> C[设置 defer 清理操作]
    C --> D{根据 value_index 类型进行分支}
    
    D -->|int| E[处理整数类型]
    D -->|err| F[处理错误类型]
    D -->|error_union| G[处理错误联合类型]
    D -->|enum_literal| H[处理枚举字面量]
    D -->|ptr| I[处理指针类型]
    D -->|slice| J[处理切片类型]
    D -->|opt| K[处理可选类型]
    D -->|aggregate| L[处理聚合类型]
    D -->|un| M[处理联合类型]
    D -->|...其他类型分支...| N[其他类型处理]
    
    E --> E1[写入大整数常量值]
    E1 --> E2[引用类型]
    E --> O
    
    F --> F1[写入错误值]
    F1 --> F2[引用类型]
    F --> O
    
    G --> G1[处理错误和有效载荷]
    G1 --> G2[写入错误字段]
    G2 --> G3[处理有效载荷字段]
    G3 --> G4[引用类型]
    G --> O
    
    H --> H1[写入字符串常量值]
    H1 --> H2[引用枚举字面量类型]
    H --> O
    
    I --> I1[计算基地址和偏移]
    I1 --> I2[写入位置信息]
    I2 --> I3[引用指针类型]
    I --> O
    
    J --> J1[处理指针和长度字段]
    J1 --> J2[引用切片类型]
    J --> O
    
    K --> K1[处理 has_value 字段]
    K1 --> K2[处理有效值字段]
    K2 --> K3[引用可选类型]
    K --> O
    
    L --> L1[遍历结构体/元组/数组字段]
    L1 --> L2[写入每个字段的值]
    L2 --> L3[引用聚合类型]
    L --> O
    
    M --> M1[处理联合标签和字段]
    M1 --> M2[写入选定字段的值]
    M2 --> M3[引用联合类型]
    M --> O
    
    N --> O
    
    O[写入终止标记]
    O --> P[更新 debug_info 条目]
    P --> Q[函数结束]
    
    style A fill:#f9f,stroke:#333
    style Q fill:#f9f,stroke:#333
    classDef branch fill:#f96,stroke:#333
    class E,F,G,H,I,J,K,L,M,N branch
