flowchart TD
    A[开始] --> B[初始化和参数提取]
    B --> C{是模块级汇编?\n(is_global_assembly)}
    C -->|是| D[验证outputs_len=0]
    D --> E[验证inputs_len=0]
    E --> F[验证clobbers_len=0]
    F --> G[添加全局汇编]
    G --> H[返回.void_value]
    
    C -->|否| I[确保运行时块\n(requireRuntimeBlock)]
    I --> J[准备输出参数]
    J --> K[遍历outputs]
    K --> L{当前output是类型?}
    L -->|是| M[处理类型\n设置arg为none]
    L -->|否| N[解析操作数\n(resolveInst)]
    M --> O[收集约束和名称]
    N --> O
    O --> P{还有更多output?}
    P -->|是| K
    P -->|否| Q[准备输入参数]
    
    Q --> R[遍历inputs]
    R --> S[解析操作数]
    S --> T{类型是comptime_int/comptime_float?}
    T -->|是| U[类型转换\n(coerce)]
    T -->|否| V[保留原值]
    U --> W[收集约束和名称]
    V --> W
    W --> X{还有更多input?}
    X -->|是| R
    X -->|否| Y[处理clobbers]
    
    Y --> Z[遍历clobbers]
    Z --> AA[收集名称]
    AA --> AB{还有更多clobber?}
    AB -->|是| Z
    AB -->|否| AC[生成Air指令]
    
    AC --> AD[组装Air.Asm结构]
    AD --> AE[填充extra数据]
    AE --> AF[返回asm_air]
    
    style A stroke:#333,stroke-width:2px
    style H stroke:#f00
    style AF stroke:#090
