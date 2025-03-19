graph TD
    A[开始] --> B[解析指令, 获取操作数 lhs 和 rhs]
    B --> C[类型检查和统一 resolved_type]
    C --> D[强制转换类型 casted_lhs 和 casted_rhs]
    D --> E{是否为整数类型?}
    E --> |是| F[检查算术操作合法性]
    E --> |否| F
    F --> G[尝试解析编译时值 maybe_lhs_val 和 maybe_rhs_val]
    G --> H{lhs 是零?}
    H --> |是| I[返回零值结果]
    H --> |否| J{rhs 是零或 undefined?}
    J --> |rhs 是零| K[报错: 除以零]
    J --> |rhs 是 undefined| L[报错: 使用未定义值]
    J --> |否| M{是否全部为编译时常量?}
    M --> |是| N[计算余数并检查]
    N --> O{余数为零?}
    O --> |否| P[报错: 存在余数]
    O --> |是| Q[计算结果并返回]
    M --> |否| R[确定运行时检查位置 runtime_src]
    R --> S{启用安全检查?}
    S --> |是| T[添加溢出和除以零安全检查]
    T --> U[生成 div_trunc 指令]
    U --> V[生成余数检查逻辑]
    V --> W{余数为零?}
    W --> |否| X[触发安全错误]
    W --> |是| Y[返回结果]
    S --> |否| Z[生成 div_exact 指令并返回]
