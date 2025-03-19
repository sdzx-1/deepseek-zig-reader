graph TD
    Start[开始] --> SetBranchQuota[设置@setEvalBranchQuota(2500)]
    SetBranchQuota --> SwitchOps1{switch inst.ops}
    
    SwitchOps1 --> |none| FixesNone[获取fixes: inst.data.none.fixes]
    SwitchOps1 --> |inst| FixesInst[获取fixes: inst.data.inst.fixes]
    SwitchOps1 --> |i_s/i_u| FixesI[获取fixes: inst.data.i.fixes]
    SwitchOps1 --> |...其他分支...| FixesOthers[...]
    SwitchOps1 --> |未实现类型| ReturnError1[返回错误: TODO lower]
    
    FixesNone --> ProcessFixes[处理fixes前缀]
    FixesInst --> ProcessFixes
    FixesI --> ProcessFixes
    FixesOthers --> ProcessFixes
    
    ProcessFixes --> |inline switch fixes| CheckTagName[检查标签名中的空格]
    CheckTagName --> |有空格| ExtractPrefix[提取前缀字段]
    CheckTagName --> |无空格| PrefixNone[前缀为.none]
    
    ExtractPrefix --> GenerateMnemonic[生成助记符]
    PrefixNone --> GenerateMnemonic
    
    GenerateMnemonic --> |拼接parts| ValidateMnemonic[验证mnemonic有效性]
    ValidateMnemonic --> |无效| ReturnError2[返回错误: 不支持的助记符]
    
    ValidateMnemonic --> SwitchOps2{switch inst.ops生成操作数}
    
    SwitchOps2 --> |none| OperandsNone[操作数列表: 空]
    SwitchOps2 --> |inst| OperandsInst[操作数列表: 立即数重定位]
    SwitchOps2 --> |i_s/i_u| OperandsI[操作数列表: 立即数]
    SwitchOps2 --> |...其他分支...| OperandsOthers[...]
    SwitchOps2 --> |未实现类型| ReturnError3[返回错误: 未实现的操作数组合]
    
    OperandsNone --> Emit[调用lower.emit]
    OperandsInst --> Emit
    OperandsI --> Emit
    OperandsOthers --> Emit
    
    Emit --> Success[成功返回]
    ReturnError1 --> Error[错误返回]
    ReturnError2 --> Error
    ReturnError3 --> Error
