graph TD
    Start[开始 binOp] --> CheckTag{检查 tag}
    CheckTag --> |基本操作 add/sub/mul 等| CheckType{检查类型是否为 int}
    CheckType --> |是| CheckBits{检查 bits <=64}
    CheckBits --> |是| CheckImmediate{检查立即数}
    CheckImmediate --> |rhs 是立即数| BinOpImmediate[调用 binOpImmediate]
    CheckImmediate --> |lhs 是立即数且可交换| SwapOperands[交换操作数并调用 binOpImmediate]
    CheckImmediate --> |都不是| BinOpRegister[调用 binOpRegister]
    CheckBits --> |否| FailBigInt[返回错误: bits >64]
    CheckType --> |否| FailUnsupportedType[返回错误: 不支持的类型]

    CheckTag --> |add_wrap/sub_wrap/mul_wrap| GenBaseOp[生成基本操作]
    GenBaseOp --> TruncateResult[截断结果]
    TruncateResult --> ReturnResult[返回结果]

    CheckTag --> |div_trunc| CheckDivType{检查类型是否为 int}
    CheckDivType --> |是| CheckDivBits{检查 bits <=64}
    CheckDivBits --> |是| CheckDivImmediate{检查 rhs 是立即数}
    CheckDivImmediate --> |是| DivImmediate[调用 binOpImmediate]
    CheckDivImmediate --> |否| DivRegister[调用 binOpRegister]
    CheckDivBits --> |否| FailDivBigInt[返回错误: bits >64]

    CheckTag --> |ptr_add| CheckPtrType{检查类型是否为指针}
    CheckPtrType --> |是| CheckElemSize{计算元素大小}
    CheckElemSize --> |elem_size=1| PtrAddDirect[直接调用 binOpRegister]
    CheckElemSize --> |elem_size≠1| MulOffset[计算偏移量后调用 binOp]

    CheckTag --> |bool_and/bool_or| CheckBoolType{检查类型为 bool}
    CheckBoolType --> |是| BoolOpRegister[调用 binOpRegister]

    CheckTag --> |shl/shr| GenShiftBase[生成基本移位操作]
    GenShiftBase --> TruncateShift[截断结果并返回]

    CheckTag --> |shl_exact/shr_exact| CheckShiftType{检查类型是否为 int}
    CheckShiftType --> |是| CheckShiftBits{检查 bits <=64}
    CheckShiftBits --> |是| CheckShiftImmediate{检查 rhs 是立即数}
    CheckShiftImmediate --> |是| ShiftImmediate[调用 binOpImmediate]
    CheckShiftImmediate --> |否| ShiftRegister[调用 binOpRegister]
    CheckShiftBits --> |否| FailShiftBigInt[返回错误: bits >64]

    CheckTag --> |其他未实现 tag| FailUnsupportedOp[返回错误: 未实现的操作]

    BinOpImmediate --> End[返回结果]
    SwapOperands --> End
    BinOpRegister --> End
    PtrAddDirect --> End
    MulOffset --> End
    BoolOpRegister --> End
    TruncateShift --> End
    DivImmediate --> End
    DivRegister --> End
    ShiftImmediate --> End
    ShiftRegister --> End
    ReturnResult --> End

    FailBigInt --> EndError[返回错误]
    FailUnsupportedType --> EndError
    FailDivBigInt --> EndError
    FailShiftBigInt --> EndError
    FailUnsupportedOp --> EndError
