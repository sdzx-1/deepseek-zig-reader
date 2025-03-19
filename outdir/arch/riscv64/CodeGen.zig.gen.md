graph TD
    A[开始gen函数] --> B[检查required_features]
    B --> C{所有特性存在?}
    C -- 否 --> D[返回错误]
    C -- 是 --> E{fn_info.cc是否为naked?}
    E -- 否 --> F[生成prologue调试标记]
    F --> G[创建backpatch伪指令]
    G --> H{处理ret_mcv.long类型}
    H -- indirect --> I[分配栈帧并保存返回值地址]
    H -- 其他 --> J[直接生成函数体]
    I --> J
    J --> K[生成函数体代码]
    K --> L[处理exitlude_jump_relocs重定位]
    L --> M[生成epilogue调试标记]
    M --> N[创建恢复用backpatch伪指令]
    N --> O[生成ret指令]
    O --> P[计算帧布局frame_layout]
    P --> Q[设置栈分配backpatch]
    Q --> R[设置ra/fp保存backpatch]
    R --> S[设置ra/fp恢复backpatch]
    S --> T{需要保存寄存器?}
    T -- 是 --> U[设置寄存器保存/恢复backpatch]
    T -- 否 --> V[跳过寄存器保存]
    U --> V
    V --> W[结束非naked分支]
    E -- 是 --> X[生成prologue调试标记]
    X --> Y[直接生成函数体]
    Y --> Z[生成epilogue调试标记]
    W --> AA[添加行号调试信息]
    Z --> AA
    AA --> AB[函数结束]
    D --> AB
