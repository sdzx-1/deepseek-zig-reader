graph TD
    A[Start: indexToKey] --> B[Assert index != .none]
    B --> C[Unwrap index and get item/data]
    C --> D{Switch item.tag}
    
    D -->|.type_int_signed| E[Construct int_type signed]
    D -->|.type_int_unsigned| F[Construct int_type unsigned]
    D -->|.type_array_big| G[Get array_info and return array_type]
    D -->|.type_array_small| H[Get array_info and return array_type]
    D -->|.simple_type| I[Return simple_type from index]
    D -->|.simple_value| J[Return simple_value from index]
    D -->|.type_vector| K[Get vector_info and return vector_type]
    D -->|.type_pointer| L[Return ptr_type from extra data]
    D -->|.type_slice| M[Process many_ptr_index and return ptr_type]
    D -->|.type_optional| N[Return opt_type from data]
    D -->|.type_anyframe| O[Return anyframe_type from data]
    D -->|.type_error_union| P[Return error_union_type from extra data]
    D -->|.type_anyerror_union| Q[Build error_union_type with anyerror_type]
    D -->|.type_error_set| R[Return error_set_type from extra data]
    D -->|.type_inferred_error_set| S[Return inferred_error_set_type]
    D -->|.type_opaque| T[Check captures_len and return opaque_type]
    D -->|.type_struct| U[Process struct_type (reified/declared)]
    D -->|.type_struct_packed| V[Process packed struct_type]
    D -->|.type_tuple| W[Return tuple_type from extra data]
    D -->|.type_union| X[Process union_type (reified/declared)]
    D -->|.type_enum_auto| Y[Process enum_auto (generated/reified/declared)]
    D -->|.type_enum_explicit| Z[Process enum_explicit]
    D -->|.type_function| AA[Return func_type from extra data]
    D -->|.undef| AB[Return undef from data]
    D -->|.opt_null| AC[Return opt with .none]
    D -->|.opt_payload| AD[Get extra data and return opt]
    D -->|.ptr_* variants| AE[Process various pointer types]
    D -->|.int_* variants| AF[Construct int values]
    D -->|.float_* variants| AG[Construct float values]
    D -->|.variable| AH[Return variable from extra data]
    D -->|.extern| AI[Process extern declaration]
    D -->|.func_* variants| AJ[Return func instances]
    D -->|.only_possible_value| AK[Handle special type cases]
    D -->|.bytes| AL[Return aggregate with bytes]
    D -->|.aggregate| AM[Process aggregate from extra data]
    D -->|.repeated| AN[Return repeated aggregate]
    D -->|.union_value| AO[Return union from extra data]
    D -->|.error_* variants| AP[Handle error types]
    D -->|.enum_literal| AQ[Return enum_literal]
    D -->|.enum_tag| AR[Return enum_tag from extra data]
    D -->|.memoized_call| AS[Process memoized_call data]

    E --> AZ[Return Key]
    F --> AZ
    G --> AZ
    H --> AZ
    I --> AZ
    J --> AZ
    K --> AZ
    L --> AZ
    M --> AZ
    N --> AZ
    O --> AZ
    P --> AZ
    Q --> AZ
    R --> AZ
    S --> AZ
    T --> AZ
    U --> AZ
    V --> AZ
    W --> AZ
    X --> AZ
    Y --> AZ
    Z --> AZ
    AA --> AZ
    AB --> AZ
    AC --> AZ
    AD --> AZ
    AE --> AZ
    AF --> AZ
    AG --> AZ
    AH --> AZ
    AI --> AZ
    AJ --> AZ
    AK --> AZ
    AL --> AZ
    AM --> AZ
    AN --> AZ
    AO --> AZ
    AP --> AZ
    AQ --> AZ
    AR --> AZ
    AS --> AZ

    AZ[Return constructed Key]
    classDef action fill:#f9f,stroke:#333;
    classDef return fill:#bbf,stroke:#333;
    class B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z,AA,AB,AC,AD,AE,AF,AG,AH,AI,AJ,AK,AL,AM,AN,AO,AP,AQ,AR,AS action;
    class AZ return;
