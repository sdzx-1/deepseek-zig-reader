flowchart TD
    Start([开始]) --> CheckIndex{检查 index 类型}
    
    CheckIndex --> |直接匹配类型| ReturnTypeID[返回对应 TypeId]
    CheckIndex --> |默认分支 _| UnwrapIndex[解包 index 并获取 Tag]
    
    UnwrapIndex --> CheckTag{检查 Tag 类型}
    
    CheckTag --> |.type_int_signed/.type_int_unsigned| Int[返回 .int]
    CheckTag --> |.type_array_big/.type_array_small| Array[返回 .array]
    CheckTag --> |.type_vector| Vector[返回 .vector]
    CheckTag --> |.type_pointer/.type_slice| Pointer[返回 .pointer]
    CheckTag --> |.type_optional| Optional[返回 .optional]
    CheckTag --> |.type_anyframe| Anyframe[返回 .anyframe]
    CheckTag --> |.type_error_union/.type_anyerror_union| ErrorUnion[返回 .error_union]
    CheckTag --> |.type_error_set/.type_inferred_error_set| ErrorSet[返回 .error_set]
    CheckTag --> |.type_enum_auto/.type_enum_explicit/.type_enum_nonexhaustive| Enum[返回 .enum]
    CheckTag --> |.type_opaque| Opaque[返回 .opaque]
    CheckTag --> |.type_struct/...| Struct[返回 .struct]
    CheckTag --> |.type_union| Union[返回 .union]
    CheckTag --> |.type_function| Fn[返回 .fn]
    CheckTag --> |其他无效 Tag| Unreachable[unreachable]
    
    ReturnTypeID --> End([结束])
    Int --> End
    Array --> End
    Vector --> End
    Pointer --> End
    Optional --> End
    Anyframe --> End
    ErrorUnion --> End
    ErrorSet --> End
    Enum --> End
    Opaque --> End
    Struct --> End
    Union --> End
    Fn --> End
    Unreachable --> End

    style Start fill:#9f9,stroke-width:0
    style End fill:#f9f,stroke-width:0
    style Unreachable fill:#f99,stroke-width:0
