graph TD
    Start[开始] --> CheckUnused{指令是否未使用?}
    CheckUnused --> |是| Unreach[返回.unreach]
    CheckUnused --> |否| ParseOps[解析操作类型op和排序order]
    ParseOps --> CheckValSize{val_size是否为2的幂次方?}
    CheckValSize --> |否| FailNonPow2[报错'TODO: non-pow2']
    CheckValSize --> |是| CheckType{检查val_ty类型}
    CheckType --> |enum/int| SelectMethod[根据val_size选择方法]
    CheckType --> |bool/float/pointer| FailType[报错'TODO: 该类型']
    SelectMethod --> |1/2字节| MethodLoop[选择loop方法]
    SelectMethod --> |4/8字节| MethodAmo[选择amo方法]
    
    MethodAmo --> SetAqRl[设置aq/rl标志]
    SetAqRl --> GenMnemonic[生成对应mnemonic指令]
    GenMnemonic --> EmitAmo[发射amo指令]
    EmitAmo --> FinishAmo[返回result_mcv]
    
    MethodLoop --> EmitLrw[发射lrw指令]
    EmitLrw --> AllocAfterReg[分配after_reg]
    AllocAfterReg --> GenBinOp[生成算术运算]
    GenBinOp --> EmitScw[发射scw指令]
    EmitScw --> CheckBNE[检查bne条件]
    CheckBNE --> |失败| JumpBack[跳转回lrw]
    CheckBNE --> |成功| FinishLoop[返回result_mcv]
    
    FinishAmo --> Finish[结束指令]
    FinishLoop --> Finish
    Unreach --> Finish
    FailNonPow2 --> Finish
    FailType --> Finish
    
    classDef decision fill:#f9f,stroke:#333,stroke-width:2px;
    classDef process fill:#bbf,stroke:#333,stroke-width:2px;
    classDef error fill:#fbb,stroke:#933;
    class CheckUnused,CheckValSize,CheckType,CheckBNE decision;
    class ParseOps,SetAqRl,GenMnemonic,EmitAmo,EmitLrw,AllocAfterReg,GenBinOp,EmitScw process;
    class FailNonPow2,FailType error;
