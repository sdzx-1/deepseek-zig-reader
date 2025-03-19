flowchart TD
    A[开始: structInitAnon] --> B[分配 types, values, names 数组]
    B --> C[初始化变量: any_values=false, extra_index=extra_end]
    C --> D[循环处理每个字段]
    D --> E{判断 kind 类型}
    E -- anon_init --> F[获取匿名初始化项]
    E -- typed_init --> G[获取类型化初始化项并解析字段名]
    G --> H
    F --> H[获取字段名并存入 names]
    H --> I[解析 init 指令]
    I --> J[检查字段类型是否为 opaque]
    J -- 是 --> K[生成错误信息并返回]
    J -- 否 --> L[记录字段类型到 types]
    L --> M{是否有初始值?}
    M -- 是 --> N[存入 values 并标记 any_values=true]
    M -- 否 --> O[标记 values 为 none 并记录 runtime_index]
    O --> D
    N --> D
    D -- 所有字段处理完成 --> P[计算类型哈希 type_hash]
    P --> Q[创建或获取结构体类型]
    Q -- 新类型 --> R[设置字段名、类型、初始值]
    R --> S[创建命名空间并提交作业]
    Q -- 已有类型 --> T[直接使用]
    S --> U
    T --> U
    U --> V{存在 runtime_index?}
    V -- 否 --> W[生成常量并返回]
    V -- 是 --> X[进入运行时代码块]
    X --> Y{是否 is_ref?}
    Y -- 是 --> Z[分配内存并处理字段存储]
    Z --> AA[返回指针常量]
    Y -- 否 --> AB[收集所有字段引用并生成聚合初始化]
    AB --> AC[返回聚合结果]
