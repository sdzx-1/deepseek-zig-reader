graph TD
    Start[开始] --> A[解包索引unwrapped_index]
    A --> B[获取extra_list和extra_items]
    B --> C{检查item.tag}
    
    C -->|type_struct| D[处理type_struct分支]
    D --> D1[获取name, namespace, zir_index]
    D1 --> D2[计算flags和extra_index]
    D2 --> D3{是否有any_captures?}
    D3 -->|是| D4[获取captures_len并更新extra_index]
    D3 -->|否| D5[跳过captures_len]
    D4/D5 --> D6{是否is_reified?}
    D6 -->|是| D7[跳过type_hash字段]
    D6 -->|否| D8[无需跳过]
    D7/D8 --> D9[初始化field_types]
    D9 --> D10[处理names_map和names]
    D10 --> D11{是否有any_default_inits?}
    D11 -->|是| D12[初始化inits]
    D11 -->|否| D13[跳过inits]
    D12/D13 --> D14{是否有any_aligned_fields?}
    D14 -->|是| D15[初始化aligns]
    D14 -->|否| D16[跳过aligns]
    D15/D16 --> D17{是否有any_comptime_fields?}
    D17 -->|是| D18[初始化comptime_bits]
    D17 -->|否| D19[跳过comptime_bits]
    D18/D19 --> D20{是否是extern?}
    D20 -->|否| D21[初始化runtime_order]
    D20 -->|是| D22[跳过runtime_order]
    D21/D22 --> D23[初始化offsets]
    D23 --> D24[返回LoadedStructType]

    C -->|type_struct_packed/_inits| E[处理packed分支]
    E --> E1[获取name, zir_index, namespace]
    E1 --> E2[计算flags和extra_index]
    E2 --> E3{是否有any_captures?}
    E3 -->|是| E4[获取captures_len并更新extra_index]
    E3 -->|否| E5[跳过captures_len]
    E4/E5 --> E6{是否is_reified?}
    E6 -->|是| E7[跳过type_hash字段]
    E6 -->|否| E8[无需跳过]
    E7/E8 --> E9[初始化field_types]
    E9 --> E10[初始化field_names]
    E10 --> E11{是否有inits?}
    E11 -->|是| E12[初始化field_inits]
    E11 -->|否| E13[跳过field_inits]
    E12/E13 --> E14[返回LoadedStructType]

    C -->|其他| F[触发unreachable]
    D24 & E14 --> End[结束]
    F --> End
