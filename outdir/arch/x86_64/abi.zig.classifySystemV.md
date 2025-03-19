flowchart TD
    Start([开始]) --> Initialize[初始化memory_class和result数组]
    Initialize --> TypeSwitch{根据类型选择分支}
    
    TypeSwitch -->|指针类型| PointerSwitch{指针大小判断}
    PointerSwitch -->|slice指针| SlicePointer[result[0]=.integer, result[1]=.integer]
    PointerSwitch -->|其他指针| OtherPointer[result[0]=.integer]
    
    TypeSwitch -->|int/enum/error_set| IntBranch{bits大小判断}
    IntBranch -->|<=64| Int64[result[0]=.integer]
    IntBranch -->|<=128| Int128[result[0-1]=.integer]
    IntBranch -->|<=192| Int192[result[0-2]=.integer]
    IntBranch -->|<=256| Int256[result[0-3]=.integer]
    IntBranch -->|>256| MemoryClassInt[返回memory_class]
    
    TypeSwitch -->|bool/void/noreturn| SimpleType[result[0]=.integer
    
    TypeSwitch -->|float| FloatSwitch{浮点位数判断}
    FloatSwitch -->|16位| Float16[根据上下文设置.memory或.sse]
    FloatSwitch -->|32位| Float32[result[0]=.float]
    FloatSwitch -->|64位| Float64[result[0]=.sse]
    FloatSwitch -->|128位| Float128[result[0]=.sse, result[1]=.sseup]
    FloatSwitch -->|80位| Float80[result[0]=.x87, result[1]=.x87up]
    
    TypeSwitch -->|vector| VectorBranch{向量处理}
    VectorBranch -->|布尔向量| BoolVector[根据bits设置整数/SSE]
    VectorBranch -->|普通向量| NormalVector[根据bits和AVX/AVX512支持设置SSE/SSEUP]
    VectorBranch -->|超过支持范围| MemoryClassVector[返回memory_class]
    
    TypeSwitch -->|optional| OptionalBranch{判断是否payload}
    OptionalBranch -->|是| RecursiveCall[递归处理子类型]
    OptionalBranch -->|否| MemoryClassOpt[返回memory_class]
    
    TypeSwitch -->|struct/union| StructUnionBranch{结构体/联合体处理}
    StructUnionBranch -->|packed布局| PackedLayout[根据大小设置integer]
    StructUnionBranch -->|非packed且大小>64| MemoryClassStruct[返回memory_class]
    StructUnionBranch -->|递归处理字段| FieldProcessing[遍历字段分类]
    FieldProcessing --> PostMergeCheck{合并后检查}
    PostMergeCheck -->|存在.memory或无效x87up| MemoryClassPost[返回memory_class]
    PostMergeCheck -->|大小>16且首非SSE| MemoryClassSize[返回memory_class]
    PostMergeCheck -->|修正sseup| FixSSEUP[调整sseup为sse]
    
    TypeSwitch -->|array| ArrayBranch{数组处理}
    ArrayBranch -->|大小<=8| ArraySmall[result[0]=.integer]
    ArrayBranch -->|大小<=16| ArrayMedium[result[0-1]=.integer]
    ArrayBranch -->|大小>16| MemoryClassArray[返回memory_class]
    
    TypeSwitch -->|其他类型| Unreachable[无法到达]
    
    SlicePointer --> ReturnResult
    OtherPointer --> ReturnResult
    Int64 --> ReturnResult
    Int128 --> ReturnResult
    Int192 --> ReturnResult
    Int256 --> ReturnResult
    MemoryClassInt --> ReturnResult
    SimpleType --> ReturnResult
    Float16 --> ReturnResult
    Float32 --> ReturnResult
    Float64 --> ReturnResult
    Float128 --> ReturnResult
    Float80 --> ReturnResult
    MemoryClassVector --> ReturnResult
    RecursiveCall --> ReturnResult
    MemoryClassOpt --> ReturnResult
    PackedLayout --> ReturnResult
    MemoryClassStruct --> ReturnResult
    FixSSEUP --> ReturnResult
    ArraySmall --> ReturnResult
    ArrayMedium --> ReturnResult
    MemoryClassArray --> ReturnResult
    
    ReturnResult[[返回result数组]]
