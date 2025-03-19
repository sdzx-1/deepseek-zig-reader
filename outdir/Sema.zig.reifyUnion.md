graph TD
    Start[开始] --> InitVar[初始化变量: pt, zcu, gpa, ip]
    InitVar --> GetFieldsLen[获取字段数量 fields_len]
    GetFieldsLen --> InitHasher[初始化哈希器 hasher]
    InitHasher --> HashLayout[哈希布局 layout]
    HashLayout --> HashTagType[哈希标签类型 opt_tag_type_val]
    HashTagType --> HashFieldsLen[哈希字段数量 fields_len]

    HashFieldsLen --> LoopFieldsStart[循环遍历字段]
    LoopFieldsStart --> LoopCondition{遍历所有字段?}
    LoopCondition -->|是| GetFieldInfo[获取字段信息 field_info]
    GetFieldInfo --> HashFieldName[哈希字段名称]
    HashFieldName --> HashFieldType[哈希字段类型]
    HashFieldType --> HashFieldAlign[哈希对齐值]
    HashFieldAlign --> CheckAlign[检查对齐值是否为0]
    CheckAlign -->|非零| SetAnyAligns[标记 any_aligns = true]
    CheckAlign --> LoopNext[继续下一个字段]
    SetAnyAligns --> LoopNext
    LoopNext --> LoopCondition

    LoopCondition -->|否| TrackZir[追踪ZIR指令 tracked_inst]
    TrackZir --> GetUnionType[尝试获取联合类型]
    GetUnionType --> TypeExists{类型是否存在?}
    TypeExists -->|是| ReturnExisting[返回现有类型]
    TypeExists -->|否| CreateWipType[创建WIP类型 wip_ty]
    CreateWipType --> SetTypeName[设置类型名称 type_name]
    SetTypeName --> AllocFields[分配字段类型和对齐数组]
    AllocFields --> CheckExplicitTag{是否存在显式标签类型?}

    CheckExplicitTag -->|是| ValidateTag[验证标签类型是枚举]
    ValidateTag --> InitSeenTags[初始化已见标签集合]
    InitSeenTags --> LoopTagFields[遍历字段检查标签]
    LoopTagFields --> CheckFieldInEnum{字段是否在枚举中?}
    CheckFieldInEnum -->|否| ErrorMissing[报错缺失字段]
    CheckFieldInEnum -->|是| CheckDuplicate[检查是否重复]
    CheckDuplicate -->|是| ErrorDup[报错重复字段]
    CheckDuplicate -->|否| UpdateSeenTags[更新已见标签]
    UpdateSeenTags --> ValidateAlign[验证对齐值有效性]
    ValidateAlign --> LoopTagNext[继续下一个字段]
    LoopTagNext --> LoopTagFields
    LoopTagFields结束 --> CheckMissingTags[检查枚举是否有未覆盖字段]
    CheckMissingTags -->|是| ErrorUncovered[报错未覆盖字段]

    CheckExplicitTag -->|否| GenerateTag[生成隐式标签类型]
    GenerateTag --> CheckUniqueNames[检查字段名称唯一性]
    CheckUniqueNames -->|重复| ErrorDupName[报错重复名称]
    CheckUniqueNames -->|唯一| CreateTagType[创建新枚举类型]

    ValidateTag流程结束 --> ValidateFieldTypes[验证字段类型]
    GenerateTag流程结束 --> ValidateFieldTypes
    ValidateFieldTypes --> CheckOpaque[检查是否包含opaque类型]
    CheckOpaque -->|是| ErrorOpaque[报错opaque类型]
    CheckOpaque -->|否| CheckLayout{布局检查}
    CheckLayout -->|extern| ValidateExtern[验证extern类型]
    ValidateExtern -->|无效| ErrorExtern[报错extern类型]
    CheckLayout -->|packed| ValidatePacked[验证packed类型]
    ValidatePacked -->|无效| ErrorPacked[报错packed类型]

    ValidateFieldTypes完成 --> FinalizeType[设置字段类型/对齐/标签类型]
    FinalizeType --> CreateNamespace[创建新命名空间]
    CreateNamespace --> QueueJobs[加入编译队列]
    QueueJobs --> AddDependency[添加类型依赖]
    AddDependency --> ReturnResult[返回结果 Air.Inst.Ref]

    ErrorMissing --> ErrorExit[错误处理]
    ErrorDup --> ErrorExit
    ErrorUncovered --> ErrorExit
    ErrorDupName --> ErrorExit
    ErrorOpaque --> ErrorExit
    ErrorExtern --> ErrorExit
    ErrorPacked --> ErrorExit
    ErrorExit --> CancelWip[取消WIP类型]
    CancelWip --> End[结束]

    ReturnExisting --> End
    ReturnResult --> End

    style Start fill:#9f9,stroke:#333
    style End fill:#f99,stroke:#333
    style ErrorExit fill:#f00,stroke:#333,color:#fff
