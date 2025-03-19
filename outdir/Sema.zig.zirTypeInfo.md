graph TD
    A[开始] --> B[初始化变量: pt, zcu, gpa, ip]
    B --> C[解析类型inst_data.operand得到ty]
    C --> D[获取内置类型type_info_ty和type_info_tag_ty]
    D --> E{检查类型声明依赖}
    E -->|存在type_decl_inst| F[声明依赖关系]
    E -->|不存在| G[进入类型分支处理]
    F --> G

    G --> H{zigTypeTag(ty)类型判断}
    
    %% 基础类型分支
    H -->|type/void/bool等| I[直接返回unionInitFromEnumTag]
    
    %% 函数类型分支
    H -->|.@""fn""| J1[创建函数参数列表]
    J1 --> J2[构建参数元数据]
    J2 --> J3[生成调用约定和返回类型]
    J3 --> J4[构造Type.Fn结构]
    J4 --> Z
    
    %% 整数类型分支
    H -->|.int| K1[获取有符号性和位宽]
    K1 --> K2[构造Type.Int结构]
    K2 --> Z
    
    %% 指针类型分支
    H -->|.pointer| L1[解析指针属性: size/is_const/alignment等]
    L1 --> L2[构造Type.Pointer结构]
    L2 --> Z
    
    %% 结构体类型分支
    H -->|.@""struct""| M1[解析字段布局]
    M1 --> M2[遍历所有结构体字段]
    M2 --> M3[生成字段元数据: 名称/类型/默认值]
    M3 --> M4[构造Type.Struct结构]
    M4 --> Z
    
    %% 错误集合分支
    H -->|.error_set| N1[解析错误名称列表]
    N1 --> N2[构造错误切片类型]
    N2 --> N3[生成Type.Error结构]
    N3 --> Z
    
    %% 枚举类型分支
    H -->|.@""enum""| O1[遍历枚举标签]
    O1 --> O2[生成标签名称和值]
    O2 --> O3[构造Type.Enum结构]
    O3 --> Z
    
    %% 联合体分支
    H -->|.@""union""| P1[解析联合体布局]
    P1 --> P2[遍历所有联合字段]
    P2 --> P3[生成字段元数据]
    P3 --> P4[构造Type.Union结构]
    P4 --> Z
    
    %% 其他类型分支
    H -->|其他类型| Q[类似模式处理...]
    Q --> Z
    
    Z[返回Air.internedToRef结果]
    
    %% 特殊处理分支
    H -->|.frame/.anyframe| R[返回异步错误]
    
    classDef logic fill:#f9f,stroke:#333;
    classDef data fill:#6f9,stroke:#333;
    classDef result fill:#9ff,stroke:#333;
    class H,J1,M1,N1,O1,P1 logic;
    class I,J4,K2,L2,M4,N3,O3,P4,Z data;
    class A,B,C,D,E,F,G,R result;
