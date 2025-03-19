flowchart TD
    Start([Start]) --> CheckMCV{Check mcv type}

    %% Dead case
    CheckMCV --> |.dead| Dead[unreachable]

    %% Unreach or None case
    CheckMCV --> |.unreach/.none| ReturnNothing[Return]

    %% Undef case
    CheckMCV --> |.undef| UndefCheckSafety{self.wantSafety()?}
    UndefCheckSafety --> |No| ReturnUndef[Return]
    UndefCheckSafety --> |Yes| CheckAbiSizeUndef{abi_size?}
    CheckAbiSizeUndef --> |1| Recursive1[Recursive call with 0xaa]
    CheckAbiSizeUndef --> |2| Recursive2[Recursive call with 0xaaaa]
    CheckAbiSizeUndef --> |4| Recursive4[Recursive call with 0xaaaaaaaa]
    CheckAbiSizeUndef --> |else| InlineMemset[Call genInlineMemset]

    %% CPSR/Immediate/PtrStackOffset case
    CheckMCV --> |.cpsr_flags<br>.immediate<br>.ptr_stack_offset| CopyToTmpReg[Copy to tmp register]
    CopyToTmpReg --> GenSetStackReg[Call genSetStack with register]

    %% Register case
    CheckMCV --> |.register| CheckAbiSizeReg{abi_size?}
    CheckAbiSizeReg --> |1 or 4| CreateOffset[Create offset]
    CreateOffset --> AddStrInst[Add strb/str instruction]
    CheckAbiSizeReg --> |2| CreateHOffset[Create strh offset]
    CreateHOffset --> AddStrHInst[Add strh instruction]
    CheckAbiSizeReg --> |else| Fail[Return error]

    %% Register C/V Flag case
    CheckMCV --> |.register_c_flag<br>.register_v_flag| LockReg[Lock register]
    LockReg --> GenSetWrapped[Set wrapped type]
    GenSetWrapped --> GetOverflowBit[Get overflow bit]
    GetOverflowBit --> MovCond[Add conditional mov]
    MovCond --> GenSetOverflow[Set overflow bit]

    %% Memory/Stack cases
    CheckMCV --> |.memory/.stack_argument_offset<br>.stack_offset| CheckSelfCopy{Is self copy?}
    CheckSelfCopy --> |Yes| ReturnSelf[Return]
    CheckSelfCopy --> |No| CheckAbiSizeMem{abi_size ≤4?}
    CheckAbiSizeMem --> |Yes| CopyToTmpRegMem[Copy to tmp register]
    CopyToTmpRegMem --> GenSetStackRegMem[Call genSetStack with register]
    CheckAbiSizeMem --> |No| AllocRegs[Allocate 5 registers]
    AllocRegs --> SetupSrc[Setup source reg]
    SetupSrc --> SetupDst[Setup dest reg]
    SetupDst --> SetLen[Set length reg]
    SetLen --> Memcpy[Call genInlineMemcpy]

    %% End connections
    Dead --> End
    ReturnNothing --> End
    ReturnUndef --> End
    Recursive1 --> End
    Recursive2 --> End
    Recursive4 --> End
    InlineMemset --> End
    GenSetStackReg --> End
    AddStrInst --> End
    AddStrHInst --> End
    Fail --> End
    GenSetOverflow --> End
    ReturnSelf --> End
    GenSetStackRegMem --> End
    Memcpy --> End

    End([End])
