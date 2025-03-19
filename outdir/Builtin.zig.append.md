flowchart TD
    Start([开始]) --> Initialize[初始化变量: target, zig_backend]
    Initialize --> SetEval[设置@setEvalBranchQuota(4000)]
    SetEval --> WriteBaseConfig[写入基本配置: 版本/后端/输出模式等]
    WriteBaseConfig --> LoopFeatures[循环处理CPU特性]
    
    LoopFeatures --> ForEachFeature[遍历所有CPU特性]
    ForEachFeature --> CheckEnabled{特性已启用?}
    CheckEnabled -- 是 --> WriteFeature[写入特性名称到buffer]
    CheckEnabled -- 否 --> SkipFeature[跳过]
    WriteFeature --> NextFeature[下一个特性]
    SkipFeature --> NextFeature
    NextFeature --> LoopEnd{所有特性处理完成?}
    LoopEnd -- 否 --> ForEachFeature
    LoopEnd -- 是 --> ProcessOS[处理操作系统版本范围]
    
    ProcessOS --> SwitchOS{OS类型判断}
    SwitchOS -- .none --> WriteNone[写入.none配置]
    SwitchOS -- .semver --> WriteSemver[写入semver范围]
    SwitchOS -- .linux --> WriteLinux[写入Linux配置]
    SwitchOS -- .hurd --> WriteHurd[写入Hurd配置]
    SwitchOS -- .windows --> WriteWindows[写入Windows版本]
    
    WriteNone --> AfterOS
    WriteSemver --> AfterOS
    WriteLinux --> AfterOS
    WriteHurd --> AfterOS
    WriteWindows --> AfterOS
    
    AfterOS --> WriteDynamicLinker[处理动态链接器配置]
    WriteDynamicLinker --> HasDL{存在动态链接器?}
    HasDL -- 是 --> WriteDLPath[写入动态链接器路径]
    HasDL -- 否 --> WriteDLNone[标记为.none]
    
    WriteDLPath --> WriteCompileOptions[写入编译选项: link_libc/mode等]
    WriteDLNone --> WriteCompileOptions
    
    WriteCompileOptions --> CheckWASI{目标OS是WASI?}
    CheckWASI -- 是 --> WriteWASI[写入WASI执行模型]
    CheckWASI -- 否 --> CheckTest{是否为测试模式?}
    
    WriteWASI --> CheckTest
    CheckTest -- 是 --> WriteTestFunc[写入测试函数配置]
    CheckTest -- 否 --> End[结束]
    
    WriteTestFunc --> End
    End --> Stop([结束])
    
    style Start fill:#90EE90,stroke:#006400
    style Stop fill:#FFA07A,stroke:#8B0000
    style CheckEnabled,SwitchOS,HasDL,CheckWASI,CheckTest fill:#FFD700,stroke:#FF8C00
    style WriteBaseConfig,LoopFeatures,ProcessOS,WriteDynamicLinker,WriteCompileOptions fill:#87CEEB,stroke:#00008B
