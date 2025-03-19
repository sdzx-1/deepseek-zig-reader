graph TD
    A[开始 emitInitMemoryFunction] --> B{检查 any_passive_inits}
    B -->|是| C{存在 init_memory_flag?}
    C -->|是| D[生成 block 结构 ($drop, $wait, $init)]
    D --> E[原子操作 i32.atomic.rmw.cmpxchg]
    E --> F[br_table 跳转]
    F -->|0: $init| G[处理段组]
    F -->|1: $wait| H[等待初始化完成]
    F -->|2: $drop| I[丢弃段]

    C -->|否| G

    G --> J[遍历 segment_groups]
    J --> K{是否是被动段?}
    K -->|是| L{是 BSS 段?}
    L -->|是| M[memory.fill 填充0]
    L -->|否| N[memory.init 初始化]
    K -->|否| J

    N --> O{是 TLS 且共享内存?}
    O -->|是| P[设置 __tls_base 全局变量]
    O -->|否| J

    G --> Q{存在 init_memory_flag?}
    Q -->|是| R[设置标志为2 (i32.atomic.store)]
    R --> S[atomic.notify 通知等待线程]
    S --> T[br 跳转到 $drop]
    T --> U[memory.atomic.wait 等待完成]
    U --> I

    Q -->|否| V[结束段组处理]

    I --> W[遍历段组丢弃非BSS/TLS被动段]
    W --> X{是否是被动段且非BSS/TLS?}
    X -->|是| Y[执行 data_drop]
    X -->|否| W

    V --> Z[结束函数体 (end)]
    Y --> Z
    Z --> AA[函数结束]
