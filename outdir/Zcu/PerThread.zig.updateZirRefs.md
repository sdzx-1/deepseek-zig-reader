graph TD
    A[开始] --> B[初始化变量: zcu, comp, ip, gpa]
    B --> C[创建updated_files哈希映射]
    C --> D[遍历zcu.import_table中的每个file_index]
    D --> E{文件类型是.zig?}
    E --> |是| F[检查旧ZIR和新ZIR是否存在]
    F --> G[将文件加入updated_files并映射旧指令到新指令]
    E --> |否| H[处理ZON文件并标记依赖失效]
    H --> I[继续下一个文件]
    G --> I
    I --> D
    D --> J{所有文件遍历完成?}
    J --> |否| D
    J --> |是| K{updated_files为空?}
    K --> |是| L[直接返回]
    K --> |否| M[遍历InternPool的locals]
    M --> N[遍历每个tracked_inst]
    N --> O{file_index在updated_files中?}
    O --> |否| P[继续下一个tracked_inst]
    O --> |是| Q[获取旧指令对应的新指令]
    Q --> R{映射成功?}
    R --> |否| S[标记为lost并触发依赖失效]
    R --> |是| T[更新tracked_inst的指令]
    T --> U[检查行号变化并触发更新]
    U --> V[检查哈希变化并触发依赖失效]
    V --> W{是否涉及命名空间?}
    W --> |否| X[继续下一个tracked_inst]
    W --> |是| Y[比较新旧命名空间声明]
    Y --> Z{有新增/删除/修改?}
    Z --> |是| AA[触发命名空间依赖失效]
    Z --> |否| X
    AA --> X
    X --> N
    N --> AB{所有tracked_inst处理完成?}
    AB --> |否| N
    AB --> |是| AC[重新哈希tracked_insts]
    AC --> AD[遍历updated_files]
    AD --> AE[释放旧ZIR资源]
    AE --> AF[更新文件命名空间]
    AF --> AD
    AD --> AG{所有文件处理完成?}
    AG --> |否| AD
    AG --> |是| AH[结束]
