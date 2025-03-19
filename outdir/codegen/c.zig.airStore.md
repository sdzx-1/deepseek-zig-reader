flowchart TD
    Start[开始] --> GetTypeInfo[获取类型信息]
    GetTypeInfo --> CheckUndef{检查值是否为undef?}
    CheckUndef -->|是| UndefHandling[处理undef情况]
    UndefHandling --> SafetyCheck{安全检查通过?}
    SafetyCheck -->|是| GenerateMemset[生成memset代码]
    SafetyCheck -->|否| ReapUndef[回收资源]
    GenerateMemset --> ReapUndef
    ReapUndef --> ReturnNone[返回.none]

    CheckUndef -->|否| CheckAlignment{检查内存对齐和数组类型}
    CheckAlignment --> NeedMemcpy{需要memcpy?}
    NeedMemcpy -->|是| HandleMemcpy[处理memcpy]
    HandleMemcpy --> IsConstant{源是常量?}
    IsConstant -->|是| CreateTempVar[创建临时变量]
    CreateTempVar --> GenerateMemcpy[生成memcpy代码]
    IsConstant -->|否| GenerateMemcpy
    GenerateMemcpy --> FreeTempVar[释放临时变量]
    FreeTempVar --> ReturnNone

    NeedMemcpy -->|否| CheckPackedOffset{检查packed offset?}
    CheckPackedOffset -->|是| HandleBitOps[处理位操作]
    HandleBitOps --> GenerateBitMask[生成掩码和位移代码]
    GenerateBitMask --> VectorizeBitOps[向量化位操作]
    VectorizeBitOps --> ReturnNone

    CheckPackedOffset -->|否| CheckSelfAssign{检查自赋值?}
    CheckSelfAssign -->|是| ReturnNone
    CheckSelfAssign -->|否| NormalAssignment[普通赋值]
    NormalAssignment --> GenerateAssignment[生成赋值代码]
    GenerateAssignment --> ReturnNone
