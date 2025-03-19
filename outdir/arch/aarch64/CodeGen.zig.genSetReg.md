flowchart TD
    Start([Start genSetReg]) --> CheckMCV{Check mcv type}
    
    CheckMCV --> |.dead| Unreachable[unreachable]
    CheckMCV --> |.unreach, .none| ReturnNothing[Return]
    CheckMCV --> |.undef| CheckRegSize{Check register size}
    CheckMCV --> |.ptr_stack_offset| LdrPtrStack[Add ldr_ptr_stack instruction]
    CheckMCV --> |.compare_flags| CSet[Add cset instruction]
    CheckMCV --> |.immediate| ProcessImmediate[Add movz instruction]
    CheckMCV --> |.register| CheckSameReg{src_reg == reg?}
    CheckMCV --> |.register_with_overflow| Unreachable2[unreachable]
    CheckMCV --> |.linker_load| ProcessLinkerLoad[Handle linker load]
    CheckMCV --> |.memory| MemoryHandling[genSetReg + genLdrRegister]
    CheckMCV --> |.stack_offset| HandleStackOffset[Handle stack offset]
    CheckMCV --> |.stack_argument_offset| HandleStackArgOffset[Handle stack argument offset]
    
    CheckRegSize --> |32| Load32[Add immediate 0xaaaaaaaa]
    CheckRegSize --> |64| Load64[Add immediate 0xaaaaaaaaaaaaaaaa]
    
    ProcessImmediate --> CheckBits1{Check x & 0xffff0000}
    CheckBits1 --> |Non-zero| AddMovk16[Add movk hw=1]
    CheckBits1 --> |Zero| CheckRegSize64{reg.size=64?}
    
    CheckRegSize64 --> |Yes| CheckBits2{Check x & 0xffff00000000}
    CheckBits2 --> |Non-zero| AddMovk32[Add movk hw=2]
    CheckBits2 --> |Zero| CheckBits3{Check x & 0xffff000000000000}
    CheckBits3 --> |Non-zero| AddMovk48[Add movk hw=3]
    
    CheckSameReg --> |Yes| ReturnNothing2[Return]
    CheckSameReg --> |No| AddMovReg[Add mov_register instruction]
    
    ProcessLinkerLoad --> HandleAtomIndex[Get atom index]
    HandleAtomIndex --> AddLoadMemory[Add load_memory instruction]
    
    MemoryHandling --> GenSetRegX[genSetReg toX register]
    GenSetRegX --> GenLdrRegister[genLdrRegister]
    
    HandleStackOffset --> CheckAbiSize1{Check abi_size}
    CheckAbiSize1 --> |1/2/4/8| AddLdrStack[Add ldr_stack variant]
    CheckAbiSize1 --> |3/5/6/7| Fail1[Fail with TODO]
    
    HandleStackArgOffset --> CheckAbiSize2{Check abi_size}
    CheckAbiSize2 --> |1/2/4/8| AddLdrStackArg[Add ldr_stack_argument variant]
    CheckAbiSize2 --> |3/5/6/7| Fail2[Fail with TODO]
    
    AddMovk16 --> CheckRegSize64
    AddMovk32 --> CheckBits3
    AddMovk48 --> EndImmediate[End immediate handling]
    
    LdrPtrStack --> End
    CSet --> End
    Load32 --> End
    Load64 --> End
    AddMovReg --> End
    AddLoadMemory --> End
    GenLdrRegister --> End
    AddLdrStack --> End
    AddLdrStackArg --> End
    Fail1 --> End
    Fail2 --> End
    
    End([End of function])
