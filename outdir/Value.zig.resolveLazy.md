graph TD
    Start[开始] --> Switch[switch val的类型]
    Switch --> |int| IntCase
    Switch --> |slice| SliceCase
    Switch --> |ptr| PtrCase
    Switch --> |aggregate| AggregateCase
    Switch --> |un| UnionCase
    Switch --> |else| ReturnVal[返回val]
    
    IntCase --> IntSwitch[int.storage类型]
    IntSwitch --> |u64/i64/big_int| ReturnInt[返回val]
    IntSwitch --> |lazy_align/lazy_size| CalcInt[调用pt.intValue返回新值]
    
    SliceCase --> ResolvePtr[递归解析ptr]
    SliceCase --> ResolveLen[递归解析len]
    ResolvePtr --> CheckPtr[检查ptr是否变化]
    ResolveLen --> CheckLen[检查len是否变化]
    CheckPtr & CheckLen --> |未变化| ReturnSlice[返回val]
    CheckPtr & CheckLen --> |变化| NewSlice[构造新slice并返回]
    
    PtrCase --> PtrSwitch[ptr.base_addr类型]
    PtrSwitch --> |nav/comptime_alloc/uav/int| ReturnPtr[返回val]
    PtrSwitch --> |comptime_field| ResolveComptimeField[递归解析field_val]
    PtrSwitch --> |eu_payload/opt_payload| ResolvePayload[递归解析base]
    PtrSwitch --> |arr_elem/field| ResolveIndexed[递归解析base_index]
    ResolveComptimeField --> |未变化| ReturnPtr
    ResolveComptimeField --> |变化| NewComptimePtr[构造新ptr并返回]
    ResolvePayload --> |未变化| ReturnPtr
    ResolvePayload --> |变化| NewPayloadPtr[构造新ptr并返回]
    ResolveIndexed --> |未变化| ReturnPtr
    ResolveIndexed --> |变化| NewIndexedPtr[构造新ptr并返回]
    
    AggregateCase --> AggregateSwitch[aggregate.storage类型]
    AggregateSwitch --> |bytes| ReturnAggregate[返回val]
    AggregateSwitch --> |elems| ResolveElems[遍历解析每个元素]
    AggregateSwitch --> |repeated_elem| ResolveRepeated[解析重复元素]
    ResolveElems --> |元素未变化| ReturnAggregate
    ResolveElems --> |元素变化| NewElemsAggregate[构造新aggregate并返回]
    ResolveRepeated --> |元素未变化| ReturnAggregate
    ResolveRepeated --> |元素变化| NewRepeatedAggregate[构造新aggregate并返回]
    
    UnionCase --> ResolveTag[解析tag]
    UnionCase --> ResolveVal[解析val]
    ResolveTag --> |tag未变化且val未变化| ReturnUnion[返回val]
    ResolveTag --> |tag或val变化| NewUnion[构造新union并返回]
