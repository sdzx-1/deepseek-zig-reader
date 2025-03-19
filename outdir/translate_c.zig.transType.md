graph TD
    A[开始: transType函数] --> B{类型类别 ty.getTypeClass()}
    B -->|Builtin| C[处理内置类型]
    C --> D[获取builtin_ty]
    D --> E{内置类型种类 builtin_ty.getKind()}
    E -->|Void| F[返回"anyopaque"]
    E -->|Bool| G[返回"bool"]
    E -->|Char_U/UChar/...| H[返回"u8"]
    E -->|SChar| I[返回"i8"]
    E -->|UShort| J[返回"c_ushort"]
    E -->|...其他内置类型...| K[返回对应类型字符串]
    E -->|其他未支持类型| L[调用fail报错]

    B -->|FunctionProto| M[处理函数原型]
    M --> N[转换函数原型fn_proto_ty]
    N --> O[返回fn_proto节点]

    B -->|FunctionNoProto| P[处理无原型函数]
    P --> Q[转换函数类型fn_no_proto_ty]
    Q --> R[返回fn_proto节点]

    B -->|Paren| S[处理括号类型]
    S --> T[递归调用transQualType处理内部类型]
    
    B -->|Pointer| U[处理指针类型]
    U --> V[获取指针指向类型child_qt]
    V --> W[判断是否为函数原型/const/volatile]
    W --> X[递归转换elem_type]
    X --> Y{是否函数原型/不透明类型?}
    Y -->|是| Z[创建optional指针]
    Y -->|否| AA[创建c_pointer]
    
    B -->|ConstantArray| AB[处理定长数组]
    AB --> AC[获取数组长度和元素类型]
    AC --> AD[返回数组类型节点]
    
    B -->|IncompleteArray| AE[处理不定长数组]
    AE --> AF[转换元素类型并创建指针]
    
    B -->|Typedef| AG[处理typedef类型]
    AG --> AH[检查全局/内置类型]
    AH -->|存在映射| AI[返回内置类型]
    AH -->|无映射| AJ[生成typedef名称节点]
    
    B -->|Record/Enum| AK[处理结构体/枚举]
    AK --> AL[转换声明并返回名称节点]
    
    B -->|Elaborated/Decayed/...| AM[处理修饰类型]
    AM --> AN[递归调用transQualType处理等效类型]
    
    B -->|Vector| AO[处理向量类型]
    AO --> AP[转换元素类型和数量]
    AP --> AQ[返回向量节点]
    
    B -->|BitInt/ExtVector| AR[暂未实现类型]
    AR --> AS[调用fail报错]
    
    B -->|其他未支持类型| AT[调用fail报错]
    
    C --> L
    E --> L
    U --> Z
    U --> AA
    AG --> AI
    AG --> AJ
    B --> AM
    B --> AO
    B --> AR
    B --> AT
