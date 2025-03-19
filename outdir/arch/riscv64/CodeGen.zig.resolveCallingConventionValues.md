graph TD
    A[开始] --> B[分配param_types数组]
    B --> C[合并fn_info参数和var_args到param_types]
    C --> D[初始化result对象]
    D --> E{调用约定cc类型?}
    E --> |naked| F[检查args长度是否为0]
    F --> G[设置return_value和stack_align]
    G --> H[结束处理]
    E --> |riscv64_lp64/auto| I[参数数量检查>8?]
    I --> |是| J[返回错误: 参数过多]
    I --> |否| K[初始化寄存器和栈配置]
    K --> L[处理返回值类型]
    L --> M{返回值类型?}
    M --> |noreturn| N[return_value=unreach]
    M --> |无运行时位| O[return_value=none]
    M --> |其他类型| P[分类处理返回值寄存器]
    P --> Q[根据类型分配整数/浮点寄存器]
    Q --> R[组合return_value]
    R --> S[处理参数]
    S --> T[遍历param_types]
    T --> U{参数是否有运行时位?}
    U --> |否| V[arg=none]
    U --> |是| W[分类参数类型]
    W --> X{类型类别?}
    X --> |integer| Y[分配整数寄存器]
    X --> |float| Z[分配浮点寄存器]
    X --> |memory| AA[分配间接寄存器]
    X --> |其他| AB[返回错误: 不支持的类型]
    Y --> AC[记录寄存器到arg_mcv]
    Z --> AC
    AA --> AC
    AC --> AD{是否处理完所有参数?}
    AD --> |否| T
    AD --> |是| AE[计算栈对齐和大小]
    AE --> H
    J --> H
    AB --> H
    H --> AF[返回result]
