graph TD
    A[开始] --> B[解析 resolved_operand_val]
    B --> C[处理标量案例循环]
    C --> D{scalar_i < scalar_cases_len?}
    D -- 是 --> E[获取当前标量案例信息]
    E --> F{操作数值是否匹配?}
    F -- 是 --> G[处理错误集合(err_set)]
    G --> H[调用 resolveProngComptime 并返回]
    F -- 否 --> I[scalar_i += 1]
    I --> D
    D -- 否 --> J[处理多案例循环]
    J --> K{multi_i < multi_cases_len?}
    K -- 是 --> L[获取多案例信息]
    L --> M[处理项目(items)]
    M --> N{遍历项目是否匹配?}
    N -- 是 --> O[处理错误集合(err_set)]
    O --> P[调用 resolveProngComptime 并返回]
    N -- 否 --> Q[处理范围(ranges)]
    Q --> R{遍历范围是否包含操作数?}
    R -- 是 --> S[处理错误集合(err_set)]
    S --> T[调用 resolveProngComptime 并返回]
    R -- 否 --> U[range_i += 1]
    U --> Q
    Q --> V[multi_i += 1]
    V --> K
    K -- 否 --> W{存在错误集合(err_set)?}
    W -- 是 --> X[处理错误解包]
    X --> Y{空枚举(empty_enum)?}
    W -- 否 --> Y
    Y -- 是 --> Z[返回 .void_value]
    Y -- 否 --> AA[处理特殊prong]
    AA --> AB[调用 resolveProngComptime 并返回]
