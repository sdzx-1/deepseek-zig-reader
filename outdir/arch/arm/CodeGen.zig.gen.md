graph TD
    A[开始 gen 函数] --> B{调用约定是否为naked?}
    B -->|否| C[生成 push {fp, lr}]
    C --> D[生成 mov fp, sp]
    D --> E[生成 sub sp, sp, #reloc]
    E --> F[标记 r4 为已分配]
    F --> G{ret_mcv 是 stack_offset?}
    G -->|是| H[分配栈空间并保存 r0 到栈]
    H --> I[更新 ret_mcv 为栈偏移]
    G -->|否| I
    I --> J[遍历所有参数]
    J --> K{参数是寄存器类型?}
    K -->|是| L[分配栈空间并复制寄存器参数到栈]
    L --> M[更新参数为栈偏移]
    K -->|否| M
    M --> N[继续遍历下一个参数]
    N --> J
    J --> O[生成 dbg_prologue_end]
    O --> P[生成函数体 genBody]
    P --> Q[回填 push 指令的寄存器列表]
    Q --> R[计算总栈空间并回填 sub 指令]
    R --> S[生成 dbg_epilogue_begin]
    S --> T{存在退出跳转重定位?}
    T -->|是| U[移除冗余跳转或回填跳转地址]
    U --> V[生成 mov sp, fp 和 pop {fp, pc}]
    T -->|否| V
    V --> W[生成 dbg_line 结束标记]
    B -->|是| X[生成 dbg_prologue_end]
    X --> Y[生成函数体 genBody]
    Y --> Z[生成 dbg_epilogue_begin]
    Z --> W
    W --> END[函数结束]
