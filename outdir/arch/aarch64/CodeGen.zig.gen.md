flowchart TD
    Start[开始gen函数] --> CheckCC{cc != .naked?}
    CheckCC -->|是| NonNakedPath
    CheckCC -->|否| NakedPath

    NonNakedPath --> SaveFP_LR[保存fp/lr: stp指令]
    SaveFP_LR --> BackpatchSaveReg[后补丁保存寄存器占位]
    BackpatchSaveReg --> MoveFP[移动fp到sp]
    MoveFP --> BackpatchReloc[后补丁栈分配占位]
    BackpatchReloc --> CheckRetMCV{ret_mcv是栈偏移?}
    
    CheckRetMCV -->|是| SaveRetPtr[分配栈空间保存返回值地址]
    CheckRetMCV -->|否| ProcessArgs
    
    SaveRetPtr --> ProcessArgs
    ProcessArgs --> LoopArgs[循环处理参数]
    LoopArgs -->|每个参数| CopyArgToStack[将寄存器参数复制到栈]
    CopyArgToStack --> LoopArgs
    LoopArgs --> EndLoopArgs[循环结束]
    
    EndLoopArgs --> DebugPrologue[添加调试标记dbg_prologue_end]
    DebugPrologue --> GenBody[生成主体代码genBody]
    
    GenBody --> CalcSavedRegs[计算需保存的寄存器]
    CalcSavedRegs --> BackpatchPush[回填push指令]
    BackpatchPush --> AdjustStack[计算总栈大小并调整]
    
    AdjustStack --> CheckStackSize{栈大小<=4096?}
    CheckStackSize -->|是| UpdateStack[生成sub指令]
    CheckStackSize -->|否| Panic[触发panic]
    
    UpdateStack --> DebugEpilogue[添加调试标记dbg_epilogue_begin]
    DebugEpilogue --> ExitJumps[处理exitlude跳转]
    ExitJumps --> RestoreStack[恢复栈指针add指令]
    RestoreStack --> PopRegs[恢复保存的寄存器]
    PopRegs --> LoadFP_LR[恢复fp/lr: ldp指令]
    LoadFP_LR --> Ret[返回指令ret]
    
    NakedPath --> NakedDebugPrologue[添加调试标记dbg_prologue_end]
    NakedDebugPrologue --> NakedGenBody[生成主体代码genBody]
    NakedGenBody --> NakedDebugEpilogue[添加调试标记dbg_epilogue_begin]
    
    Ret --> AddDebugLine
    NakedDebugEpilogue --> AddDebugLine
    AddDebugLine[添加调试行信息] --> End[函数结束]
