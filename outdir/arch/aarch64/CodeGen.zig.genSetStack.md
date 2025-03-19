graph TD
    A[开始genSetStack] --> B{检查mcv类型}
    B -->|.dead| C[触发unreachable]
    B -->|.unreach, .none| D[直接返回]
    B -->|.undef| E[安全模式?]
    E -->|否| D
    E -->|是| F{abi_size}
    F -->|1,2,4,8| G[递归调用自身设置立即数]
    F -->|其他| H[调用genInlineMemset]
    B -->|.compare_flags, .immediate, .ptr_stack_offset| I[复制到临时寄存器]
    I --> J[递归调用处理.register]
    B -->|.register| K{abi_size}
    K -->|1,2,4,8| L[生成对应存储指令strb/strh/str]
    K -->|其他| M[抛出TODO错误]
    B -->|.register_with_overflow| N[处理主寄存器]
    N --> O[处理溢出标志位]
    O --> P[生成cset指令]
    P --> Q[存储条件寄存器]
    B -->|.linker_load, .memory, .stack_argument_offset, .stack_offset| R{abi_size≤8?}
    R -->|是| I
    R -->|否| S[准备指针类型]
    S --> T[分配寄存器]
    T --> U[设置源地址]
    U --> V[设置目标地址]
    V --> W[设置长度]
    W --> X[调用genInlineMemcpy]
    X --> D
