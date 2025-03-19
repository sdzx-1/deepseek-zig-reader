graph TD
    Start[开始] --> InitVars[初始化变量 pt, zcu, gpa, ip]
    InitVars --> CalcFieldsLen[计算 fields_len = fields_val.arrayLen]
    CalcFieldsLen --> InitHasher[初始化哈希器 hasher]
    InitHasher --> HashTypeInfo[哈希 tag_ty, is_exhaustive, fields_len]
    
    HashTypeInfo --> LoopFields1[遍历字段 (第一次循环)]
    LoopFields1 --> |每个字段| GetFieldInfo[获取 field_info]
    GetFieldInfo --> GetFieldNameValue[获取 field_name_val]
    GetFieldNameValue --> GetFieldValue[获取 field_value_val]
    GetFieldValue --> HashField[哈希字段名和值]
    HashField --> LoopFields1
    
    LoopFields1 --> AfterHash[哈希完成]
    AfterHash --> TrackInst[追踪指令 tracked_inst]
    TrackInst --> GetEnumType[尝试获取/创建枚举类型 wip_ty]
    
    GetEnumType --> |存在现有类型| ExistingType[返回现有类型引用]
    GetEnumType --> |创建新类型| SetupWipType[设置新类型]
    
    SetupWipType --> CheckTagType[检查 tag_ty 是否为整数类型]
    CheckTagType --> |否| FailTagType[报错并返回]
    CheckTagType --> |是| SetTypeName[设置类型名称]
    
    SetTypeName --> CreateNamespace[创建新命名空间]
    CreateNamespace --> PrepareType[准备类型属性]
    PrepareType --> LoopFields2[遍历字段 (第二次循环)]
    
    LoopFields2 --> |每个字段| GetFieldInfo2[获取 field_info]
    GetFieldInfo2 --> CheckValueFit[检查值是否适合 tag_ty]
    CheckValueFit --> |否| FailValueFit[报错并返回]
    CheckValueFit --> |是| CoerceValue[类型转换 coerced_field_val]
    CoerceValue --> CheckConflict[检查字段名/值冲突]
    CheckConflict --> |冲突| FailConflict[生成错误信息]
    CheckConflict --> |无冲突| AddField[添加字段到类型]
    AddField --> LoopFields2
    
    LoopFields2 --> AfterFields[字段处理完成]
    AfterFields --> CheckExhaustive[检查非穷举枚举有效性]
    CheckExhaustive --> |无效| FailExhaustive[报错并返回]
    CheckExhaustive --> |有效| CodegenCheck[代码生成检查]
    
    CodegenCheck --> |需要生成代码| QueueCodegen[加入代码生成队列]
    CodegenCheck --> Finalize[完成类型创建]
    Finalize --> Return[返回类型引用]
    
    classDef error fill:#fdd,stroke:#f00;
    class FailTagType,FailValueFit,FailConflict,FailExhaustive error;
