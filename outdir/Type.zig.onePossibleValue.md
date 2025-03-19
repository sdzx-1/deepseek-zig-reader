graph TD
    Start[开始: onePossibleValue] --> Loop[进入循环]
    Loop --> SwitchType{根据类型进行判断}
    
    SwitchType -->|empty_tuple_type| ReturnEmptyTuple[返回空元组]
    SwitchType -->|int_type| CheckBits{检查bits是否为0}
    CheckBits -->|是| ReturnInt0[返回整数值0]
    CheckBits -->|否| ReturnNull1[返回null]
    
    SwitchType -->|array_type/vector_type| CheckSeqLength{检查序列长度+哨兵}
    CheckSeqLength -->|长度为0| ReturnEmptyAggregate[返回空聚合值]
    CheckSeqLength -->|长度非0| CheckChildOPV[递归检查子类型]
    CheckChildOPV -->|有唯一值| ReturnRepeatedElem[返回重复元素聚合]
    CheckChildOPv -->|无唯一值| ReturnNull2[返回null]
    
    SwitchType -->|struct_type| CheckStructFields[遍历结构体字段]
    CheckStructFields -->|存在非comptime字段| CheckFieldOPV[递归检查字段类型]
    CheckFieldOPV -->|所有字段有唯一值| ReturnStructAggregate[返回结构体聚合]
    CheckFieldOPV -->|任一字段无唯一值| ReturnNull3[返回null]
    
    SwitchType -->|enum_type| CheckEnumTagMode{检查枚举tag模式}
    CheckEnumTagMode -->|nonexhaustive| CheckTagOPV[检查tag类型唯一值]
    CheckEnumTagMode -->|auto/explicit| CheckEnumCases[检查枚举项数量]
    CheckTagOPV -->|有唯一值| ReturnEnumTag[返回枚举标签]
    CheckEnumCases -->|0项| ReturnEmptyEnum[返回空枚举]
    CheckEnumCases -->|1项| ReturnSingleEnum[返回唯一枚举值]
    
    SwitchType -->|其他类型| ReturnDefaultNull[返回null]
    
    ReturnEmptyTuple --> Exit[结束]
    ReturnInt0 --> Exit
    ReturnNull1 --> Exit
    ReturnEmptyAggregate --> Exit
    ReturnRepeatedElem --> Exit
    ReturnNull2 --> Exit
    ReturnStructAggregate --> Exit
    ReturnNull3 --> Exit
    ReturnEnumTag --> Exit
    ReturnEmptyEnum --> Exit
    ReturnSingleEnum --> Exit
    ReturnDefaultNull --> Exit
    
    classDef decision fill:#f9f,stroke:#333;
    classDef process fill:#9f9,stroke:#333;
    classDef startend fill:#ff9,stroke:#333;
    
    class SwitchType,CheckBits,CheckSeqLength,CheckChildOPV,CheckStructFields,CheckFieldOPV,CheckEnumTagMode,CheckTagOPV,CheckEnumCases decision
    class ReturnEmptyTuple,ReturnInt0,ReturnNull1,ReturnEmptyAggregate,ReturnRepeatedElem,ReturnNull2,ReturnStructAggregate,ReturnNull3,ReturnEnumTag,ReturnEmptyEnum,ReturnSingleEnum,ReturnDefaultNull process
    class Start,Loop,Exit startend
