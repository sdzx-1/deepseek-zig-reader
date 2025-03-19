graph TD
    A[开始] --> B[初始化items_len, extra_len, limbs_len为0]
    B --> C[遍历ip.locals中的每个local]
    C --> D[累加items.len到items_len]
    D --> E[累加extra.len到extra_len]
    E --> F[累加limbs.len到limbs_len]
    F --> G{是否遍历完所有locals?}
    G -- 否 --> C
    G -- 是 --> H[计算items_size, extra_size, limbs_size]
    H --> I[计算total_size = @sizeOf(InternPool) + 各部分大小]
    I --> J[打印总内存统计信息]
    J --> K[创建TagStats结构体和counts哈希表]
    K --> L[遍历ip.locals中的每个local]
    L --> M[获取items和extra数据]
    M --> N[遍历每个item的tag和data]
    N --> O[根据tag类型计算bytes]
    O --> P[更新counts中的统计信息]
    P --> Q{是否处理完所有items?}
    Q -- 否 --> N
    Q -- 是 --> R{是否遍历完所有locals?}
    R -- 否 --> L
    R -- 是 --> S[按bytes降序排序counts]
    S --> T[截取前50个结果]
    T --> U[输出前50的tag统计]
    U --> V[结束]
    
    style A fill:#9f9,stroke:#333,stroke-width:2px
    style V fill:#f99,stroke:#333,stroke-width:2px
