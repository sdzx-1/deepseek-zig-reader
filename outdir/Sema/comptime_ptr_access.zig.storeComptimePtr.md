graph TD
    Start[开始] --> Init[初始化变量: pt, zcu, ptr_info]
    Init --> CheckType[检查store_val类型与ptr_info.child是否匹配]
    CheckType --> CalcHostBits[计算host_bits和bit_offset]
    CalcHostBits --> DeterminePseudoStoreTy[确定pseudo_store_ty]
    DeterminePseudoStoreTy --> PrepareStrat[调用prepareComptimePtrStore获取strat]
    
    PrepareStrat --> SwitchStrat{根据strat分支}
    
    SwitchStrat --> |strat=comptime_field| LoadValue[加载现有值expected_mv]
    LoadValue --> CompareValue[比较store_val与expected_mv]
    CompareValue --> |不匹配| ReturnMismatch[返回comptime_field_mismatch]
    CompareValue --> |匹配| ReturnSuccess1[返回success]
    
    SwitchStrat --> |strat=runtime_store| ReturnRuntimeStore[返回runtime_store]
    SwitchStrat --> |strat=undef| ReturnUndef[返回undef]
    SwitchStrat --> |strat=err_payload| ReturnErr[返回err_payload]
    SwitchStrat --> |strat=null_payload| ReturnNull[返回null_payload]
    SwitchStrat --> |strat=inactive_union_field| ReturnUnionField[返回inactive_union_field]
    SwitchStrat --> |strat=needed_well_defined| ReturnWellDefined[返回needed_well_defined]
    SwitchStrat --> |strat=out_of_bounds| ReturnOOB[返回out_of_bounds]
    
    SwitchStrat --> |strat=direct/index/flat_index/reinterpret| CheckRuntimeCondition[检查是否在runtime条件内]
    CheckRuntimeCondition --> CheckHostBits{host_bits==0?}
    
    CheckHostBits --> |是| HandleZeroHostBits[根据strat处理直接存储]
    HandleZeroHostBits --> |direct| CoerceDirect[类型强制并存储]
    HandleZeroHostBits --> |index| CoerceIndex[类型强制并设置元素]
    HandleZeroHostBits --> |flat_index| FlattenArray[展开数组并存储]
    HandleZeroHostBits --> |reinterpret| ProceedBitcast[进入位转换流程]
    
    CheckHostBits --> |否| ProceedBitcast
    ProceedBitcast --> CheckLayout[检查val_ptr和store_val的布局]
    CheckLayout --> |布局无效| ReturnLayoutError[返回needed_well_defined]
    CheckLayout --> |布局有效| PerformBitcast[执行位转换操作]
    PerformBitcast --> UpdateValue[更新val_ptr的值]
    UpdateValue --> ReturnSuccess2[返回success]
    
    Start -.-> |错误路径| ErrorHandling[错误处理节点]
    ErrorHandling --> |exceeds_host_size| ReturnExceeds[返回exceeds_host_size]
    ErrorHandling --> |其他错误| ReturnOtherErrors[对应错误返回]
    
    ReturnMismatch --> End[结束]
    ReturnSuccess1 --> End
    ReturnRuntimeStore --> End
    ReturnUndef --> End
    ReturnErr --> End
    ReturnNull --> End
    ReturnUnionField --> End
    ReturnWellDefined --> End
    ReturnOOB --> End
    ReturnExceeds --> End
    ReturnOtherErrors --> End
    ReturnSuccess2 --> End
