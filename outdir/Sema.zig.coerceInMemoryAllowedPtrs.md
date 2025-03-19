graph TD
    Start([开始]) --> CheckPtrSize[检查指针大小兼容性]
    CheckPtrSize --> |不兼容| ReturnPtrSize[返回ptr_size错误]
    CheckPtrSize --> |兼容| CheckConst[检查常量性]
    
    CheckConst --> |不兼容| ReturnPtrConst[返回ptr_const错误]
    CheckConst --> |兼容| CheckVolatile[检查易失性]
    
    CheckVolatile --> |不兼容| ReturnPtrVolatile[返回ptr_volatile错误]
    CheckVolatile --> |兼容| CheckAllowZero[检查允许零指针]
    
    CheckAllowZero --> |不兼容| ReturnPtrAllowZero[返回ptr_allowzero错误]
    CheckAllowZero --> |兼容| CheckAddrSpace[检查地址空间]
    
    CheckAddrSpace --> |不匹配| ReturnPtrAddrSpace[返回ptr_addrspace错误]
    CheckAddrSpace --> |匹配| CheckChildType[递归检查子类型]
    
    CheckChildType --> |失败且非特殊情况| ReturnPtrChild[返回ptr_child错误]
    CheckChildType --> |成功或特殊情况| CheckBitRange[检查位范围]
    
    CheckBitRange --> |不匹配| ReturnPtrBitRange[返回ptr_bit_range错误]
    CheckBitRange --> |匹配| CheckSentinel[检查哨兵兼容性]
    
    CheckSentinel --> |不兼容| ReturnPtrSentinel[返回ptr_sentinel错误]
    CheckSentinel --> |兼容| CheckAlignment[检查对齐方式]
    
    CheckAlignment --> |不满足条件| ReturnPtrAlignment[返回ptr_alignment错误]
    CheckAlignment --> |满足条件| ReturnOK[返回.ok]
    
    ReturnPtrSize --> End([结束])
    ReturnPtrConst --> End
    ReturnPtrVolatile --> End
    ReturnPtrAllowZero --> End
    ReturnPtrAddrSpace --> End
    ReturnPtrChild --> End
    ReturnPtrBitRange --> End
    ReturnPtrSentinel --> End
    ReturnPtrAlignment --> End
    ReturnOK --> End
