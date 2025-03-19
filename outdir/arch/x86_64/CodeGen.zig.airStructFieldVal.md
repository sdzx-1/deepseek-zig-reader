graph TD
    Start[开始 airStructFieldVal] --> A[获取ty_pl和extra数据]
    A --> B[获取container_ty和field_ty]
    B --> C{检查field_ty是否有运行时位}
    C -- 无 --> D[返回.none]
    C -- 有 --> E[获取src_mcv = resolveInst(operand)]
    E --> F{src_mcv类型}
    
    F -- register --> G[处理寄存器情况]
    G --> G1[锁定寄存器并检查重用]
    G1 --> G2{field_off是否为0?}
    G2 -- 是 --> G3[直接复制寄存器]
    G2 -- 否 --> G4[生成位移指令]
    G4 --> G5[截断寄存器]
    G5 --> G6[返回结果]
    
    F -- register_pair --> H[处理双寄存器情况]
    H --> H1[检查字段位偏移]
    H1 --> H2{字段是否跨寄存器?}
    H2 -- 否 --> H3[选择单个寄存器处理]
    H2 -- 是 --> H4[分配新寄存器对并拷贝]
    H4 --> H5[生成位移指令]
    H5 --> H6[截断高位寄存器]
    H6 --> G6
    
    F -- register_overflow --> I[处理溢出寄存器]
    I --> I1{字段索引是0或1?}
    I1 -- 0 --> I2[返回寄存器值]
    I1 -- 1 --> I3[生成EFLAGS操作]
    I3 --> G6
    
    F -- load_frame --> J[处理内存加载]
    J --> J1{字段偏移是8的倍数?}
    J1 -- 是 --> J2[直接加载内存]
    J2 --> J3[处理截断]
    J1 -- 否 --> J4[复杂位偏移加载]
    J4 --> J5[多步加载并位移]
    J5 --> J3
    J3 --> G6
    
    F -- else --> K[抛出未实现错误]
    
    G6 --> Z[调用finishAir返回结果]
    D --> Z
    H6 --> Z
    I2 --> Z
    I3 --> Z
    K --> Z
