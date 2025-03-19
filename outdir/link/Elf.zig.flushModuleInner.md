graph TD
    A[开始flushModuleInner] --> B[处理模块对象路径]
    B --> C{是否有ZigObject需要flush?}
    C --> |是| D[调用zig_object.flush]
    C --> |否| E[继续流程]
    D --> E
    E --> F{输出模式是什么?}
    F --> |Obj| G[调用relocatable.flushObject]
    F --> |Lib| H{链接模式是动态还是静态?}
    H --> |static| I[调用relocatable.flushStaticLib]
    H --> |dynamic| J[继续流程]
    F --> |Exe| J
    G --> K[检查错误]
    I --> K
    J --> K
    K --> L{存在错误?}
    L --> |是| M[返回LinkFailure]
    L --> |否| N[创建linker_defined输入文件]
    N --> O[跨文件解析符号]
    O --> P[标记EhFrameAtoms为dead]
    P --> Q[解析合并节]
    Q --> R[转换公共符号]
    R --> S[标记导入导出]
    S --> T{启用GC?}
    T --> |是| U[执行GC]
    U --> V{打印GC节?}
    V --> |是| W[输出GC节信息]
    W --> X[检查重复符号]
    V --> |否| X
    T --> |否| X
    X --> Y{发现重复?}
    Y --> |是| Z[返回LinkFailure]
    Y --> |否| AA[添加注释字符串]
    AA --> AB[最终化合并节]
    AB --> AC[初始化输出节]
    AC --> AD[初始化start/stop符号]
    AD --> AE[处理未解析符号]
    AE --> AF[扫描重定位]
    AF --> AG[生成合成节]
    AG --> AH[初始化特殊Phdrs]
    AH --> AI[排序节头]
    AI --> AJ[设置动态节]
    AJ --> AK[排序动态符号表]
    AK --> AL[设置哈希节]
    AL --> AM[设置版本符号表]
    AM --> AN[排序init/fini]
    AN --> AO[更新节大小]
    AO --> AP[添加加载Phdrs]
    AP --> AQ[分配Phdr表]
    AQ --> AR[分配alloc节]
    AR --> AS[排序Phdrs]
    AS --> AT[分配非alloc节]
    AT --> AU[分配特殊Phdrs]
    AU --> AV[调试日志输出]
    AV --> AW[清理对象状态]
    AW --> AX[处理重定位]
    AX --> AY{存在重定位错误?}
    AY --> |是| AZ[返回LinkFailure]
    AY --> |否| BA[写入Phdr表]
    BA --> BB[写入Shdr表]
    BB --> BC[写入Atoms]
    BC --> BD[写入合并节]
    BD --> BE[写入合成节]
    BE --> BF{存在错误?}
    BF --> |是| BG[返回LinkFailure]
    BF --> |否| BH{是Exe且无entry point?}
    BH --> |是| BI[设置无入口点标志]
    BH --> |否| BJ[写入ELF头]
    BI --> BK[最终错误检查]
    BJ --> BK
    BK --> BL{存在错误?}
    BL --> |是| BM[返回LinkFailure]
    BL --> |否| BN[正常结束]
