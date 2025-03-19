graph TD
    A[开始flushModule] --> B[处理LLVM对象]
    B --> C[创建子进度节点]
    C --> D[处理模块对象路径]
    D --> E[检查verbose_link并dump参数]
    E --> F{是否为静态库或对象文件?}
    F --> |是| G[调用relocatable对应方法]
    F --> |否| H[收集positional输入]
    H --> I[添加C对象文件]
    I --> J[添加模块对象路径]
    J --> K[添加sanitizer库]
    K --> L[处理UBSAN运行时]
    L --> M[分类所有输入文件]
    M --> N[处理系统库和框架]
    N --> O[处理动态库输入]
    O --> P[处理编译器运行时]
    P --> Q[解析输入文件]
    Q --> R[处理依赖动态库]
    R --> S[检查诊断错误]
    S --> |有错误| T[返回LinkFailure]
    S --> |无错误| U[创建内部对象]
    U --> V[符号解析]
    V --> W[转换暂定定义]
    W --> X[字面量去重]
    X --> Y[GC段回收]
    Y --> Z[检查重复符号]
    Z --> AA[标记导入导出]
    AA --> AB[处理dylib序号]
    AB --> AC[扫描重定位]
    AC --> AD[初始化输出节]
    AD --> AE[生成展开信息]
    AE --> AF[初始化段]
    AF --> AG[分配段和节]
    AG --> AH[重设节大小]
    AH --> AI[写入节并更新linkedit]
    AI --> AJ[分配linkedit段]
    AJ --> AK[处理代码签名预留]
    AK --> AL[写入加载命令]
    AL --> AM[写入头信息]
    AM --> AN[写入UUID]
    AN --> AO[处理调试符号]
    AO --> AP{需要代码签名?}
    AP --> |是| AQ[写入签名并刷新缓存]
    AP --> |否| AR[结束流程]
    AQ --> AR
    T --> AR
    
    style A fill:#4CAF50,stroke:#388E3C
    style T fill:#FF5252,stroke:#D32F2F
    style AR fill:#4CAF50,stroke:#388E3C
    
    classDef error fill:#FFEBEE,stroke:#FFCDD2;
    class T error
