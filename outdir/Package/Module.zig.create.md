graph TD
    Start[开始] --> CheckAsserts[检查前置条件断言]
    CheckAsserts --> ResolveTarget[解析resolved_target]
    ResolveTarget --> DetermineOptimizeMode[确定optimize_mode]
    DetermineOptimizeMode --> DetermineStrip[确定strip]
    DetermineStrip --> CheckValgrind[检查Valgrind支持]
    CheckValgrind --> |不支持且启用| ReturnValgrindError[返回Valgrind错误]
    CheckValgrind --> |其他情况| DetermineValgrind[确定valgrind值]
    DetermineValgrind --> HandleSingleThreaded[处理single_threaded]
    HandleSingleThreaded --> |目标强制单线程且冲突| ReturnSingleThreadedError[返回单线程错误]
    HandleSingleThreaded --> DetermineErrorTracing[确定error_tracing]
    DetermineErrorTracing --> HandlePIC[处理位置无关代码pic]
    HandlePIC --> |目标需要PIC但禁用| ReturnPICError[返回PIC错误]
    HandlePIC --> |其他情况| DetermineRedZone[处理red_zone]
    DetermineRedZone --> |目标无红区但启用| ReturnRedZoneError[返回红区错误]
    DetermineRedZone --> DetermineOmitFramePointer[确定omit_frame_pointer]
    DetermineOmitFramePointer --> HandleSanitizeThread[处理sanitize_thread]
    HandleSanitizeThread --> DetermineUnwindTables[确定unwind_tables]
    DetermineUnwindTables --> HandleFuzz[处理fuzz]
    HandleFuzz --> DetermineCodeModel[确定code_model]
    DetermineCodeModel --> CheckSafeMode[确定is_safe_mode]
    CheckSafeMode --> HandleSanitizeC[处理sanitize_c]
    HandleSanitizeC --> HandleStackCheck[处理stack_check]
    HandleStackCheck --> |不支持但启用| ReturnStackCheckError[返回栈检查错误]
    HandleStackCheck --> DetermineStackProtector[确定stack_protector]
    DetermineStackProtector --> |条件不满足| HandleStackProtectorError[返回栈保护错误]
    DetermineStackProtector --> HandleStructuredCFG[处理structured_cfg]
    HandleStructuredCFG --> HandleNoBuiltin[处理no_builtin]
    HandleNoBuiltin --> GenerateLLVMFeatures[生成llvm_cpu_features]
    GenerateLLVMFeatures --> CreateModule[创建Module实例]
    CreateModule --> GenerateBuiltinMod[生成内置模块？]
    GenerateBuiltinMod --> |需要生成| CreateBuiltinFile[创建内置文件]
    CreateBuiltinFile --> AddDeps[添加依赖关系]
    AddDeps --> ReturnModule[返回Module指针]
    
    %% 错误路径统一指向终止节点
    ReturnValgrindError --> ErrorExit[错误退出]
    ReturnSingleThreadedError --> ErrorExit
    ReturnPICError --> ErrorExit
    ReturnRedZoneError --> ErrorExit
    ReturnStackCheckError --> ErrorExit
    HandleStackProtectorError --> ErrorExit
    
    style Start fill:#90EE90
    style ErrorExit fill:#FFB6C1
    style ReturnModule fill:#87CEFA
