graph TD
    A[开始] --> B{检查self.lazy_syms.getPtr(.anyerror_type)}
    B -->|存在| C[激活PerThread (pt)]
    C --> D1{metadata.text_state != .unused?}
    D1 -->|是| E1[调用updateLazySymbol (text)]
    D1 -->|否| D2{metadata.rodata_state != .unused?}
    D2 -->|是| E2[调用updateLazySymbol (rodata)]
    E1 --> D2
    E2 --> F[deactivate pt]
    B -->|不存在| G[遍历所有lazy_syms]
    G --> H1[设置text_state为.flushed]
    G --> H2[设置rodata_state为.flushed]
    H1 --> H2
    H2 --> I{enable_logging?}
    I -->|是| J1[遍历navs检查checkNavAllocated]
    J1 --> J2[遍历uavs检查checkUavAllocated]
    I -->|否| K{存在self.dwarf?}
    J2 --> K
    K -->|是| L[激活PerThread (pt)]
    L --> M[调用dwarf.flushModule]
    M --> N[遍历调试节和符号]
    N --> O1[处理cross_unit_relocs]
    N --> O2[处理cross_section_relocs]
    N --> O3[处理entry内的重定位]
    O1 --> O2 --> O3 --> P[更新section_dirty标志]
    P --> Q[deactivate pt]
    K -->|否| R[断言检查section_dirty标志]
    Q --> R
    R --> S[结束]
