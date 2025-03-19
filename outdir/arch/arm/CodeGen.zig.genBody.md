flowchart TD
    A[开始 genBody] --> B[初始化变量 pt, zcu, ip, air_tags]
    B --> C[遍历 body 中的每个指令 inst]
    C --> D{检查指令是否未使用且无需处理}
    D -- 是 --> C
    D -- 否 --> E[保存旧的 air_bookkeeping]
    E --> F[确保处理容量]
    F --> G[重置 reused_operands]
    G --> H[根据 air_tags 处理指令]
    H --> I[调用对应处理函数或返回错误]
    I --> J{是否所有分支处理完成？}
    J -- 是 --> K[检查寄存器和 air_bookkeeping 状态]
    K --> L{是否有未更新 bookkeeping？}
    L -- 是 --> M[触发运行时 panic]
    L -- 否 --> C
    C --> N[所有指令处理完毕]
    N --> O[结束函数]
    
    subgraph 指令处理分支
        H --> |add/sub/mul 等| I1[调用 airBinOp]
        H --> |ptr_add/ptr_sub| I2[调用 airPtrArithmetic]
        H --> |min/max| I3[调用 airMinMax]
        H --> |cmp_lt/cmp_eq 等| I4[调用 airCmp]
        H --> |未实现的指令| I5[返回 fail 或 panic]
        H --> |其他指令| I6[...更多处理分支...]
    end
