graph TD
    A[开始 writeStabs] --> B[定义 writeFuncStab 内部函数]
    B --> C[初始化 index 和 n_strx]
    C --> D{检查是否有 compile_unit?}
    D -- 是 --> E[获取 comp_dir 和 tu_name]
    E --> F[写入 N_SO comp_dir]
    F --> G[复制 comp_dir 到 strtab]
    G --> H[写入 N_SO tu_name]
    H --> I[复制 tu_name 到 strtab]
    I --> J[写入 N_OSO path]
    J --> K[处理路径拼接（在存档或直接路径）]
    K --> L[遍历 symbols.items]
    L --> M{符号符合条件?}
    M -- 是 --> N{是代码段?}
    N -- 是 --> O[调用 writeFuncStab]
    O --> P[index +=4]
    N -- 否 --> Q{全局可见性?}
    Q -- 是 --> R[写入 N_GSYM]
    Q -- 否 --> S[写入 N_STSYM]
    R & S --> T[index +=1]
    T --> U[继续遍历]
    M -- 否 --> U
    U --> L
    L --> V[遍历结束]
    V --> W[写入关闭的 N_SO]
    D -- 否 --> X[遍历 stab_files]
    X --> Y[获取每个文件的 comp_dir, tu_name, oso_path]
    Y --> Z[写入 N_SO comp_dir]
    Z --> AA[复制 comp_dir 到 strtab]
    AA --> AB[写入 N_SO tu_name]
    AB --> AC[复制 tu_name 到 strtab]
    AC --> AD[写入 N_OSO path]
    AD --> AE[复制 oso_path 到 strtab]
    AE --> AF[遍历文件中的 stabs]
    AF --> AG{符号符合条件?}
    AG -- 是 --> AH{是函数?}
    AH -- 是 --> AI[调用 writeFuncStab]
    AI --> AJ[index +=4]
    AH -- 否 --> AK{全局可见性?}
    AK -- 是 --> AL[写入 N_GSYM]
    AK -- 否 --> AM[写入 N_STSYM]
    AL & AM --> AN[index +=1]
    AN --> AO[继续遍历]
    AG -- 否 --> AO
    AO --> AF
    AF --> AP[遍历结束]
    AP --> AQ[写入关闭的 N_SO]
    AQ --> X
    X --> AR[所有文件处理完成]
    W & AR --> AZ[函数结束]
