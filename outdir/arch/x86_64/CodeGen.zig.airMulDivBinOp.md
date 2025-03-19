graph TD
    Start[开始 airMulDivBinOp] --> CheckType{检查 dst_ty 类型}
    CheckType --> |浮点或向量| GenBinOp[生成二元操作 genBinOp]
    CheckType --> |其他类型| GetDstInfo[获取目标类型信息]
    GetDstInfo --> CheckAbiSize{检查 dst_abi_size 是否为16且 src_abi_size 是否为16}
    CheckAbiSize --> |是| CheckTag{检查操作类型 tag}
    CheckAbiSize --> |否| SpillAndGen[溢出寄存器并生成乘除操作 genMulDivBinOp]
    
    CheckTag --> |mul, mul_wrap| GenCall[生成库函数调用 genCall]
    CheckTag --> |div_trunc, div_floor, div_exact, rem, mod| HandleDivMod[处理除法/取模]
    
    HandleDivMod --> SignedCheck{是否是有符号除法且 tag 为 div_floor?}
    SignedCheck --> |是| SetupFrame[设置帧和状态保存]
    SetupFrame --> RestoreState[恢复状态并修补重定位]
    RestoreState --> GenDivCall[生成除法库调用]
    GenDivCall --> AdjustResult[调整结果并释放资源]
    
    SignedCheck --> |否| GenModCall[生成取模库调用]
    GenModCall --> ModAdjust[调整取模结果]
    
    HandleDivMod --> |mod 操作| RegLock[锁定寄存器并复制值]
    RegLock --> MathOp[执行算术操作]
    MathOp --> Cmovcc[条件移动结果]
    
    GenCall --> MergeResult[合并结果]
    AdjustResult --> MergeResult
    ModAdjust --> MergeResult
    Cmovcc --> MergeResult
    SpillAndGen --> MergeResult
    
    MergeResult --> FinishAir[完成并返回 finishAir]
    GenBinOp --> FinishAir
