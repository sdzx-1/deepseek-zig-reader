graph TD
    A[开始] --> B[初始化变量: gpa, pt, zcu, ip, src_node_offset, src]
    B --> C[处理标量cases]
    C --> D[循环处理每个标量case]
    D --> E[调用validateSwitchItemError]
    E --> F[将结果加入case_vals]
    F --> D
    D -->|标量循环结束| G[处理多cases]
    G --> H[循环处理每个多case]
    H --> I[读取items_len和ranges_len]
    I --> J[调用validateSwitchNoRange]
    J --> K[循环处理每个item]
    K --> L[调用validateSwitchItemError]
    L --> M[将结果加入case_vals]
    M --> K
    K -->|item循环结束| H
    H -->|多case循环结束| N[解析错误集合类型]
    N --> O{是否为anyerror类型?}
    O -->|是| P{是否有else分支?}
    P -->|否| Q[生成错误: 缺少else分支]
    P -->|是| R[返回anyerror类型]
    O -->|否| S[遍历错误集合中的错误名称]
    S --> T{存在未处理的错误且无else?}
    T -->|是| U[生成错误提示]
    T -->|否| V{有else分支且所有错误已处理?}
    V -->|是| W[检查else分支合法性]
    W --> X{else分支是否合法?}
    X -->|否| Y[生成错误: 无效else]
    X -->|是| Z[创建错误集合类型]
    V -->|否| Z
    Z --> AA[返回错误集合类型或null]
    Q --> AB[结束]
    R --> AB
    Y --> AB
    AA --> AB
