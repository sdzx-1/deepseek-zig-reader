graph TD
    A[开始] --> B{ptr_depth == 0?}
    B -->|是| C[写入...并返回根步骤]
    B -->|否| D[确定result_kind]
    D --> E{want_kind存在?}
    E -->|是| F[need_kind = want_kind]
    E -->|否| G[need_kind = result_kind]
    F --> H{need_kind是rvalue且result_kind是lvalue?}
    G --> H
    H -->|是| I[写入 &]
    H -->|否| J[处理derivation类型]
    I --> J
    J --> K[根据derivation类型递归处理]
    K -->|例如: .eu_payload_ptr| L[写入 ( 递归调用 写入 catch unreachable )]
    K -->|例如: .opt_payload_ptr| M[递归调用后写入 .?]
    K -->|例如: .field_ptr| N[递归调用后处理结构体/联合体/指针字段]
    K -->|例如: .elem_ptr| O[递归调用后写入 [索引]]
    K -->|例如: .offset_and_cast| P[写入类型转换和偏移量]
    K -->|其他类型| Q[处理根步骤策略]
    Q --> R{root_strat类型?}
    R -->|.str| S[写入字符串]
    R -->|.print_val| T[根据derivation类型打印值]
    T -->|.int| U[写入@ptrFromInt]
    T -->|.nav_ptr| V[写入导航指针名]
    T -->|.uav_ptr| W[写入类型和值]
    T -->|其他类型| X[未处理情况]
    J --> Y{need_kind是lvalue且result_kind是rvalue?}
    Y -->|是| Z[写入 .*]
    Y -->|否| AA[返回结果]
    Z --> AA
    C --> AA
    AA[返回root_or_null或derivation]
