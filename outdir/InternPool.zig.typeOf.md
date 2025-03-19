flowchart TD
    A([Start]) --> B{index is predefined type?}
    B -- Yes --> C[Return .type_type]
    B -- No --> D{index is specific value?}
    D -- Yes --> E[Return corresponding type]
    D -- No --> F[Unwrap index to get item]
    F --> G{Check item.tag}
    G --> |.type_int_signed, .type_int_unsigned, etc.| H[Return .type_type]
    G --> |.undef, .opt_null, etc.| I[Return item.data as Index]
    G --> |.ptr_nav, .ptr_comptime_alloc, etc.| J[Get extra data and return type]
    G --> |.int_u8, .int_u16, etc.| K[Return corresponding numeric type]
    G --> |.int_positive, .int_negative| L[Extract from limbs and return int.ty]
    G --> |.enum_literal| M[Return .enum_literal_type]
    G --> |.float_f16, .float_f32, etc.| N[Return corresponding float type]
    G --> |.float_comptime_float| O[Return .comptime_float_type]
    G --> |.memoized_call| P[Unreachable]
    C --> Q([End])
    E --> Q
    H --> Q
    I --> Q
    J --> Q
    K --> Q
    L --> Q
    M --> Q
    N --> Q
    O --> Q
    P --> Q
