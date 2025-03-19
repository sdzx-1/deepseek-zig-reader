graph TD
    Start([Start]) --> ParsePrefixes[解析前缀 parsePrefixes]
    ParsePrefixes --> ErrorPrefixes{错误?}
    ErrorPrefixes -- 是 --> |EndOfStream| ReturnNull[返回 null]
    ErrorPrefixes -- 其他错误 --> ReturnError[返回错误]
    ErrorPrefixes -- 无错误 --> ParseEncoding[解析编码 parseEncoding]
    ParseEncoding --> CheckEnc{enc存在?}
    CheckEnc -- 否 --> ErrorUnknownOpcode[返回 UnknownOpcode]
    CheckEnc -- 是 --> SwitchOpEn{根据 enc.data.op_en 分支}
    
    SwitchOpEn -->|.z| Z[inst(enc, 空操作数)]
    SwitchOpEn -->|.o| O[解析寄存器低3位, 构造reg操作数]
    SwitchOpEn -->|.zo| ZO[op1固定reg, op2解析reg]
    SwitchOpEn -->|.oz| OZ[op1解析reg, op2固定reg]
    SwitchOpEn -->|.oi| OI[op1解析reg, op2解析立即数]
    SwitchOpEn -->|.i/.d| ID[op1解析立即数]
    SwitchOpEn -->|.zi| ZI[op1固定reg, op2解析立即数]
    SwitchOpEn -->|.ii| II[两个立即数]
    SwitchOpEn -->|.ia| IA[立即数 + eax寄存器]
    SwitchOpEn -->|.m/mi/m1/mc| M[解析ModRm和Sib, 处理内存操作数]
    SwitchOpEn -->|.fd| FD[段寄存器 + 偏移地址]
    SwitchOpEn -->|.td| TD[偏移地址 + 段寄存器]
    SwitchOpEn -->|.mr/mri/mrc| MR[解析ModRm, 处理寄存器到内存]
    SwitchOpEn -->|.rm/rmi| RM[解析ModRm, 处理内存到寄存器]
    SwitchOpEn -->|其他| Unimplemented[未实现分支]
    
    M --> ParseModRm[解析ModRm字节]
    ParseModRm --> CheckDirect{modrm.direct()?}
    CheckDirect -- 是 --> DirectCase[构造寄存器操作数]
    CheckDirect -- 否 --> ParseDisp[解析位移 disp]
    ParseDisp --> CheckRIP{modrm.rip()?}
    CheckRIP -- 是 --> RIPCase[构造RIP相对内存]
    CheckRIP -- 否 --> ParseSIB[解析SIB并构造内存操作数]
    
    MR --> ParseModRmMR[解析ModRm]
    ParseModRmMR --> CheckDirectMR{modrm.direct()?}
    CheckDirectMR -- 是 --> DirectReg[构造寄存器到寄存器]
    CheckDirectMR -- 否 --> ParseDispMR[解析位移并构造内存操作]
    
    RM --> ParseModRmRM[解析ModRm]
    ParseModRmRM --> CheckDirectRM{modrm.direct()?}
    CheckDirectRM -- 是 --> DirectRegRM[构造寄存器操作]
    CheckDirectRM -- 否 --> ParseDispRM[解析位移构造内存操作]
    
    Z --> ReturnInst[调用 inst 返回指令]
    O --> ReturnInst
    ZO --> ReturnInst
    OZ --> ReturnInst
    OI --> ReturnInst
    ID --> ReturnInst
    ZI --> ReturnInst
    II --> ReturnInst
    IA --> ReturnInst
    DirectCase --> ReturnInst
    RIPCase --> ReturnInst
    ParseSIB --> ReturnInst
    DirectReg --> ReturnInst
    ParseDispMR --> ReturnInst
    DirectRegRM --> ReturnInst
    ParseDispRM --> ReturnInst
    FD --> ReturnInst
    TD --> ReturnInst
    Unimplemented --> ReturnInst
    
    ReturnInst --> End([End])
    ReturnNull --> End
    ReturnError --> End
    ErrorUnknownOpcode --> End
