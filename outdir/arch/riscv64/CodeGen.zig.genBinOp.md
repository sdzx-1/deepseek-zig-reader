graph TD
    Start[开始] --> PromoteReg[提升寄存器 lhs_reg 和 rhs_reg, 锁定寄存器]
    PromoteReg --> DeferLock[延迟解锁寄存器]
    DeferLock --> SwitchTag{根据 tag 分支}
    
    SwitchTag --> |add/sub/mul等算术操作| Arithmetic[处理算术操作]
    Arithmetic --> CheckMathType{检查类型}
    
    CheckMathType --> |整数类型| IntType[选择整数指令]
    CheckMathType --> |浮点类型| FloatType[选择浮点指令]
    CheckMathType --> |向量类型| VectorType[设置VL寄存器，选择向量指令]
    
    IntType --> GenIntInst[生成整数指令]
    FloatType --> GenFloatInst[生成浮点指令]
    VectorType --> GenVectorInst[生成向量指令]
    
    SwitchTag --> |add_sat| AddSat[处理饱和加法]
    AddSat --> AddInst[生成 add 指令]
    AddInst --> SltuInst[生成 sltu 指令]
    SltuInst --> NegInst[生成 sub 指令取反]
    NegInst --> OrInst[生成 or 指令组合结果]
    
    SwitchTag --> |ptr_add/ptr_sub| PtrArith[指针算术]
    PtrArith --> CopySizeReg[复制元素大小到临时寄存器]
    CopySizeReg --> GenMul[生成乘法指令]
    GenMul --> GenAddSub[生成 add/sub 指令]
    
    SwitchTag --> |bit_and/or等| Bitwise[位操作/逻辑操作]
    Bitwise --> GenAndOr[生成 and/or 指令]
    Bitwise --> TruncateBool{是布尔操作?}
    TruncateBool --> |是| TruncateReg[截断寄存器到bool]
    
    SwitchTag --> |shl/shr等位移| Shift[处理位移]
    Shift --> TruncateRhs[截断右操作数]
    TruncateRhs --> GenShiftInst[生成位移指令]
    
    SwitchTag --> |cmp_*比较操作| Compare[生成伪比较指令]
    
    SwitchTag --> |min/max| MinMax[处理极值]
    MinMax --> GenSlt[生成 slt/sltu 指令]
    GenSlt --> GenSub[生成 sub 取反掩码]
    GenSub --> GenXor1[生成异或操作]
    GenXor1 --> GenAnd[生成与操作]
    GenAnd --> GenXor2[生成最终异或操作]
    
    SwitchTag --> |其他未实现操作| TodoFail[触发TODO错误]
    
    GenIntInst --> AfterArith[后续处理]
    GenFloatInst --> AfterArith
    GenVectorInst --> AfterArith
    OrInst --> AfterArith
    GenAddSub --> AfterArith
    TruncateReg --> AfterArith
    GenShiftInst --> AfterArith
    Compare --> AfterArith
    GenXor2 --> AfterArith
    
    AfterArith --> ReleaseLocks[释放寄存器锁]
    ReleaseLocks --> End[结束]
    
    TodoFail --> ReleaseLocks
