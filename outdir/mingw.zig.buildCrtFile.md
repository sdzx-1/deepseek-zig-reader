graph TD
    A[开始] --> B{检查LLVM支持}
    B -->|否| C[返回错误]
    B -->|是| D[初始化Arena分配器]
    D --> E[获取目标平台]
    E --> F{检查目标架构是否为x86}
    F -->|是| G[设置unwind_tables为none]
    F -->|否| H[设置unwind_tables为async]
    H --> I{switch crt_file类型}
    G --> I
    I -->|crt2_o| J[初始化参数列表]
    J --> K{检查Unicode入口点}
    K -->|是| L[添加UNICODE宏定义]
    K -->|否| M[跳过宏定义]
    L --> N[设置crtexe.c源文件]
    M --> N
    N --> O[调用build_crt_file生成crt2.o]
    
    I -->|dllcrt2_o| P[初始化参数列表]
    P --> Q[设置crtdll.c源文件]
    Q --> R[调用build_crt_file生成dllcrt2.o]
    
    I -->|mingw32_lib| S[初始化c_source_files列表]
    S --> T[添加通用源文件]
    T --> U{检查目标架构}
    U -->|x86系列| V[添加x86专用文件]
    V --> W{是否为32位x86}
    W -->|是| X[添加x86_32文件]
    W -->|否| Y[跳过]
    U -->|ARM系列| Z[添加ARM专用文件]
    Z --> AA{是否为aarch64}
    AA -->|是| AB[添加arm64文件]
    AA -->|否| AC[添加arm32文件]
    U -->|其他架构| AD[panic]
    Y --> AE[处理winpthreads参数]
    X --> AE
    AB --> AE
    AC --> AE
    AE --> AF[添加winpthreads源文件]
    AF --> AG[调用build_crt_file生成mingw32.lib]
    
    O --> AH[结束]
    R --> AH
    AG --> AH
    AD --> AH
