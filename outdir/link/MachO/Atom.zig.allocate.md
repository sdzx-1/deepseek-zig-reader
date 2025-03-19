flowchart TD
    A([开始]) --> B[初始化变量]
    B --> C[遍历空闲列表]
    C --> D{i < 空闲列表长度?}
    D -- 是 --> E[获取当前空闲块big_atom]
    E --> F[计算容量和理想地址]
    F --> G{新地址是否合法?}
    G -- 是 --> H[检查剩余容量是否保留空闲节点]
    H --> I[记录atom_placement和free_list_removal]
    I --> J[跳出循环]
    G -- 否 --> K[更新或移除空闲项]
    K --> C
    D -- 否 --> L{存在最后一个Atom?}
    L -- 是 --> M[在末尾分配新地址]
    L -- 否 --> N[设置地址为0]
    M --> J
    N --> J
    J --> O[记录分配日志]
    O --> P{需要扩展节?}
    P -- 是 --> Q[扩展节并更新元数据]
    Q --> R[更新节对齐信息]
    P -- 否 --> R
    R --> S[解除旧链接关系]
    S --> T[更新前后Atom指针]
    T --> U[移除已处理的空闲项]
    U --> V[标记Atom为存活]
    V --> W([结束])

    style A fill:#4db8ff,stroke:#333
    style W fill:#4db8ff,stroke:#333
    style D fill:#ffcc00,stroke:#333
    style G fill:#ffcc00,stroke:#333
    style L fill:#ffcc00,stroke:#333
    style P fill:#ffcc00,stroke:#333
