flowchart TD
    A[开始] --> B[检查target.os.tag是否为WASI]
    B -->|是| C[设置wasi_exec_model]
    B -->|否且wasi_exec_model不为空| D[返回错误WasiExecModelRequiresWasi]
    C --> E[处理shared_memory逻辑]
    E --> F{目标是否是WebAssembly?}
    F -->|否| G[检查shared_memory是否为true]
    G -->|是| H[返回错误SharedMemoryIsWasmOnly]
    G -->|否| I[设置shared_memory=false]
    F -->|是| J{输出模式是否为Obj?}
    J -->|是| K[检查shared_memory是否为true]
    K -->|是| L[返回错误ObjectFilesCannotShareMemory]
    K -->|否| M[设置shared_memory=false]
    J -->|否| N{是否支持atomics和bulk_memory?}
    N -->|否| O[检查shared_memory是否为true]
    O -->|是| P[返回错误SharedMemoryRequiresAtomicsAndBulkMemory]
    O -->|否| Q[设置shared_memory=false]
    N -->|是| R{是否有非单线程选项?}
    R -->|是| S[检查shared_memory是否为false]
    S -->|是| T[返回错误ThreadsRequireSharedMemory]
    S -->|否| U[设置shared_memory=true]
    R -->|否| V[使用选项值或默认false]
    
    E --> W[处理use_lib_llvm逻辑]
    W --> X{是否编译了LLVM支持?}
    X -->|否| Y[检查use_lib_llvm是否为true]
    Y -->|是| Z[返回错误LlvmLibraryUnavailable]
    Y -->|否| AA[设置use_lib_llvm=false]
    X -->|是| AB[使用选项值或默认true]
    
    W --> AC[确定root_optimize_mode]
    AC --> AD[处理use_llvm逻辑]
    AD --> AE{是否有需要编译的Zig代码?}
    AE -->|否| AF[设置use_llvm=false]
    AE -->|是| AG{是否生成LLVM IR/BC?}
    AG -->|是| AH[强制use_llvm=true并检查目标支持]
    AH --> AI{LLVM是否支持目标?}
    AI -->|否| AJ[返回错误LlvmLacksTargetSupport]
    AG -->|否| AK{LLVM是否支持目标?}
    AK -->|否| AL[检查use_llvm是否为true]
    AL -->|是| AM[返回错误LlvmLacksTargetSupport]
    AK -->|是| AN{Zig是否支持目标?}
    AN -->|否| AO[强制use_llvm=true]
    AN -->|是| AP[根据优化模式和后端稳定性决定]
    
    AD --> AQ[处理链接器选择]
    AQ --> AR{是否支持LLD?}
    AR -->|否| AS[设置use_lld=false]
    AR -->|是| AT{是否启用LTO?}
    AT -->|是| AU[强制use_lld=true]
    AT -->|否| AV[根据其他条件决定]
    
    AQ --> AW[选择C前端]
    AW --> AX{是否编译了LLVM?}
    AX -->|否| AY[强制使用Aro]
    AX -->|是| AZ[根据选项选择Clang或Aro]
    
    AQ --> BA[处理LTO模式]
    BA --> BB{是否使用LLD?}
    BB -->|否| BC[LTO必须为none]
    BB -->|是| BD[使用选项值或默认none]
    
    AQ --> BE[处理libc/libcpp/libunwind链接]
    BE --> BF{是否必须链接libc?}
    BF -->|是| BG[强制link_libc=true]
    BF -->|否| BH[根据其他条件判断]
    
    AQ --> BI[确定链接模式]
    BI --> BJ{目标是否支持动态链接?}
    BJ -->|否| BK[强制static链接]
    BJ -->|是| BL[根据输出模式和依赖库决定]
    
    BI --> BM[处理PIE设置]
    BM --> BN{目标是否需要PIE?}
    BN -->|是| BO[强制pie=true]
    BN -->|否| BP[使用选项值或默认false]
    
    BI --> BQ[处理调试信息]
    BQ --> BR{是否启用strip?}
    BR -->|是| BS[设置debug_format=strip]
    BR -->|否| BT[根据目标格式选择调试格式]
    
    BI --> BU[处理错误追踪]
    BU --> BV{后端是否支持错误追踪?}
    BV -->|否| BW[强制关闭错误追踪]
    BV -->|是| BX[根据优化模式决定]
    
    BI --> BY[处理动态库导出函数]
    BY --> BZ{是否是Windows?}
    BZ -->|否| CA[禁止dll_export_fns]
    BZ -->|是| CB[根据rdynamic和输出模式决定]
    
    BI --> CC[收集所有配置项]
    CC --> CD[返回Config结构体]
    
    style A fill:#9f9,stroke:#333,stroke-width:2px
    style D fill:#f99,stroke:#333,stroke-width:2px
    style H fill:#f99,stroke:#333,stroke-width:2px
    style L fill:#f99,stroke:#333,stroke-width:2px
    style P fill:#f99,stroke:#333,stroke-width:2px
    style T fill:#f99,stroke:#333,stroke-width:2px
    style Z fill:#f99,stroke:#333,stroke-width:2px
    style AJ fill:#f99,stroke:#333,stroke-width:2px
    style AM fill:#f99,stroke:#333,stroke-width:2px
