flowchart TD
    A[开始] --> B[初始化变量: pt, zcu, target, ctype_pool]
    B --> C[定义ExpectedContents结构体]
    C --> D[分配栈内存并获取allocator]
    D --> E{检查data.val是否为Undef?}
    E -- 是 --> F[分配并初始化undef_limbs]
    E -- 否 --> G[将data.val转换为BigInt]
    F --> H[截断undef_int并转换为BigInt]
    G --> H
    H --> I[验证int是否符合符号和位数]
    I --> J[确定c_bits和c_limb_info]
    J --> K{c_limb_info.count ==1?}
    K -- 是 --> L[处理符号和前缀]
    L --> M[选择数值格式(base/case)]
    M --> N[转换BigInt为字符串并输出]
    K -- 否 --> O[将BigInt转换为补码形式]
    O --> P[按字节序分解为多个C肢体]
    P --> Q[遍历每个肢体]
    Q --> R{是否为最高有效肢体且需要符号处理?}
    R -- 是 --> S[设置符号和类型为signed]
    R -- 否 --> T[保持类型为unsigned]
    S --> U[递归调用formatIntLiteral]
    T --> U
    U --> V[输出逗号分隔符]
    V --> Q
    Q --> W[所有肢体处理完成]
    W --> X[添加类型后缀]
    N --> X
    X --> Y[结束]
    
    style A fill:#90EE90,stroke:#006400
    style Y fill:#FFB6C1,stroke:#8B0000
    style K fill:#FFD700,stroke:#DAA520
    style E fill:#87CEEB,stroke:#4682B4
