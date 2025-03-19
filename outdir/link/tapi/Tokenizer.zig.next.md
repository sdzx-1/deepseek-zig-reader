graph TD
    A[开始] --> B[初始化Token和状态]
    B --> C{还有字符未处理?}
    C --> |是| D[获取当前字符c]
    D --> E[根据当前状态处理字符]
    E -->|状态: start| F[检查c的类型]
    F -->|空格| G[切换到space状态]
    F -->|\t| H[切换到tab状态]
    F -->|\n| I[设置new_line Token并结束]
    F -->|\r| J[切换到new_line状态]
    F -->|-| K[检查模式匹配]
    K -->|匹配---| L[设置doc_start Token并结束]
    K -->|匹配- | M[设置seq_item_ind Token并结束]
    K -->|不匹配| N[切换到literal状态]
    F -->|.| O[检查...模式]
    O -->|匹配...| P[设置doc_end Token并结束]
    O -->|不匹配| Q[切换到literal状态]
    F -->|特殊符号如#,*,&等| R[设置对应Token并结束]
    F -->|'或"| S[切换到引号对应状态]
    F -->|其他字符| T[切换到literal状态]
    
    E -->|状态: comment| U[检查换行符]
    U -->|遇到换行符| V[设置comment Token并结束]
    
    E -->|状态: space/tab| W[持续匹配相同空白符]
    W -->|遇到不同字符| X[设置space/tab Token并结束]
    
    E -->|状态: new_line| Y[检查\n]
    Y -->|遇到\n| Z[设置new_line Token并结束]
    
    E -->|状态: single_quoted| AA[检查结束引号]
    AA -->|遇到非转义'| AB[设置single_quoted Token并结束]
    
    E -->|状态: double_quoted| AC[检查结束引号]
    AC -->|遇到非转义"| AD[设置double_quoted Token并结束]
    
    E -->|状态: literal| AE[检查分隔符]
    AE -->|遇到分隔符| AF[设置literal Token并结束]
    
    C --> |否| AG[处理结束状态]
    AG -->|状态是literal| AH[设置literal Token]
    AG -->|其他状态| AI[保持默认eof]
    AG --> AJ[记录日志并返回Token]
