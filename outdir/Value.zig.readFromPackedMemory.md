graph TD
    Start([开始]) --> TypeCheck{检查类型标签}
    
    TypeCheck -->|void| Void[返回Value.void]
    TypeCheck -->|bool| Bool
    TypeCheck -->|int| Int
    TypeCheck -->|enum| Enum
    TypeCheck -->|float| Float
    TypeCheck -->|vector| Vector
    TypeCheck -->|struct| Struct
    TypeCheck -->|union| Union
    TypeCheck -->|pointer| Pointer
    TypeCheck -->|optional| Optional
    TypeCheck -->|其他类型| Panic[触发panic]
    
    %% 布尔类型处理
    Bool --> GetByte[根据字节序获取字节]
    GetByte --> BitCheck[检查特定位是否为1]
    BitCheck -->|0| ReturnFalse[返回Value.false]
    BitCheck -->|1| ReturnTrue[返回Value.true]
    
    %% 整型处理
    Int --> CheckBits{bits <=64?}
    CheckBits -->|是| FastPath[快速路径：直接读取整数值]
    CheckBits -->|否| SlowPath[慢速路径：构建大整数]
    FastPath --> ReturnInt[返回整数值]
    SlowPath --> AllocLimbs[分配大整数内存]
    AllocLimbs --> ReadBigInt[读取打包的补码]
    ReadBigInt --> ReturnBigInt[返回大整数值]
    
    %% 枚举类型处理
    Enum --> ReadInt[递归读取底层整型]
    ReadInt --> Coerce[强制转换为枚举类型]
    
    %% 浮点类型处理
    Float --> ReadPacked[根据位数读取打包数据]
    ReadPacked --> ConstructFloat[构造浮点数值]
    
    %% 向量类型处理
    Vector --> LoopVector[循环处理每个元素]
    LoopVector --> AdjustIndex[调整大端元素顺序]
    AdjustIndex --> ReadElement[递归读取元素值]
    ReadElement --> BuildVector[构建向量聚合值]
    
    %% 结构体类型处理
    Struct --> LoopFields[循环处理每个字段]
    LoopFields --> ReadField[递归读取字段值]
    ReadField --> BuildStruct[构建结构体聚合值]
    
    %% 联合类型处理
    Union --> ReadBacking[读取底层类型值]
    ReadBacking --> WrapUnion[包装为联合类型值]
    
    %% 指针类型处理
    Pointer --> ReadAddr[读取地址整数值]
    ReadAddr --> ConstructPtr[构造指针值]
    
    %% 可选类型处理
    Optional --> CheckNull{是否为0?}
    CheckNull -->|是| ReturnNull[返回.none]
    CheckNull -->|否| WrapOptional[包装可选值]
    
    %% 公共结束节点
    Void --> End
    ReturnFalse --> End
    ReturnTrue --> End
    ReturnInt --> End
    ReturnBigInt --> End
    Coerce --> End
    ConstructFloat --> End
    BuildVector --> End
    BuildStruct --> End
    WrapUnion --> End
    ConstructPtr --> End
    ReturnNull --> End
    WrapOptional --> End
    Panic --> End
    
    End([结束])
    
    classDef decision fill:#f9f,stroke:#333,stroke-width:2px;
    classDef process fill:#9f9,stroke:#333,stroke-width:2px;
    class TypeCheck,CheckBits,CheckNull decision
    class GetByte,BitCheck,FastPath,SlowPath,ReadInt,ReadPacked,LoopVector,ReadElement,LoopFields,ReadField,ReadBacking,ReadAddr process
