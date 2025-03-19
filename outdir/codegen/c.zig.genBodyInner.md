flowchart TD
    A[开始 genBodyInner] --> B[初始化变量: zcu, ip, air_tags, air_datas]
    B --> C[遍历 body 中的每个指令 inst]
    C --> D{是否在 naked 函数中?}
    D -->|是| E[调用 fail 返回错误]
    D -->|否| F{指令是否未被使用且无需处理?}
    F -->|是| C[跳过，继续下一个指令]
    F -->|否| G[根据 air_tags 选择处理分支]
    G --> H[处理各类指令（如 .add, .sub 调用 BinOp；.sqrt 调用 UnBuiltinCall）]
    H --> I{结果类型是否为 new_local?}
    I -->|是| J[记录映射关系到 value_map]
    I -->|否| K[直接存储结果或跳过]
    J --> C
    K --> C
    C --> L[所有指令处理完成?]
    L -->|是| M[结束/返回]
    L -->|否| C
    E --> M
    style A fill:#f9f,stroke:#333
    style M fill:#f96,stroke:#333
