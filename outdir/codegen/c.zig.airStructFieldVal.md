graph TD
    A[开始] --> B[获取ty_pl和extra数据]
    B --> C{inst_ty是否有运行时位?}
    C --> |否| D[回收操作数, 返回.none]
    C --> |是| E[解析struct_operand得到struct_byval]
    E --> F[确保结构体类型定义可见]
    F --> G{结构体类型类别}
    G --> |struct_type| H{布局类型}
    G --> |tuple_type| I[直接使用字段索引]
    G --> |union_type| J{布局类型}
    H --> |auto/extern| K[获取字段名或索引]
    H --> |packed| L[计算位偏移和类型转换]
    L --> M[生成临时变量并位操作]
    M --> N[类型是否匹配?]
    N --> |否| O[内存拷贝并释放临时变量]
    N --> |是| P[直接返回临时变量]
    J --> |auto/extern| Q[获取联合体标签或字段名]
    J --> |packed| R[直接内存拷贝]
    I --> S[分配局部变量并赋值]
    K --> S
    Q --> S
    R --> S
    O --> S
    P --> S
    S --> T[返回局部变量]
