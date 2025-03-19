graph TD
    Start[开始] --> CheckType{old_ty == new_ty?}
    CheckType -->|是| ReturnVal[返回 val]
    
    CheckType -->|否| SwitchVal[switch val]
    
    SwitchVal --> UndefCase[.undef]
    UndefCase --> ReturnUndef[返回 new_ty 的 undef]
    
    SwitchVal --> NullCase[.null_value]
    NullCase --> CheckOptType{new_ty 是可选类型?}
    CheckOptType -->|是| ReturnOptNone[返回 opt.none]
    
    CheckOptType -->|否| CheckPtrType{new_ty 是指针类型?}
    CheckPtrType -->|是| CheckPtrSize[检查指针大小]
    
    CheckPtrSize -->|one/many/c| ReturnPtr[返回 ptr 结构]
    CheckPtrSize -->|slice| ReturnSlice[返回 slice 结构]
    
    SwitchVal --> OtherCase[其他类型]
    OtherCase --> UnwrapVal[解包 val]
    UnwrapVal --> CheckFuncTag{标签是函数相关?}
    CheckFuncTag -->|func_decl| CallFuncDecl[调用 getCoercedFuncDecl]
    CheckFuncTag -->|func_instance| CallFuncInstance[调用 getCoercedFuncInstance]
    
    SwitchVal --> KeySwitch[根据 val 的 Key 分支]
    
    KeySwitch --> IntCase[.int]
    IntCase --> CheckNewType[检查 new_ty 类型]
    CheckNewType -->|枚举类型| ReturnEnumTag[返回 enum_tag]
    CheckNewType -->|指针类型| ReturnPtrFromInt[返回指针结构]
    CheckNewType -->|整数类型| CoerceInt[调用 getCoercedInts]
    
    KeySwitch --> FloatCase[.float]
    FloatCase --> ReturnFloat[返回对应精度的 float]
    
    KeySwitch --> SliceCase[.slice]
    SliceCase --> CheckSliceTarget{目标是 slice 指针?}
    CheckSliceTarget -->|是| ReturnCoercedSlice[递归转换 ptr 和保留 len]
    CheckSliceTarget -->|整数类型| ReturnPtrCoerced[递归转换 ptr]
    
    KeySwitch --> AggregateCase[.aggregate]
    AggregateCase --> CheckStorage[检查存储类型]
    CheckStorage -->|bytes/elems/repeated| HandleElements[处理元素]
    HandleElements --> CoerceElements[递归转换每个元素]
    CoerceElements --> ReturnNewAggregate[返回新聚合类型]
    
    KeySwitch --> OtherKeyCase[其他 Key 类型]
    OtherKeyCase --> FinalCheck[检查 new_ty 是否为可选/错误联合]
    
    FinalCheck -->|opt_type| ReturnOpt[返回包装后的 opt]
    FinalCheck -->|error_union| ReturnErrorUnion[返回错误联合]
    
    FinalCheck -->|未处理| Panic[触发 panic/unreachable]
    
    ReturnVal --> End[结束]
    ReturnUndef --> End
    ReturnOptNone --> End
    ReturnPtr --> End
    ReturnSlice --> End
    CallFuncDecl --> End
    CallFuncInstance --> End
    ReturnEnumTag --> End
    ReturnPtrFromInt --> End
    CoerceInt --> End
    ReturnFloat --> End
    ReturnCoercedSlice --> End
    ReturnPtrCoerced --> End
    ReturnNewAggregate --> End
    ReturnOpt --> End
    ReturnErrorUnion --> End
    Panic --> End
