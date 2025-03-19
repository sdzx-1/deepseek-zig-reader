graph TD
    Start([开始]) --> CheckConstPtr{指针是常量?}
    CheckConstPtr -- 是 --> Fail[报错: 无法赋值给常量指针]
    CheckConstPtr -- 否 --> GetElemType[获取指针元素类型 elem_ty]
    
    GetElemType --> CheckTupleAndArray{操作数是元组且元素类型是数组?}
    CheckTupleAndArray -- 是 --> TupleLoop[循环处理元组字段]
    TupleLoop --> GetField[获取元组字段 elem]
    GetField --> GetElemIndex[生成字段索引 elem_index]
    GetElemIndex --> GetElemPtr[生成元素指针 elem_ptr]
    GetElemPtr --> RecursiveStore[递归调用 storePtr2]
    RecursiveStore --> LoopCondition{字段循环完成?}
    LoopCondition -- 否 --> TupleLoop
    LoopCondition -- 是 --> End
    
    CheckTupleAndArray -- 否 --> CheckBitcastVector{存在位转换的向量指针?}
    CheckBitcastVector -- 是 --> CoerceVector[将操作数强制转换为向量类型]
    CoerceVector --> StoreVector[存储到向量指针]
    StoreVector --> End
    
    CheckBitcastVector -- 否 --> CoerceOperand[强制转换操作数到 elem_ty]
    CoerceOperand --> CheckComptimeValue{操作数是编译时已知值?}
    CheckComptimeValue -- 是 --> ComptimeStore[执行编译时存储]
    ComptimeStore --> CheckPointeeType{检查元素类型是否允许运行时存储}
    
    CheckComptimeValue -- 否 --> CheckRuntimeBlock[确保处于运行时块]
    CheckRuntimeBlock --> CheckVectorIndex{指针类型有运行时向量索引?}
    CheckVectorIndex -- 是 --> GenerateVectorStore[生成向量存储指令]
    CheckVectorIndex -- 否 --> GenerateNormalStore[生成普通存储指令]
    
    GenerateVectorStore --> End
    GenerateNormalStore --> CheckComptimeKnown[检查存储是否编译时已知]
    CheckComptimeKnown --> End
    
    CheckPointeeType -- 不允许 --> FailComptimeOnly[报错: 运行时不能存储编译时类型]
    CheckPointeeType -- 允许 --> End
    
    End([结束])
    
    Fail --> End
    FailComptimeOnly --> End

    style Start fill:#90EE90,stroke:#006400
    style End fill:#FFA07A,stroke:#8B0000
    style Fail fill:#FFB6C1,stroke:#FF0000
    style FailComptimeOnly fill:#FFB6C1,stroke:#FF0000
