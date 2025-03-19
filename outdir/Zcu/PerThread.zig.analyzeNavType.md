flowchart TD
    A[开始] --> B[初始化变量: zcu, gpa, ip, anal_unit, old_nav]
    B --> C[记录日志]
    C --> D[解析inst_resolved]
    D -->|失败| E[返回AnalysisFail错误]
    D -->|成功| F[获取file和zir]
    F --> G[将anal_unit加入analysis_in_progress]
    G --> H[创建analysis_arena和comptime_err_ret_trace]
    H --> I[初始化sema结构体]
    I --> J[声明依赖关系]
    J --> K[初始化block结构体]
    K --> L{检查type_body是否存在?}
    L -->|不存在| M[声明nav_val依赖并更新]
    M --> N[返回type_changed=true]
    L -->|存在| O[解析类型并处理修饰符]
    O --> P[判断是否是const/is_extern_decl]
    P --> Q{比较新旧类型和修饰符是否相同?}
    Q -->|相同| R[返回type_changed=false]
    Q -->|不同| S[更新Nav类型信息]
    S --> T[返回type_changed=true]

    %% 资源清理部分（简化表示）
    G -->|defer| U[从analysis_in_progress移除anal_unit]
    H -->|defer| V[释放analysis_arena]
    I -->|defer| W[释放sema资源]
    K -->|defer| X[释放block.instructions]
