flowchart TD
    Start([Start]) --> CheckType{检查类型}
    
    CheckType -->|int_type| IntType[处理整数类型]
    IntType --> GetSign[获取符号字符]
    GetSign -->|signed| SignI[输出'i']
    GetSign -->|unsigned| SignU[输出'u']
    SignI --> PrintInt[输出位数]
    SignU --> PrintInt
    PrintInt --> ReturnInt[返回]

    CheckType -->|ptr_type| PtrType[处理指针类型]
    PtrType --> CheckSentinel{检查sentinel}
    CheckSentinel -->|存在| HandleSentinel[根据size输出格式]
    CheckSentinel -->|不存在| HandleBasePtr[输出基本指针格式]
    HandleSentinel --> CheckAllowZero
    HandleBasePtr --> CheckAllowZero
    CheckAllowZero -->|是| WriteAllowZero[输出'allowzero']
    CheckAllowZero -->|否| CheckAlignment
    WriteAllowZero --> CheckAlignment
    CheckAlignment -->|需要对齐信息| WriteAlign[输出对齐信息]
    CheckAlignment -->|不需要| CheckAddressSpace
    WriteAlign --> CheckAddressSpace
    CheckAddressSpace -->|非通用地址空间| WriteAddrSpace[输出地址空间]
    CheckAddressSpace -->|通用| CheckConstVolatile
    WriteAddrSpace --> CheckConstVolatile
    CheckConstVolatile -->|const| WriteConst[输出'const']
    CheckConstVolatile -->|volatile| WriteVolatile[输出'volatile']
    WriteConst --> RecurseChild
    WriteVolatile --> RecurseChild
    CheckConstVolatile -->|无| RecurseChild
    RecurseChild[递归处理子类型] --> ReturnPtr[返回]

    CheckType -->|array_type| ArrayType[处理数组类型]
    ArrayType --> CheckArraySentinel{检查sentinel}
    CheckArraySentinel -->|存在| WriteArrayWithSentinel[输出带sentinel的数组]
    CheckArraySentinel -->|不存在| WriteSimpleArray[输出普通数组]
    WriteArrayWithSentinel --> RecurseArrayChild
    WriteSimpleArray --> RecurseArrayChild
    RecurseArrayChild[递归处理子类型] --> ReturnArray[返回]

    CheckType -->|vector_type| VectorType[输出@Vector并递归子类型]
    VectorType --> ReturnVector[返回]

    CheckType -->|opt_type| OptType[输出'?'并递归子类型]
    OptType --> ReturnOpt[返回]

    CheckType -->|error_union_type| ErrorUnion[处理错误联合类型]
    ErrorUnion --> PrintErrorSet[输出错误集]
    PrintErrorSet --> WriteExcl[输出'!']
    WriteExcl --> CheckPayload{检查payload类型}
    CheckPayload -->|generic_poison| WriteAnytype[输出'anytype']
    CheckPayload -->|其他| RecursePayload[递归处理payload]
    WriteAnytype --> ReturnErrorUnion
    RecursePayload --> ReturnErrorUnion
    ReturnErrorUnion --> ReturnErrorUnionEnd[返回]

    CheckType -->|simple_type| SimpleType[处理简单类型]
    SimpleType --> OutputSimple[直接输出类型名]
    OutputSimple --> ReturnSimple[返回]

    CheckType -->|struct_type| StructType[输出结构体名称]
    StructType --> ReturnStruct[返回]

    CheckType -->|...其他类型分支...| OtherCases[其他类型处理]
    OtherCases --> ReturnOthers[返回]

    ReturnInt --> End([End])
    ReturnPtr --> End
    ReturnArray --> End
    ReturnVector --> End
    ReturnOpt --> End
    ReturnErrorUnionEnd --> End
    ReturnSimple --> End
    ReturnStruct --> End
    ReturnOthers --> End
