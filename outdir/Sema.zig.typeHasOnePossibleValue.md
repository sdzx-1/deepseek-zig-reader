graph TD
    Start[开始] --> CheckType{检查类型}
    CheckType -->|基本类型| HandleBasicType[处理基本类型]
    CheckType -->|复杂类型| HandleComplexType[处理复杂类型]
    CheckType -->|特殊类型| HandleSpecialType[处理特殊类型]
    
    HandleBasicType -->|如u0/i0等| ReturnZero[返回0值]
    HandleBasicType -->|如void| ReturnVoid[返回void值]
    HandleBasicType -->|其他数值类型| ReturnNull[返回null]
    
    HandleSpecialType -->|如noreturn| ReturnUnreachable[返回unreachable]
    HandleSpecialType -->|如null/undefined| ReturnNullUndef[返回null/undef]
    HandleSpecialType -->|如empty_tuple| ReturnEmptyTuple[返回空元组]
    
    HandleComplexType -->|数组/向量| CheckArrayVector[检查元素类型]
    HandleComplexType -->|结构体| CheckStructFields[递归检查字段]
    HandleComplexType -->|联合体| CheckUnionTagVal[检查标签和值]
    HandleComplexType -->|枚举| CheckEnumVariants[检查枚举变体]
    HandleComplexType -->|元组| CheckTupleValues[检查元组元素]
    
    CheckArrayVector -->|长度+哨兵=0| ReturnEmptyArray[返回空数组值]
    CheckArrayVector -->|元素类型唯一| ReturnRepeatedElem[返回重复元素值]
    CheckArrayVector -->|其他情况| ReturnNull
    
    CheckStructFields -->|无字段| ReturnEmptyStruct[返回空结构体值]
    CheckStructFields -->|所有字段类型唯一| BuildStructValue[构建结构体值]
    CheckStructFields -->|存在非唯一字段| ReturnNull
    
    CheckUnionTagVal -->|标签类型唯一且字段唯一| BuildUnionValue[构建联合体值]
    CheckUnionTagVal -->|其他情况| ReturnNull
    
    CheckEnumVariants -->|非穷尽枚举| CheckTagType[检查标签类型唯一性]
    CheckEnumVariants -->|自动/显式枚举| CheckTagRuntimeBits[检查标签运行时信息]
    CheckEnumVariants -->|多个变体| ReturnNull
    
    CheckTupleValues -->|所有元素已知| BuildTupleValue[构建元组值]
    CheckTupleValues -->|存在未知元素| ReturnNull
    
    ReturnZero --> End[结束]
    ReturnVoid --> End
    ReturnNull --> End
    ReturnUnreachable --> End
    ReturnNullUndef --> End
    ReturnEmptyTuple --> End
    ReturnEmptyArray --> End
    ReturnRepeatedElem --> End
    ReturnEmptyStruct --> End
    BuildStructValue --> End
    BuildUnionValue --> End
    BuildTupleValue --> End

    classDef logic fill:#f9f,stroke:#333;
    classDef data fill:#f96,stroke:#333;
    classDef result fill:#9f9,stroke:#333;
    
    class CheckType,HandleBasicType,HandleComplexType,HandleSpecialType logic
    class CheckArrayVector,CheckStructFields,CheckUnionTagVal,CheckEnumVariants,CheckTupleValues logic
    class ReturnZero,ReturnVoid,ReturnNull,ReturnUnreachable,ReturnNullUndef,ReturnEmptyTuple,ReturnEmptyArray,ReturnRepeatedElem,ReturnEmptyStruct,BuildStructValue,BuildUnionValue,BuildTupleValue result
