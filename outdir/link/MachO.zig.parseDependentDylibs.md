flowchart TD
    A[开始 parseDependentDylibs] --> B[初始化 tracy 跟踪]
    B --> C{检查 dylibs 是否为空?}
    C --> |是| D[结束 tracy 跟踪并返回]
    C --> |否| E[初始化 arena_alloc 和路径变量]
    E --> F[进入循环：遍历所有 dylibs]
    F --> G{是否处理完所有 dylibs?}
    G --> |是| H{存在错误?}
    G --> |否| I[获取当前 dylib 信息]
    I --> J[初始化 dependents 列表]
    J --> K[遍历依赖项]
    K --> L{路径类型判断}
    L --> |框架目录| M[尝试访问框架路径]
    L --> |库目录| N[尝试访问库路径]
    L --> |绝对路径| O[拼接 sysroot 并检查]
    L --> |@rpath| P[解析 rpath 并生成真实路径]
    L --> |@loader_path| Q[报告未处理错误]
    L --> |@executable_path| R[报告未处理错误]
    L --> |默认路径| S[尝试直接访问路径]
    M --> T{找到有效路径?}
    N --> T
    O --> T
    P --> T
    Q --> T
    R --> T
    S --> T
    T --> |是| U[打开文件并解析 Mach-O 头]
    T --> |否| V[记录缺失依赖错误]
    V --> W[标记错误并继续]
    U --> X{是否为 MH_DYLIB?}
    X --> |是| Y[添加为动态库]
    X --> |否| Z[添加为 TBD 文件]
    Y --> AA[将文件索引加入 dependents]
    Z --> AA
    AA --> K
    K --> |处理完所有依赖项| AB[更新 umbrella 的导出和 rpaths]
    AB --> F
    H --> |是| AC[返回 MissingLibraryDependencies 错误]
    H --> |否| AD[正常结束]
