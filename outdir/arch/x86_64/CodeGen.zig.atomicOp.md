graph TD
    A[开始atomicOp] --> B[锁定ptr_mcv寄存器]
    A --> C[锁定val_mcv寄存器]
    B --> D[生成ptr_mem]
    C --> D
    D --> E{ptr_mem.mod是.rm?}
    E -->|是| F[锁定mem寄存器]
    E -->|否| G[返回错误]
    F --> H[确定strat策略]
    H -->|strat=lock| I[选择MIR指令]
    I --> J[分配目标寄存器dst_reg]
    J --> K[生成mov/xadd等指令]
    K --> L{rmw_op是Sub且xadd?}
    L -->|是| M[生成neg指令]
    L -->|否| N[生成内存-寄存器操作]
    N --> O[返回结果或unreach]
    H -->|strat=loop| P{val_abi_size <=8?}
    P -->|是| Q[使用SSE或临时寄存器]
    Q --> R[生成mov指令加载到rax]
    R --> S[生成循环标签loop]
    S --> T{是否有rmw_op?}
    T -->|是| U[生成对应操作指令]
    T -->|否| V[生成比较交换指令]
    U --> V
    V --> W[生成lock cmpxchg]
    W --> X[生成条件跳转回loop]
    X --> Y[返回结果或unreach]
    P -->|否| Z[处理16字节分高低位]
    Z --> AA[加载低8字节到rax]
    AA --> AB[加载高8字节到rdx]
    AB --> AC[生成循环标签loop]
    AC --> AD{是否有rmw_op?}
    AD -->|是| AE[生成对应高低位操作]
    AD -->|否| AF[生成比较交换指令]
    AE --> AF
    AF --> AG[生成lock cmpxchg16b]
    AG --> AH[生成条件跳转回loop]
    AH --> AI[保存结果到临时内存]
    AI --> AJ[返回结果]
    H -->|strat=libcall| AK[返回未实现错误]
