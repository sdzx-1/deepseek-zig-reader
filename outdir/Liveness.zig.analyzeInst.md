flowchart TD
    A[Start: analyzeInst] --> B[Get ip, inst_tags, inst_datas]
    B --> C{Switch inst_tags}
    
    C -->|Binary Operations| D[Extract bin_op\nCall analyzeOperands]
    C -->|Unary Operations| E[Extract un_op/ty_op\nCall analyzeOperands]
    C -->|Function Call| F[Handle callee & args\nCheck args len]
    F -->|Small args| G[Use buffer\nCall analyzeOperands]
    F -->|Big args| H[Use AnalyzeBigOperands\nFeed args in reverse]
    C -->|Control Flow| I[Call specific handlers\nanalyzeInstBr/Repeat/Switch/etc]
    C -->|Aggregate Init| J[Check elements len\nBuffer or BigOperands]
    C -->|Memory Operations| K[Handle ptr/offset\nCall analyzeOperands]
    C -->|Special Cases| L[Handle traps/returns\nCall analyzeFuncEnd]
    C -->|Assembly| M[Process outputs/inputs\nBuffer or BigOperands]
    C -->|Conditional Branches| N[Call analyzeInstCondBr]
    
    D --> Z[End]
    E --> Z
    G --> Z
    H --> Z
    I --> Z
    J --> Z
    K --> Z
    L --> Z
    M --> Z
    N --> Z

    style A fill:#f9f,stroke:#333
    style Z fill:#bbf,stroke:#333
    classDef default stroke:#333,fill:#fff
