flowchart TD
    A[开始] --> B[获取air_tags和air_datas]
    B --> C{switch(air_tags[inst])}
    
    %% 基本二元操作分支
    C --> |add/sub/mul等二元操作| D[检查operand是否为lhs或rhs]
    D -->|是lhs| E[返回matchOperandSmallIndex(0, .none)]
    D -->|是rhs| F[返回matchOperandSmallIndex(1, .none)]
    D -->|都不是| G[返回.none]
    
    %% 存储类操作分支
    C --> |store/atomic_store等| H[检查operand是否为lhs或rhs]
    H -->|是lhs| I[返回matchOperandSmallIndex(0, .write)]
    H -->|是rhs| J[返回matchOperandSmallIndex(1, .write)]
    H -->|都不是| K[返回.write]
    
    %% 直接返回.none的特殊指令
    C --> |arg/alloc/unreach等| L[直接返回.none]
    
    %% 单操作数指令分支
    C --> |not/bitcast/load等| M[检查operand是否是唯一操作数]
    M -->|是| N[返回matchOperandSmallIndex(0, .none)]
    M -->|否| O[返回.none]
    
    %% 函数调用处理分支
    C --> |call系列指令| P[处理参数列表]
    P -->|小参数列表| Q[逐个检查参数位置]
    P -->|大参数列表| R[迭代BigTomb检查]
    Q & R --> S[返回.write或.tomb]
    
    %% 复杂控制流分支
    C --> |block/loop/cond_br等| T[返回.complex]
    
    %% 原子操作分支
    C --> |cmpxchg/atomic_rmw| U[检查ptr/expected/new_value]
    U -->|匹配操作数| V[返回对应.write]
    U -->|都不匹配| K
    
    %% 向量/聚合类型处理
    C --> |aggregate_init/shuffle| W[遍历元素检查]
    W -->|小元素列表| X[逐个匹配索引]
    W -->|大元素列表| Y[BigTomb迭代]
    X & Y --> Z[返回.write或.tomb]
    
    %% 默认结束节点
    C --> |其他未明确分支| AA[返回对应类别]
    E & F & G & I & J & K & L & N & O & S & T & V & Z & AA --> AB[结束]
