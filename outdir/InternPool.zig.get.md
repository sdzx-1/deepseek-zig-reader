graph TD
    A[开始] --> B[调用 ip.getOrPutKey 获取或创建条目]
    B --> C[defer gop.deinit]
    C --> D{检查 key 类型}
    D -->|int_type| E[处理整数类型]
    D -->|ptr_type| F[处理指针类型]
    D -->|array_type| G[处理数组类型]
    D -->|vector_type| H[处理向量类型]
    D -->|opt_type| I[处理可选类型]
    D -->|error_union_type| J[处理错误联合类型]
    D -->|...其他类型分支...| K[...]
    
    F --> F1{ptr_type.flags.size == .slice?}
    F1 -->|是| F2[调整 key 并递归调用 ip.get]
    F2 --> F3[重新获取或创建条目]
    F3 --> F4[添加 slice 类型数据]
    F1 -->|否| F5[处理普通指针类型]
    
    G --> G1{数组长度是否可转换为 u32?}
    G1 -->|是| G2[添加小数组类型数据]
    G1 -->|否| G3[添加大数组类型数据]
    
    E --> E1[根据符号和位数生成标签]
    E1 --> E2[追加到 items]
    
    J --> J1{error_set_type 是否为 anyerror_type?}
    J1 -->|是| J2[添加 anyerror_union 类型数据]
    J1 -->|否| J3[添加普通 error_union 类型数据]
    
    K --> K1[处理其他类型...]
    
    F4 --> Z
    F5 --> Z
    G2 --> Z
    G3 --> Z
    E2 --> Z
    J2 --> Z
    J3 --> Z
    K1 --> Z
    
    Z[返回 gop.put 生成的索引]
    
    style A fill:#90EE90,stroke:#555
    style B fill:#ADD8E6,stroke:#555
    style C fill:#FFB6C1,stroke:#555
    style D fill:#FFD700,stroke:#555
    style Z fill:#90EE90,stroke:#555
