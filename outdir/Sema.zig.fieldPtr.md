flowchart TD
    Start[开始] --> CheckPointerType[检查对象指针类型是否为指针类型]
    CheckPointerType -- 是 --> GetObjectType[获取对象类型]
    CheckPointerType -- 否 --> ReturnError1[返回错误: 期望指针类型]

    GetObjectType --> CheckInnerType[检查内部类型]
    CheckInnerType --> SwitchInnerType{内部类型判断}
    
    SwitchInnerType -- array --> HandleArray[处理数组类型]
    HandleArray --> CheckArrayFieldName{字段名是 len 或 ptr?}
    CheckArrayFieldName -- len --> ReturnArrayLen[返回数组长度的指针]
    CheckArrayFieldName -- ptr --> AdjustPtrType[调整指针类型并返回]
    CheckArrayFieldName -- 其他 --> ReturnError2[返回错误: 无效字段名]

    SwitchInnerType -- pointer(slice) --> HandleSlice[处理切片类型]
    HandleSlice --> CheckSliceFieldName{字段名是 ptr 或 len?}
    CheckSliceFieldName -- ptr --> GetSlicePtr[生成切片指针的指针]
    CheckSliceFieldName -- len --> GetSliceLen[生成切片长度的指针]
    CheckSliceFieldName -- 其他 --> ReturnError3[返回错误: 无效字段名]

    SwitchInnerType -- type --> HandleType[处理类型字段]
    HandleType --> CheckTypeTag{类型的具体类别}
    CheckTypeTag -- error_set --> HandleErrorSet[处理错误集字段]
    CheckTypeTag -- union --> HandleUnionType[处理联合类型字段]
    CheckTypeTag -- enum --> HandleEnumType[处理枚举类型字段]
    CheckTypeTag -- struct/opaque --> HandleStructType[处理结构体/不透明类型字段]
    CheckTypeTag -- 其他 --> ReturnError4[返回错误: 类型无成员]

    HandleErrorSet --> VerifyErrorName[验证错误名是否存在]
    VerifyErrorName -- 存在 --> ReturnErrorVal[返回错误值指针]
    VerifyErrorName -- 不存在 --> ReturnError5[返回错误: 无此错误名]

    HandleUnionType --> CheckUnionTag[检查联合标签类型]
    CheckUnionTag -- 有枚举标签 --> FindEnumField[查找枚举字段索引]
    FindEnumField -- 存在 --> ReturnEnumVal[返回枚举值指针]
    FindEnumField -- 不存在 --> ReturnError6[返回错误: 无效字段名]

    HandleEnumType --> FindEnumFieldIndex[查找枚举字段索引]
    FindEnumFieldIndex -- 存在 --> ReturnEnumVal2[返回枚举值指针]
    FindEnumFieldIndex -- 不存在 --> ReturnError7[返回错误: 无效字段名]

    HandleStructType --> LookupStructField[查找结构体字段]
    LookupStructField -- 存在 --> ReturnStructField[返回结构体字段指针]
    LookupStructField -- 不存在 --> ReturnError8[返回错误: 无效字段名]

    SwitchInnerType -- struct --> HandleStruct[处理结构体类型]
    HandleStruct --> CallStructFieldPtr[调用 structFieldPtr]
    CallStructFieldPtr --> CheckAllocPtr[检查分配指针有效性]
    CheckAllocPtr --> ReturnStructPtr[返回结构体字段指针]

    SwitchInnerType -- union --> HandleUnion[处理联合体类型]
    HandleUnion --> CallUnionFieldPtr[调用 unionFieldPtr]
    CallUnionFieldPtr --> CheckAllocPtr2[检查分配指针有效性]
    CheckAllocPtr2 --> ReturnUnionPtr[返回联合体字段指针]

    SwitchInnerType -- 其他 --> ReturnError9[返回错误: 无效字段访问]
