graph TD
    Start[开始] --> Init[初始化变量和分配器]
    Init --> CheckCC{检查调用约定 cc}
    
    CheckCC --> |naked| Naked[处理naked调用约定]
    Naked --> SetUnreach[设置return_value为unreach]
    Naked --> SetStackAlign[设置stack_align为4或8]
    
    CheckCC --> |x86_64_sysv/x86_64_win| SysvWin[处理SystemV/Windows调用约定]
    SysvWin --> InitRegCounters[初始化寄存器计数器]
    SysvWin --> HandleReturnValue[处理返回值]
    HandleReturnValue --> ClassifyRet[分类返回类型到寄存器/内存]
    ClassifyRet --> |integer| RetIntReg[使用整数寄存器]
    ClassifyRet --> |sse/float| RetSSEReg[使用SSE寄存器]
    ClassifyRet --> |memory| RetMem[使用内存间接返回]
    
    SysvWin --> HandleParams[处理参数]
    HandleParams --> ForEachParam[遍历每个参数]
    ForEachParam --> ClassifyParam[分类参数类型]
    ClassifyParam --> |integer| ParamIntReg[使用整数寄存器]
    ClassifyParam --> |sse/float| ParamSSEReg[使用SSE寄存器]
    ClassifyParam --> |stack| ParamStack[分配到栈空间]
    
    CheckCC --> |auto| Auto[处理Zig自定义调用约定]
    Auto --> HandleAutoReturn[处理自动返回]
    Auto --> HandleAutoParams[处理自动参数]
    HandleAutoParams --> |registers| UseRegisters[优先使用寄存器]
    HandleAutoParams --> |stack| UseStack[次优分配到栈]
    
    CheckCC --> |其他| Error[返回未实现错误]
    
    SysvWin --> Finalize[最终计算栈大小和对齐]
    Auto --> Finalize
    Naked --> Finalize
    
    Finalize --> Return[返回CallMCValues结构]
    Error --> Return
