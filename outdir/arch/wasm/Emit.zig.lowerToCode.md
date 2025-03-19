graph TD
    A[开始] --> B[初始化变量: mir, code, wasm, comp等]
    B --> C[进入主循环]
    C --> D{检查tags[inst]}
    D -->|.dbg_epilogue_begin| E[返回]
    D -->|.block, .loop| F[处理block/loop指令]
    D -->|.uav_ref| G[处理uav_ref指令]
    D -->|.uav_ref_off| H[处理uav_ref_off指令]
    D -->|.nav_ref| I[处理nav_ref指令]
    D -->|.nav_ref_off| J[处理nav_ref_off指令]
    D -->|.func_ref| K[处理func_ref指令]
    D -->|.dbg_line| L[跳过调试行]
    D -->|.errors_len| M[写入错误数量]
    D -->|.error_name_table_ref| N[处理错误名称表引用]
    D -->|.br_if, .br等| O[处理分支指令]
    D -->|.local_get等| P[处理局部变量操作]
    D -->|.br_table| Q[处理跳转表]
    D -->|.call_nav| R[处理导航调用]
    D -->|.call_indirect| S[处理间接调用]
    D -->|.call_tag_name| T[处理标签名调用]
    D -->|.call_intrinsic| U[处理内置函数调用]
    D -->|.global_set_sp| V[设置全局栈指针]
    D -->|.f32_const等常量| W[写入浮点/整数常量]
    D -->|内存操作指令| X[处理加载/存储指令]
    D -->|单操作符指令| Y[直接写入操作码]
    D -->|.misc_prefix| Z[处理Misc前缀操作]
    D -->|.simd_prefix| AA[处理SIMD指令]
    D -->|.atomics_prefix| AB[处理原子操作]

    F --> AC[写入block类型]
    G --> AD[区分对象/可执行模式]
    H --> AD
    N --> AE[写入地址或生成重定位]
    Q --> AF[写入跳转表数据]
    R --> AG[生成函数调用重定位]
    S --> AH[处理函数类型索引]
    U --> AI[生成符号调用]
    X --> AJ[编码内存参数]
    Z --> AK[处理数据段/内存初始化]
    AA --> AL[处理SIMD操作码]
    AB --> AM[处理原子操作码]

    AC --> C
    AD --> C
    AE --> C
    AF --> C
    AG --> C
    AH --> C
    AI --> C
    AJ --> C
    AK --> C
    AL --> C
    AM --> C

    style A fill:#90EE90,stroke:#006400
    style E fill:#FFA07A,stroke:#8B0000
