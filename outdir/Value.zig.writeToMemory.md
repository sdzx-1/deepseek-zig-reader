graph TD
    Start([Start]) --> CheckUndef{val.isUndef?}
    CheckUndef -- Yes --> Memset[用0xaa填充buffer并返回]
    CheckUndef -- No --> TypeSwitch[类型判断]

    TypeSwitch --> |void| Void[空操作]
    TypeSwitch --> |bool| Bool[写入bool值到buffer[0]]
    TypeSwitch --> |int/enum/error_set/pointer| IntType[处理整数类型]
    TypeSwitch --> |float| FloatType[根据精度写入浮点数]
    TypeSwitch --> |array| ArrayType[递归处理数组元素]
    TypeSwitch --> |vector| VectorType[调用packed内存写入]
    TypeSwitch --> |struct| StructType[处理结构体布局]
    TypeSwitch --> |union| UnionType[处理联合体类型]
    TypeSwitch --> |optional| OptionalType[处理可选类型]
    TypeSwitch --> |其他类型| Unimplemented[返回Unimplemented错误]

    IntType --> PointerCheck{是否为指针类型?}
    PointerCheck -- Yes --> SliceCheck{是否为切片?}
    SliceCheck -- Yes --> IllDefined[返回布局错误]
    SliceCheck -- No --> CheckAddrTag[检查地址标签]
    CheckAddrTag -- 非整数 --> ReinterpretDeclRef[返回声明引用错误]
    PointerCheck -- No --> ProcessInt[处理整数信息]
    ProcessInt --> WriteBigInt[写入大整数到buffer]

    FloatType --> |16/32/64/80/128位| WriteFloat[调用对应精度的writeInt]

    ArrayType --> InitLoop[初始化循环变量]
    InitLoop --> LoopCheck{elem_i < len?}
    LoopCheck -- Yes --> GetElem[获取数组元素]
    GetElem --> Recurse[递归调用writeToMemory]
    Recurse --> UpdateOffset[更新buffer偏移]
    UpdateOffset --> LoopCheck
    LoopCheck -- No --> EndLoop[结束循环]

    StructType --> LayoutCheck{结构体布局}
    LayoutCheck --> |auto| StructAuto[返回布局错误]
    LayoutCheck --> |extern| StructExtern[遍历字段写入]
    LayoutCheck --> |packed| StructPacked[调用packed内存写入]

    UnionType --> ContainerLayout{容器布局}
    ContainerLayout --> |auto| UnionAuto[返回布局错误]
    ContainerLayout --> |extern| UnionExtern[处理标签字段]
    ContainerLayout --> |packed| UnionPacked[调用packed内存写入]

    OptionalType --> CheckOptional{是否为指针式可选类型?}
    CheckOptional -- No --> IllDefined
    CheckOptional -- Yes --> OptionalValue[获取可选值]
    OptionalValue -- Some --> RecurseSome[递归写入值]
    OptionalValue -- None --> WriteNull[写入0值]

    classDef error fill:#f9d5d5,stroke:#b71c1c
    class IllDefined,ReinterpretDeclRef,Unimplemented error
