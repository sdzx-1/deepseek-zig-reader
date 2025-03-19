flowchart TD
    Start[("Start checkBody")] --> Loop[Loop through each inst in body]
    Loop --> GetData[Get tag and data for current inst]
    GetData --> Switch{Switch on tag}
    
    Switch -->|.inferred_alloc, .inferred_alloc_comptime| Unreachable[("unreachable")]
    
    Switch -->|.arg| CheckArgType[Check data.arg.ty type]
    CheckArgType -->|Fail| ReturnFalse[("Return false")]
    CheckArgType -->|Pass| ContinueLoop[Continue loop]
    
    Switch -->|Binary operations| CheckBinRefs[Check lhs and rhs refs]
    CheckBinRefs -->|Fail| ReturnFalse
    CheckBinRefs -->|Pass| ContinueLoop
    
    Switch -->|TyOp operations| CheckTyOp[Check ty and operand ref]
    CheckTyOp -->|Fail| ReturnFalse
    CheckTyOp -->|Pass| ContinueLoop
    
    Switch -->|Block/Loop/DbgInlineBlock| CheckSubBody[Check sub-body recursively]
    CheckSubBody -->|Fail| ReturnFalse
    CheckSubBody -->|Pass| ContinueLoop
    
    Switch -->|Aggregate_init| CheckAggregate[Check all elements]
    CheckAggregate -->|Fail| ReturnFalse
    CheckAggregate -->|Pass| ContinueLoop
    
    Switch -->|Assembly| CheckAsmIO[Check outputs/inputs]
    CheckAsmIO -->|Fail| ReturnFalse
    CheckAsmIO -->|Pass| ContinueLoop
    
    Switch -->|Other special cases| CheckSpecial[Special case checks]
    CheckSpecial -->|Fail| ReturnFalse
    CheckSpecial -->|Pass| ContinueLoop
    
    Switch -->|No action needed| ContinueLoop
    
    ContinueLoop --> Loop
    Loop -->|All inst processed| ReturnTrue[("Return true")]
    
    ReturnFalse --> End[("End")]
    ReturnTrue --> End
