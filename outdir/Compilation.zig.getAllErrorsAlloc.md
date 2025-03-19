graph TD
    A[开始] --> B[初始化ErrorBundle]
    B --> C[处理failed_c_objects]
    C --> D[处理failed_win32_resources]
    D --> E[处理link_diags.lld错误]
    E --> F[处理misc_failures]
    F --> G[检查内存分配失败]
    G --> H{是否有ZCU?}
    H -->|是| I[处理ZCU文件错误]
    I --> J[排序failed_analysis]
    J --> K[处理分析错误]
    K --> L[处理codegen/type/export错误]
    L --> M[检查错误数量限制]
    H -->|否| N[检查空错误列表]
    M --> N
    N --> O{是否有空错误?}
    O -->|是| P[添加入口点错误]
    O -->|否| Q[处理libc缺失]
    P --> Q
    Q --> R[添加链接诊断信息]
    R --> S{是否有ZCU?}
    S -->|是| T[处理编译日志]
    T --> U[增量编译检查]
    S -->|否| U
    U --> V[生成最终ErrorBundle]
    V --> W[返回结果]

    subgraph ZCU错误处理
        I --> I1[遍历failed_files]
        I1 --> I2{是否有错误信息?}
        I2 -->|是| I3[添加模块错误]
        I2 -->|否| I4[检查ZIR/ZOIR错误]
        I4 --> I5[添加ZIR错误]
        I4 --> I6[添加ZOIR错误]
    end

    subgraph 分析错误排序
        J --> J1[克隆entries]
        J1 --> J2[自定义排序]
        J2 --> J3[处理排序错误]
    end

    subgraph 增量编译检查
        U --> U1[遍历transitive_failed_analysis]
        U1 --> U2{是否引用失败单元?}
        U2 -->|是| U3[输出调试信息并panic]
    end

    style A fill:#4CAF50,color:white
    style W fill:#4CAF50,color:white
    style U3 fill:#FF5252,color:white
