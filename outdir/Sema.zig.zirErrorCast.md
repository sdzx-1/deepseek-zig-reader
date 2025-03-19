graph TD
    Start[开始] --> ResolveTypes[解析目标类型和操作数类型]
    ResolveTypes --> CheckDestTag[检查dest_tag是否为error_set或error_union]
    CheckDestTag -->|否| FailDestType[返回错误: 目标类型无效]
    CheckDestTag -->|是| CheckOperandTag[检查operand_tag是否为error_set或error_union]
    CheckOperandTag -->|否| FailOperandType[返回错误: 操作数类型无效]
    CheckOperandTag -->|是| CheckDestOperandCombo[检查dest_tag和operand_tag组合]
    CheckDestOperandCombo -->|dest=error_set且operand=error_union| FailCastToSet[返回错误: 不能将error_union转为error_set]
    CheckDestOperandCombo -->|dest=error_union且operand=error_union| CheckPayloadMatch[检查负载类型是否匹配]
    CheckPayloadMatch -->|不匹配| FailPayloadMismatch[返回错误: 负载类型不匹配]
    CheckPayloadMatch -->|匹配| ExtractErrorTypes[提取dest_err_ty和operand_err_ty]
    CheckDestOperandCombo -->|其他有效组合| ExtractErrorTypes
    ExtractErrorTypes --> CheckDisjoint[检查错误集是否不相交]
    CheckDisjoint -->|是且非error_union到error_union| FailDisjoint[返回错误: 错误集无交集]
    CheckDisjoint -->|否或允许组合| ResolveOperandValue[解析操作数值]
    ResolveOperandValue -->|值已定义| HandleDefinedValue[处理已定义值]
    HandleDefinedValue -->|验证错误名有效性| CheckErrorName[检查错误名是否在目标错误集中]
    CheckErrorName -->|无效| FailErrorName[返回错误: 错误名不在目标集中]
    CheckErrorName -->|有效| GenerateResult[生成结果指令]
    ResolveOperandValue -->|值未定义| AddSafetyChecks[添加运行时安全检查]
    AddSafetyChecks -->|根据类型生成检查逻辑| GenerateCast[生成类型转换指令]
    GenerateResult --> End[结束]
    GenerateCast --> End
    FailDestType --> End
    FailOperandType --> End
    FailCastToSet --> End
    FailPayloadMismatch --> End
    FailDisjoint --> End
    FailErrorName --> End
