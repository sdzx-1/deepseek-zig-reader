flowchart TD
    A[Start: pointerDerivationAdvanced] --> B[Get ptr from ptr_val]
    B --> C{Switch ptr.base_addr}
    C --> |.int| D[Return int step]
    C --> |.nav| E[Return nav_ptr step]
    C --> |.uav| F[Create const_ty and return uav_ptr step]
    C --> |.comptime_alloc| G[Get alloc, val, ty and return comptime_alloc_ptr step]
    C --> |.comptime_field| H[Return comptime_field_ptr step]
    C --> |.eu_payload| I[Recursive call for eu_ptr parent, return eu_payload_ptr step]
    C --> |.opt_payload| J[Recursive call for opt_ptr parent, return opt_payload_ptr step]
    C --> |.field| K[Calculate field_ty/align, create parent step, return field_ptr step]
    C --> |.arr_elem| L[Recursive call for arr base, adjust ptr size, return elem_ptr step]
    
    K --> K1{Check agg_ty type}
    K1 --> |struct/union| K2[Get field offset/alignment]
    K1 --> |pointer| K3[Handle slice ptr/len fields]
    
    L --> L1[Create parent step]
    L1 --> L2[Adjust ptr size to .one]
    
    C --> M[Check byte_offset and type]
    M --> |0 offset + matching type| N[Return base_derive]
    M --> |Other| O[Analyze ptr_ty_info and need_child]
    
    O --> P{need_child.comptimeOnly?}
    P --> |Yes| Q[Create offset_and_cast step]
    P --> |No| R[Init cur_derive and cur_offset]
    
    R --> S{Refinement loop}
    S --> T[Check current type and offset]
    T --> |Match need_child| U[Break loop]
    T --> |Not match| V{Check cur_ty type}
    
    V --> |optional| W[Handle optional payload]
    V --> |array| X[Calculate element index/offset]
    V --> |struct| Y[Find matching field]
    V --> |other types| Z[Break loop]
    
    Y --> Y1{Check field boundaries}
    Y1 --> |Fit| Y2[Create field_ptr step]
    Y1 --> |Overlap| Z
    
    X --> X1[Calculate start_idx/end_idx]
    X1 --> |Single element| X2[Create elem_ptr step]
    X1 --> |Multiple elements| X3[Adjust for first element]
    
    S --> AA[Check final offset]
    AA --> |0 offset| AB{Check type compatibility}
    AB --> |Compatible| AC[Return cur_derive]
    AB --> |Incompatible| AD[Create offset_and_cast step]
    AA --> |Non-zero| AD
