graph TD
    A[开始lowerInt] --> B{节点类型?}
    B -->|int_literal| C[int类型处理]
    B -->|float_literal| D[浮点处理]
    B -->|char_literal| E[字符处理]
    B -->|其他| F[返回WrongType错误]

    C --> G{small或big?}
    G -->|small| H[检查small值范围]
    G -->|big| I[检查big值是否适配目标类型]

    H --> J[目标类型是整数?]
    J -->|是| K[检查符号和位数]
    K --> L{是否超出范围?}
    L -->|是| M[返回错误]
    L -->|否| N[生成i64存储结果]
    J -->|否| N

    I --> O[检查big整数是否适配]
    O -->|是| P[生成big_int存储结果]
    O -->|否| M

    D --> Q[检查是否有小数部分]
    Q -->|是| R[返回错误]
    Q -->|否| S[转换为有理数]
    S --> T[验证分母为1]
    T --> U[检查是否适配目标类型]
    U -->|是| V[生成big_int存储结果]
    U -->|否| R

    E --> W[目标类型是整数?]
    W -->|是| X[检查字符值范围]
    X --> Y{是否超出范围?}
    Y -->|是| Z[返回错误]
    Y -->|否| AA[生成i64存储结果]
    W -->|否| AA

    N --> AB[返回结果]
    P --> AB
    V --> AB
    AA --> AB
    M --> AC[结束错误]
    R --> AC
    Z --> AC
    F --> AC
