graph TD
    Start[开始] --> A[添加--no-default-config]
    A --> B{clang_passthrough_mode?}
    B -- 否且支持诊断 --> C[添加-fno-caret-diagnostics]
    B -- 是 --> D[跳过诊断选项]
    C --> E{目标需要集成汇编器?}
    D --> E
    E -- 是 --> F[添加-integrated-as]
    E -- 否 --> G[跳过集成汇编选项]
    F --> H[添加目标三元组-target]
    G --> H
    H --> I{ARM架构?}
    I -- 是 --> J[添加-mthumb/-mno-thumb]
    I -- 否 --> K[跳过ARM选项]
    J --> L{存在MABI?}
    K --> L
    L -- 是且非汇编文件 --> M[添加-mabi参数]
    L -- 否 --> N[跳过MABI选项]
    M --> O{支持浮点ABI参数?}
    N --> O
    O -- 是 --> P[添加float-abi参数]
    O -- 否 --> Q[跳过浮点选项]
    P --> R{支持PIC?}
    Q --> R
    R -- 是 --> S[添加-fPIC/-fno-PIC]
    R -- 否 --> T[跳过PIC选项]
    S --> U{MinGW Unicode入口?}
    T --> U
    U -- 是 --> V[添加-municode]
    U -- 否 --> W[跳过MinGW选项]
    V --> X[处理代码模型]
    W --> X
    X --> Y[处理调试格式]
    Y --> Z{strip?}
    Z -- 是 --> AA[无操作]
    Z -- codeview --> AB[添加-g -gcodeview]
    Z -- dwarf --> AC[添加对应dwarf参数]
    AB --> AD[处理LTO选项]
    AC --> AD
    AA --> AD
    AD --> AE[添加依赖文件参数]
    AE --> AF{文件类型是否为汇编?}
    AF -- 否 --> AG[添加-hosted/freestanding参数]
    AF -- 是 --> AH[跳过非汇编选项]
    AG --> AI[处理红区/栈保护/内置函数等]
    AH --> AJ[处理汇编专用参数]
    AI --> AK[添加标准库包含路径]
    AJ --> AL[处理RISC-V/MIPS架构参数]
    AK --> AM{是否是C家族文件?}
    AL --> AM
    AM -- 是 --> AN[处理拼写检查/CPU特性/OS参数]
    AM -- 否 --> AO[跳过C选项]
    AN --> AP[处理sanitizer/优化模式]
    AO --> AP
    AP --> AQ[添加全局参数]
    AQ --> End[结束]
