flowchart TD
    Start([开始]) --> CheckTagType{联合类型有标签类型?}
    CheckTagType --> |否| 生成错误1[生成错误: 无法将枚举转换为无标签联合]
    CheckTagType --> |是| Coerce[强制转换实例到标签类型]
    Coerce --> ResolveValue{解析到确定值?}
    ResolveValue --> |是| FindFieldIndex{查找联合字段索引}
    FindFieldIndex --> |失败| 生成错误2[生成错误: 无匹配标签值]
    FindFieldIndex --> |成功| CheckNoreturn{字段类型是noreturn?}
    CheckNoreturn --> |是| 生成错误3[生成错误: 无法初始化noreturn字段]
    CheckNoreturn --> |否| CheckOnePossibleValue{字段类型有唯一可能值?}
    CheckOnePossibleValue --> |否| 生成错误4[生成错误: 必须初始化具体字段]
    CheckOnePossibleValue --> |是| ReturnResult[返回联合值操作数]
    
    ResolveValue --> |否| RequireRuntime[需要运行时块]
    RequireRuntime --> CheckNonexhaustive{标签类型是非穷举枚举?}
    CheckNonexhaustive --> |是| 生成错误5[生成错误: 运行时转换非穷举枚举]
    CheckNonexhaustive --> |否| CheckAllZeroBits{联合字段全零位?}
    CheckAllZeroBits --> |是| BitCast[生成位转换指令]
    
    CheckAllZeroBits --> |否| CheckNoreturnFields[检查联合noreturn字段]
    CheckNoreturnFields --> |存在| 生成错误6[生成错误: 包含noreturn字段]
    CheckNoreturnFields --> |不存在| 生成错误7[生成错误: 存在非空字段]
    
    生成错误1 --> End
    生成错误2 --> End
    生成错误3 --> End
    生成错误4 --> End
    生成错误5 --> End
    生成错误6 --> End
    生成错误7 --> End
    ReturnResult --> End([结束])
    BitCast --> End
