flowchart TD
    Start[开始] --> GetTypeInfo[获取类型信息: result_ty, len, elements]
    GetTypeInfo --> CheckType{检查result_ty类型}
    
    CheckType --> |数组| HandleArray[处理数组]
    CheckType --> |结构体| CheckStructLayout{检查结构体布局}
    CheckType --> |向量| ReturnError[返回TODO错误]
    
    HandleArray --> AllocArrayStack[分配栈空间: result = allocStack]
    AllocArrayStack --> CheckByRef{元素类型是否为引用?}
    
    CheckByRef --> |是| RefArrayLoop[创建offset指针并循环存储元素]
    RefArrayLoop --> ForEachElementRef[循环处理每个元素]
    ForEachElementRef --> StoreElementRef[store offset指针]
    StoreElementRef --> ModifyOffset{是否最后一个元素且无哨兵?}
    ModifyOffset --> |是| AdjustOffset[调整offset指针]
    ModifyOffset --> |否| NextElementRef[继续循环]
    
    CheckByRef --> |否| ValueArrayLoop[直接计算偏移循环存储元素]
    ValueArrayLoop --> ForEachElementVal[循环处理每个元素]
    ForEachElementVal --> StoreElementVal[store result+offset]
    StoreElementVal --> IncreaseOffset[增加offset值]
    IncreaseOffset --> NextElementVal[继续循环]
    
    CheckStructLayout --> |packed| HandlePackedStruct[处理打包结构体]
    CheckStructLayout --> |其他| HandleNormalStruct[处理普通结构体]
    
    HandlePackedStruct --> AllocBackingInt[分配背景整数存储空间]
    AllocBackingInt --> ZeroInit[初始化为0]
    ZeroInit --> BitLoop[循环处理字段位操作]
    BitLoop --> ForEachPackedElement[处理每个字段]
    ForEachPackedElement --> LoadResult[加载当前结果到栈]
    LoadResult --> BitcastExtend[类型转换和位扩展]
    BitcastExtend --> ShiftAndOR[移位并与结果OR]
    ShiftAndOR --> UpdateResult[更新存储结果]
    
    HandleNormalStruct --> AllocStructStack[分配栈空间]
    AllocStructStack --> CreateOffsetPtr[创建offset指针]
    CreateOffsetPtr --> StructFieldLoop[循环处理结构体字段]
    StructFieldLoop --> CheckComptimeField{是否是编译时字段?}
    CheckComptimeField --> |否| AdjustPtrOffset[调整指针偏移]
    AdjustPtrOffset --> StoreField[存储字段值]
    
    All分支 --> FinalizeResult[最终结果处理]
    FinalizeResult --> CheckElementsLen{元素数量<=Liveness.bpi-1?}
    CheckElementsLen --> |是| CopyToBuffer[拷贝到缓冲区并返回]
    CheckElementsLen --> |否| IterateBigTomb[处理大tomb并返回]
    
    ReturnError --> FinalizeResult
    NextElementRef --> CheckSentinel{存在哨兵?}
    CheckSentinel --> |是| StoreSentinelRef[存储哨兵]
    NextElementVal --> CheckSentinelVal{存在哨兵?}
    CheckSentinelVal --> |是| StoreSentinelVal[存储哨兵]
    
    StoreSentinelRef --> FinalizeResult
    StoreSentinelVal --> FinalizeResult
    UpdateResult --> FinalizeResult
    StoreField --> FinalizeResult
