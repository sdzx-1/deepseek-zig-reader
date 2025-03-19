flowchart TD
    A[开始] --> B[断言子块最后指令是noreturn]
    B --> C[检查block_tag类型]
    C -->|block_tag是.block或.dbg_inline_block| D[判断merges.results.items.len]
    D -->|len == 0| E[处理无结果情况]
    E -->|block_tag是.block| F{是否需要调试作用域?}
    F -->|是| G[设置父块need_debug_scope]
    F -->|否| H[将子块指令追加到父块]
    H --> I[返回最后指令引用]
    E -->|block_tag是.dbg_inline_block| J[生成DbgInlineBlock AIR块]
    J --> K[返回block_inst引用]
    D -->|len == 1| L[检查最后指令是否为break]
    L -->|是且无需调试作用域| M[追加指令到父块并返回结果]
    L -->|否或需要调试| N[处理编译时已知值]
    N --> O{值是否编译时已知?}
    O -->|是| P[生成对应类型AIR块并返回结果]
    O -->|否| Q[继续处理常规块]
    D -->|len > 1| R[断言非comptime块]
    R --> S[解析合并类型]
    S --> T[检查comptime-only类型错误]
    T --> U[生成错误?]
    U -->|是| V[返回错误]
    U -->|否| W[验证运行时值]
    W --> X[生成AIR块]
    X --> Y[处理中断指令类型强制]
    Y --> Z[遍历每个br指令]
    Z --> AA{类型是否匹配?}
    AA -->|否| AB[生成类型强制子块]
    AB --> AC[更新br指令操作数]
    AA -->|是| AD[直接继续]
    AD --> Z
    AC --> Z
    Y --> AE[返回block_inst引用]
    I --> AE
    K --> AE
    M --> AE
    P --> AE
