flowchart TD
    A[开始] --> B[解析条件cond]
    B --> C[生成条件跳转reloc]
    C --> D{条件操作数是否死亡?}
    D -- 是 --> E[处理操作数死亡]
    D -- 否 --> F[保存父分支状态]
    E --> F
    F --> G[进入then分支]
    G --> H[处理then_body死亡操作数]
    H --> I[生成then_body代码]
    I --> J[恢复父状态]
    J --> K[执行else分支重定位]
    K --> L[进入else分支]
    L --> M[处理else_body死亡操作数]
    M --> N[生成else_body代码]
    N --> O[合并分支状态]
    O --> P[遍历else_branch条目]
    P --> Q{条目在then_branch存在?}
    Q -- 是 --> R[使用then值作为规范]
    Q -- 否 --> S[向上查找父分支值]
    R --> T[同步寄存器/内存状态]
    S --> T
    T --> U[遍历then_branch剩余条目]
    U --> V[向上查找父分支值]
    V --> W[同步寄存器/内存状态]
    W --> X[清理分支堆栈]
    X --> Y[完成指令返回]
    
    style A fill:#f9f,stroke:#333
    style Y fill:#bbf,stroke:#333
