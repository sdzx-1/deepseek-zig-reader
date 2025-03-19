flowchart TD
    Start[开始] --> CheckUndef{val.isUndefDeep?}
    CheckUndef -->|是| Undef[填充0xaa并返回]
    CheckUndef -->|否| SwitchType[根据值的类型进入分支]

    SwitchType -->|.simple_value| SimpleValue
    SwitchType -->|.int| IntType[写入二进制补码]
    SwitchType -->|.err| ErrType[写入错误值]
    SwitchType -->|.error_union| ErrorUnion[处理错误联合]
    SwitchType -->|.enum_tag| EnumTag[递归处理枚举标签]
    SwitchType -->|.float| FloatType[写入浮点值]
    SwitchType -->|.ptr| PtrType[递归处理指针]
    SwitchType -->|.slice| SliceType[递归处理ptr和len]
    SwitchType -->|.opt| OptType[处理可选类型]
    SwitchType -->|.aggregate| Aggregate[处理复合类型]
    SwitchType -->|.un| UnionType[处理联合类型]

    SimpleValue -->|.false/.true| WriteBool[写入0或1]
    SimpleValue -->|其他| Skip[直接返回]

    ErrorUnion --> CheckPayload{payload_ty是否有运行时数据?}
    CheckPayload -->|否| WriteErrorVal[直接写入错误值]
    CheckPayload -->|是| AlignCheck[对齐判断]
    AlignCheck -->|error_align > payload_align| ErrorFirst[先写错误值再写payload]
    AlignCheck -->|error_align <= payload_align| PayloadFirst[先写payload再写错误值]
    ErrorFirst --> RecursivePayload[递归生成payload并填充对齐]
    PayloadFirst --> RecursivePayload

    Aggregate --> SwitchContainer[根据容器类型分支]
    SwitchContainer -->|数组/向量| ProcessElements[递归处理每个元素]
    SwitchContainer -->|结构体| StructLayout[处理结构体布局]
    SwitchContainer -->|元组| TupleProcess[递归处理元组字段]

    StructLayout -->|packed| PackedStruct[位填充字段]
    StructLayout -->|auto/extern| AutoStruct[处理字段偏移和对齐]
    PackedStruct --> WritePacked[直接写入位数据]
    AutoStruct --> FieldIteration[按偏移逐个处理字段并填充]

    UnionType --> LayoutCheck[判断标签位置]
    LayoutCheck -->|tag在前| WriteTagFirst[先写标签再写payload]
    LayoutCheck -->|tag在后| WritePayloadFirst[先写payload再写标签]
    WriteTagFirst --> RecursivePayload
    WritePayloadFirst --> RecursivePayload

    OptType --> CheckRepr{是否为payload表示?}
    CheckRepr -->|是| WritePayloadOrZero[直接写payload或全零]
    CheckRepr -->|否| WriteBoolPadding[写入有效性标记并填充]

    RecursivePayload -.->|递归调用| generateSymbol
    ProcessElements -.->|递归调用| generateSymbol
    FieldIteration -.->|递归调用| generateSymbol

    Undef --> End[结束]
    WriteBool --> End
    IntType --> End
    WriteErrorVal --> End
    RecursivePayload --> End
    WriteFloat --> End
    PtrType --> End
    SliceType --> End
    OptType --> End
    StructLayout --> End
    UnionType --> End
