graph TD
    A[开始 airCall] --> B{是 naked 函数?}
    B -->|是| C[返回 .none]
    B -->|否| D[获取 gpa 和 writer]
    D --> E[解析参数和 extra 数据]
    E --> F[分配 resolved_args 数组]
    F --> G[遍历参数处理每个 arg]
    G --> H{arg_ctype 是 void?}
    H -->|是| I[resolved_arg = none]
    H -->|否| J[解析 arg 实例]
    J --> K{ctype 是否匹配?}
    K -->|否| L[生成 memcpy 代码并更新 resolved_arg]
    K -->|是| M[保留原 resolved_arg]
    L --> G
    M --> G
    G --> N[处理 callee 实例]
    N --> O{callee 是已知函数?}
    O -->|是| P[处理函数修饰符和类型转换]
    O -->|否| Q[处理函数指针调用]
    P --> R[生成函数名或指针调用代码]
    Q --> R
    R --> S[生成参数列表和调用代码]
    S --> T{是否有返回值?}
    T -->|否| U[返回 .none]
    T -->|是| V[分配 result_local]
    V --> W{返回值是数组类型?}
    W -->|是| X[生成 memcpy 返回数组代码]
    W -->|否| Y[直接使用 result_local]
    X --> Z[返回 array_local]
    Y --> Z[返回 result_local]
    Z --> AA[结束]
