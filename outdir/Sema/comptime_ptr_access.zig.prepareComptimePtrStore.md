graph TD
    A[开始] --> B[解析指针值]
    B --> C{检查指针类型}
    C --> |undef| D[返回.undef]
    C --> |ptr| E[处理指针base_addr]
    E --> F{base_addr类型}
    F --> |nav/uav/int| G[返回.runtime_store]
    F --> |comptime_field| H[返回.comptime_field]
    F --> |comptime_alloc| I[构造direct策略]
    F --> |eu_payload| J[递归处理基指针]
    F --> |opt_payload| K[递归处理基指针]
    F --> |arr_elem| L[处理数组元素偏移]
    F --> |field| M[处理结构体/联合体字段]
    
    J --> N{处理错误联合结果}
    N --> |错误| O[返回错误]
    N --> |成功| P[处理eu_payload结果]
    P --> Q[返回direct策略]
    
    K --> R{处理可选类型结果}
    R --> |错误| S[返回错误]
    R --> |成功| T[处理opt_payload结果]
    T --> U[返回direct策略]
    
    L --> V[检查数组类型兼容性]
    V --> W{类型是否匹配}
    W --> |是| X[递归处理基指针]
    W --> |否| Y[设置存储数量为1]
    X --> Z[返回数组索引策略]
    
    M --> AA[递归处理基指针]
    AA --> AB{聚合类型判断}
    AB --> |struct/pointer| AC[返回字段direct策略]
    AB --> |union| AD[检查联合标签匹配]
    AD --> AE{标签是否匹配}
    AE --> |是| AF[返回联合payload策略]
    AE --> |否| AG[返回.inactive_union_field]
    
    I --> AH[构造direct策略返回]
    
    subgraph 后续处理
        E --> AI[检查字节偏移]
        AI --> AJ{偏移是否为0}
        AJ --> |是| AK[检查类型兼容性]
        AJ --> |否| AL[处理数组重结构化]
        
        AK --> AM{类型是否兼容}
        AM --> |是| AN[返回基策略]
        AM --> |否| AL
        
        AL --> AO[计算元素偏移]
        AO --> AP{是否越界}
        AP --> |是| AQ[返回.out_of_bounds]
        AP --> |否| AR[构造flat_index/index策略]
        
        AR --> AS[返回存储策略]
    end
    
    subgraph 内存重新解释
        E --> AT[检查布局合法性]
        AT --> AU{存储类型布局是否合法}
        AU --> |否| AV[返回.needed_well_defined]
        AU --> |是| AW[计算内存区域]
        AW --> AX{是否越界}
        AX --> |是| AY[返回.out_of_bounds]
        AX --> |否| AZ[类型细化处理]
        
        AZ --> BA{类型分类处理}
        BA --> |基本类型| BB[直接存储]
        BA --> |数组/结构体/联合体| BC[递归解析内存布局]
        BC --> BD[最终确定存储策略]
    end
    
    BD --> BE[返回reinterpret策略]
    BB --> BF[返回direct策略]
