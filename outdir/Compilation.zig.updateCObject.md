graph TD
    A[开始 updateCObject] --> B{检查 c_frontend 是否为 aro}
    B -->|是| C[返回错误: aro 不支持]
    B -->|否| D{检查是否有 LLVM}
    D -->|否| E[返回错误: 无 clang]
    D -->|是| F{检查 self_exe_path}
    F -->|不存在| G[返回错误: clang 禁用]
    F -->|存在| H[记录调试日志]
    H --> I[清除之前的失败状态]
    I --> J[初始化缓存 manifest]
    J --> K[添加编译选项到哈希]
    K --> L[处理 C 源文件哈希]
    L --> M[创建 Arena 分配器]
    M --> N{是否为单一 C 文件构建对象}
    N -->|是| O[设置直接输出文件名]
    N -->|否| P[使用源文件名]
    O --> Q[确定目标文件扩展名]
    P --> Q
    Q --> R{是否命中缓存}
    R -->|是| S[获取缓存摘要]
    R -->|否| T[构建 clang 命令行参数]
    T --> U{是否启用 passthrough 模式}
    U -->|是| V[执行 execv 调用 clang]
    U -->|否| W[生成临时输出路径]
    W --> X[处理依赖文件和诊断文件]
    X --> Y[执行子进程编译]
    Y --> Z{编译是否成功}
    Z -->|失败| AA[记录错误信息]
    Z -->|成功| AB[处理依赖文件]
    AB --> AC[重命名输出文件]
    AC --> AD[生成缓存摘要]
    AD --> AE[更新 manifest]
    AE --> AF[设置对象状态为成功]
    AF --> AG[触发后续链接任务]
    AG --> AH[结束]
    S --> AF
    AA --> AH
    C --> AH
    E --> AH
    G --> AH
