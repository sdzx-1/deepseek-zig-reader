flowchart TD
    A[开始] --> B{ctype.toPoolIndex()是否为null?}
    B -->|是| C[返回 .basic = ctype.index]
    B -->|否| D[获取pool_index和item]
    D --> E{switch(item.tag)}
    
    E -->|.pointer| F[返回指针类型, 设置elem_ctype]
    E -->|.pointer_const| G[返回指针类型, 设置elem_ctype和const]
    E -->|.pointer_volatile| H[返回指针类型, 设置elem_ctype和volatile]
    E -->|.pointer_const_volatile| I[返回指针类型, 设置elem_ctype, const和volatile]
    
    E -->|.aligned| J[获取extra数据, 返回对齐类型]
    E -->|.array_small| K[获取extra数据, 返回小数组类型]
    E -->|.array_large| L[获取extra数据, 返回大数组类型]
    E -->|.vector| M[获取extra数据, 返回向量类型]
    
    E -->|.fwd_decl_*_anon| N[获取extra_trail, 返回匿名前向声明结构/联合]
    E -->|.fwd_decl_*| O[返回命名前向声明结构/联合]
    
    E -->|.aggregate_*_anon| P[获取extra_trail, 返回匿名聚合结构/联合]
    E -->|.aggregate_*_packed_anon| Q[返回带packed的匿名聚合结构/联合]
    E -->|.aggregate_*| R[获取extra_trail, 返回命名聚合结构/联合]
    E -->|.aggregate_*_packed| S[返回带packed的命名聚合结构/联合]
    
    E -->|.function| T[获取extra_trail, 返回函数类型]
    E -->|.function_varargs| U[获取extra_trail, 返回带可变参数的函数类型]
