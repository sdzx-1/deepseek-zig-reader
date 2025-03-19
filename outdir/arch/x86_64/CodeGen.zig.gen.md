flowchart TD
    A[开始] --> B{fn_info.cc != .naked?}
    B -->|是| C[生成push rbp等prologue指令]
    C --> D[处理返回值存储]
    D --> E{fn_info.is_var_args?}
    E -->|是| F[根据调用约定生成可变参数处理代码]
    E -->|否| G[跳过可变参数处理]
    F --> G
    G --> H[插入调试信息prologue_end]
    H --> I[生成函数体genBody]
    I --> J[处理epilogue]
    J --> K{是否有epilogue_relocs?}
    K -->|是| L[恢复栈和寄存器,生成ret指令]
    K -->|否| M[直接生成ret指令]
    L --> M
    M --> N[处理栈对齐和调整]
    N --> O[保存寄存器列表]
    O --> P[插入调试信息epilogue_begin]
    P --> Q[结束]
    B -->|否| R[生成函数体genBody]
    R --> S[插入调试信息epilogue_begin]
    S --> Q
    Q --> T[插入结束调试信息]
