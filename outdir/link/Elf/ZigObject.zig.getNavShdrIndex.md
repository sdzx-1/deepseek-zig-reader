graph TD
    A[开始] --> B{nav_val是函数类型?}
    B -- 是 --> C{text_index存在?}
    C -- 是 --> D[返回现有.text段]
    C -- 否 --> E[创建.text段]
    E --> F[设置text_index并返回新段]
    
    B -- 否 --> G[解析变量属性: is_const/is_threadlocal/nav_init]
    G --> H{any_non_single_threaded且是线程本地?}
    H -- 是 --> I{是.bss段?}
    I -- 是 --> J{tbss_index存在?}
    J -- 是 --> K[返回现有.tbss段]
    J -- 否 --> L[创建.tbss段]
    L --> M[设置tbss_index并返回新段]
    
    I -- 否 --> N{tdata_index存在?}
    N -- 是 --> O[返回现有.tdata段]
    N -- 否 --> P[创建.tdata段]
    P --> Q[设置tdata_index并返回新段]
    
    H -- 否 --> R{是常量?}
    R -- 是 --> S{data_relro_index存在?}
    S -- 是 --> T[返回现有.data.rel.ro段]
    S -- 否 --> U[创建.data.rel.ro段]
    U --> V[设置data_relro_index并返回新段]
    
    R -- 否 --> W{nav_init未定义且Debug/ReleaseSafe模式?}
    W -- 是 --> X{data_index存在?}
    X -- 是 --> Y[返回现有.data段]
    X -- 否 --> Z[创建.data段]
    Z --> AA[设置data_index并返回新段]
    
    W -- 否 --> AB{是.bss段?}
    AB -- 是 --> AC{bss_index存在?}
    AC -- 是 --> AD[返回现有.bss段]
    AC -- 否 --> AE[创建.bss段]
    AE --> AF[设置bss_index并返回新段]
    
    AB -- 否 --> AG{data_index存在?}
    AG -- 是 --> AH[返回现有.data段]
    AG -- 否 --> AI[创建.data段]
    AI --> AJ[设置data_index并返回新段]
