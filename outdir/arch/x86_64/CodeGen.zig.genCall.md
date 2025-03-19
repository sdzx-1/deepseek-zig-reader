flowchart TD
    Start[开始genCall] --> GetPT[获取pt, zcu, ip]
    GetPT --> DetermineFnType[确定函数类型fn_ty]
    DetermineFnType --> |info是.air| AirType[解析AIR指令类型]
    DetermineFnType --> |info是.lib| LibType[生成C调用约定的函数类型]
    
    AirType --> CheckCalleeType[检查callee类型是否为函数或指针]
    LibType --> CreateFuncType[创建lib的函数类型]
    
    CheckCalleeType --> GetFnType[获取函数类型]
    CreateFuncType --> GetFnType
    
    GetFnType --> AllocStack[分配栈空间]
    AllocStack --> InitVarArgs[初始化可变参数var_args]
    InitVarArgs --> AllocFrameIndices[分配帧索引frame_indices]
    
    AllocFrameIndices --> ResolveCallConv[解析调用约定call_info]
    ResolveCallConv --> AlignStackFrame[对齐栈帧]
    
    AlignStackFrame --> SpillRegisters[溢出eflags和调用者保存寄存器]
    SpillRegisters --> HandleStackArgs[处理栈参数]
    
    HandleStackArgs --> |间接返回值| HandleIndirectRet[分配帧索引并设置返回值地址]
    HandleStackArgs --> |寄存器参数| LockRegisters[锁定寄存器]
    
    LockRegisters --> SetRegisterArgs[设置寄存器参数]
    SetRegisterArgs --> |可变参数| SetAL[设置AL寄存器]
    
    SetAL --> GenerateCall[生成调用指令]
    GenerateCall --> |ELF| ELFCall[生成ELF符号调用]
    GenerateCall --> |COFF| COFFCall[生成COFF原子调用]
    GenerateCall --> |Mach-O| MachOCall[生成Mach-O符号调用]
    GenerateCall --> |Plan9| Plan9Call[生成Plan9调用]
    GenerateCall --> |外部符号| ExternCall[生成外部符号调用]
    GenerateCall --> |函数指针| PtrCall[通过寄存器调用函数指针]
    
    GenerateCall --> ReturnValue[处理返回值]
    ReturnValue --> Cleanup[清理资源]
    Cleanup --> UnlockRegisters[解锁寄存器]
    UnlockRegisters --> End[返回call_info.return_value.short]
    
    classDef condition fill:#f9f,stroke:#333
    classDef process fill:#bbf,stroke:#333
    classDef terminal fill:#ff9,stroke:#333
    
    class DetermineFnType,CheckCalleeType,CreateFuncType,HandleIndirectRet,SetAL condition
    class GetPT,GetFnType,AllocStack,InitVarArgs,AllocFrameIndices,ResolveCallConv,AlignStackFrame,SpillRegisters,HandleStackArgs,LockRegisters,SetRegisterArgs,GenerateCall,ReturnValue,Cleanup,UnlockRegisters process
    class Start,End terminal
