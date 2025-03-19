graph TD
    A[开始 populateTestFunctions] --> B[获取zcu, gpa, ip]
    B --> C[导入builtin模块并分析]
    C --> D{分析成功?}
    D -- 是 --> E[获取builtin根类型和命名空间]
    D -- 否 --> Z[返回错误]
    E --> F[查找'test_functions'声明]
    F --> G[启动语义分析进度节点]
    G --> H[确保导航值更新]
    H --> I{更新成功?}
    I -- 是 --> J[获取test_functions值和类型]
    I -- 否 --> Z
    J --> K[分配test_fn_vals数组]
    K --> L[遍历测试函数]
    L --> M{测试函数分析失败?}
    M -- 是 --> N[提前返回]
    M -- 否 --> O[处理测试函数名称]
    O --> P[构造测试函数结构体]
    P --> Q[填充到数组]
    Q --> L
    L --> R[创建数组类型并初始化]
    R --> S[更新切片指针信息]
    S --> T[启动代码生成进度节点]
    T --> U[更新链接器导航]
    U --> V[结束进度节点]
    V --> W[函数结束]
    Z --> W
    N --> W
    style A stroke:#333,stroke-width:2px
    style W stroke:#090,stroke-width:2px
    style Z stroke:#f00,stroke-width:2px
    style N stroke:#f00,stroke-width:2px

    subgraph 循环处理测试函数
    L --> M
    M --> O
    O --> P
    P --> Q
    Q --> L
    end
