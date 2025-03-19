graph TD
    A[开始zirArrayInit] --> B[获取pt, zcu, gpa]
    B --> C[解析指令数据inst_data]
    C --> D[提取参数args]
    D --> E{args.len >= 2?}
    E -- 是 --> F[解析结果类型result_ty]
    E -- 否 --> E1[触发断言失败]
    F --> G{result_ty是否已知?}
    G -- 未知 --> H[调用arrayInitAnon并返回]
    G -- 已知 --> I[确定array_ty和is_tuple]
    I --> J[分配resolved_args数组]
    J --> K[遍历元素i: 0..final_len-1]
    K --> L{是否i+2 > args.len?}
    L -- 是 --> M{是元组?}
    M -- 是 --> N[检查默认值或报错]
    M -- 否 --> O[使用哨兵值填充]
    L -- 否 --> P[解析参数arg]
    P --> Q[确定elem_ty]
    Q --> R[类型检查并存储到resolved_args]
    R --> S{是元组且comptime字段?}
    S -- 是 --> T[验证字段值是否匹配]
    S -- 否 --> K
    K --> U[处理下一个元素]
    U --> V{遍历完成?}
    V -- 否 --> K
    V -- 是 --> W{存在root_msg错误?}
    W -- 是 --> X[添加错误并返回]
    W -- 否 --> Y[检查所有参数是否comptime]
    Y --> Z{存在运行时索引?}
    Z -- 否 --> AA[生成编译时常量]
    Z -- 是 --> AB[需要运行时块]
    AB --> AC{是引用类型?}
    AC -- 是 --> AD[分配内存并存储元素]
    AC -- 否 --> AE[生成聚合初始化]
    AD --> AF[返回指针常量]
    AE --> AG[类型强制并返回]
    AA --> AH[返回常量引用]
