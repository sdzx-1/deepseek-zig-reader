graph TD
    A[开始: updateFile] --> B[检查文件非内置模块]
    B --> C[获取文件stat信息]
    C --> D{文件状态判断}
    D --> |状态为 never_loaded 或 retryable_failure| E[获取共享锁]
    D --> |状态为 astgen_failure 或 success| F[检查metadata是否变化]
    F --> |metadata未变化| G[直接返回]
    F --> |metadata变化| H[获取独占锁]
    E --> I[清除旧编译错误]
    H --> I
    I --> J{是否存在旧ZIR需要保留?}
    J --> |是| K[保存旧ZIR到prev_zir]
    J --> |否| L[卸载所有旧数据]
    K --> L
    L --> M[循环创建/打开缓存文件]
    M --> N{缓存文件是否有效?}
    N --> |有效| O[加载缓存数据]
    O --> P[标记无需更新]
    N --> |无效或过期| Q[标记需要更新]
    Q --> R[清空缓存文件内容]
    R --> S[读取源文件内容]
    S --> T[解析生成AST]
    T --> U{文件模式判断}
    U --> |.zig| V[生成ZIR并保存到缓存]
    U --> |.zon| W[生成ZOIR并保存到缓存]
    V --> X[更新文件状态]
    W --> X
    X --> Y{是否有编译错误?}
    Y --> |是| Z[记录失败状态]
    Y --> |否| AA[标记为成功]
    Z --> AB[返回AnalysisFail]
    AA --> AC[返回成功]
    P --> X
    G --> AC
