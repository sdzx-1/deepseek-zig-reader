```zig
pub const Tag = enum(u8) {
    /// This special tag represents a value which was removed from this pool via
    /// `InternPool.remove`. The item remains allocated to preserve indices, but
    /// lookups will consider it not equal to any other item, and all queries
    /// assert not this tag. `data` is unused.
    removed,

    /// An integer type.
    /// data is number of bits
    type_int_signed,
    /// An integer type.
    /// data is number of bits
    type_int_unsigned,
    /// An array type whose length requires 64 bits or which has a sentinel.
    /// data is payload to Array.
    type_array_big,
    /// An array type that has no sentinel and whose length fits in 32 bits.
    /// data is payload to Vector.
    type_array_small,
    /// A vector type.
    /// data is payload to Vector.
    type_vector,
    /// A fully explicitly specified pointer type.
    type_pointer,
    /// A slice type.
    /// data is Index of underlying pointer type.
    type_slice,
    /// An optional type.
    /// data is the child type.
    type_optional,
    /// The type `anyframe->T`.
    /// data is the child type.
    /// If the child type is `none`, the type is `anyframe`.
    type_anyframe,
    /// An error union type.
    /// data is payload to `Key.ErrorUnionType`.
    type_error_union,
    /// An error union type of the form `anyerror!T`.
    /// data is `Index` of payload type.
    type_anyerror_union,
    /// An error set type.
    /// data is payload to `ErrorSet`.
    type_error_set,
    /// The inferred error set type of a function.
    /// data is `Index` of a `func_decl` or `func_instance`.
    type_inferred_error_set,
    /// An enum type with auto-numbered tag values.
    /// The enum is exhaustive.
    /// data is payload index to `EnumAuto`.
    type_enum_auto,
    /// An enum type with an explicitly provided integer tag type.
    /// The enum is exhaustive.
    /// data is payload index to `EnumExplicit`.
    type_enum_explicit,
    /// An enum type with an explicitly provided integer tag type.
    /// The enum is non-exhaustive.
    /// data is payload index to `EnumExplicit`.
    type_enum_nonexhaustive,
    /// A type that can be represented with only an enum tag.
    simple_type,
    /// An opaque type.
    /// data is index of Tag.TypeOpaque in extra.
    type_opaque,
    /// A non-packed struct type.
    /// data is 0 or extra index of `TypeStruct`.
    type_struct,
    /// A packed struct, no fields have any init values.
    /// data is extra index of `TypeStructPacked`.
    type_struct_packed,
    /// A packed struct, one or more fields have init values.
    /// data is extra index of `TypeStructPacked`.
    type_struct_packed_inits,
    /// A `TupleType`.
    /// data is extra index of `TypeTuple`.
    type_tuple,
    /// A union type.
    /// `data` is extra index of `TypeUnion`.
    type_union,
    /// A function body type.
    /// `data` is extra index to `TypeFunction`.
    type_function,

    /// Typed `undefined`.
    /// `data` is `Index` of the type.
    /// Untyped `undefined` is stored instead via `simple_value`.
    undef,
    /// A value that can be represented with only an enum tag.
    simple_value,
    /// A pointer to a `Nav`.
    /// data is extra index of `PtrNav`, which contains the type and address.
    ptr_nav,
    /// A pointer to a decl that can be mutated at comptime.
    /// data is extra index of `PtrComptimeAlloc`, which contains the type and address.
    ptr_comptime_alloc,
    /// A pointer to an anonymous addressable value.
    /// data is extra index of `PtrUav`, which contains the pointer type and decl value.
    /// The alignment of the uav is communicated via the pointer type.
    ptr_uav,
    /// A pointer to an unnamed addressable value.
    /// data is extra index of `PtrUavAligned`, which contains the pointer
    /// type and decl value.
    /// The original pointer type is also provided, which will be different than `ty`.
    /// This encoding is only used when a pointer to a Uav is
    /// coerced to a different pointer type with a different alignment.
    ptr_uav_aligned,
    /// data is extra index of `PtrComptimeField`, which contains the pointer type and field value.
    ptr_comptime_field,
    /// A pointer with an integer value.
    /// data is extra index of `PtrInt`, which contains the type and address (byte offset from 0).
    /// Only pointer types are allowed to have this encoding. Optional types must use
    /// `opt_payload` or `opt_null`.
    ptr_int,
    /// A pointer to the payload of an error union.
    /// data is extra index of `PtrBase`, which contains the type and base pointer.
    ptr_eu_payload,
    /// A pointer to the payload of an optional.
    /// data is extra index of `PtrBase`, which contains the type and base pointer.
    ptr_opt_payload,
    /// A pointer to an array element.
    /// data is extra index of PtrBaseIndex, which contains the base array and element index.
    /// In order to use this encoding, one must ensure that the `InternPool`
    /// already contains the elem pointer type corresponding to this payload.
    ptr_elem,
    /// A pointer to a container field.
    /// data is extra index of PtrBaseIndex, which contains the base container and field index.
    ptr_field,
    /// A slice.
    /// data is extra index of PtrSlice, which contains the ptr and len values
    ptr_slice,
    /// An optional value that is non-null.
    /// data is extra index of `TypeValue`.
    /// The type is the optional type (not the payload type).
    opt_payload,
    /// An optional value that is null.
    /// data is Index of the optional type.
    opt_null,
    /// Type: u8
    /// data is integer value
    int_u8,
    /// Type: u16
    /// data is integer value
    int_u16,
    /// Type: u32
    /// data is integer value
    int_u32,
    /// Type: i32
    /// data is integer value bitcasted to u32.
    int_i32,
    /// A usize that fits in 32 bits.
    /// data is integer value.
    int_usize,
    /// A comptime_int that fits in a u32.
    /// data is integer value.
    int_comptime_int_u32,
    /// A comptime_int that fits in an i32.
    /// data is integer value bitcasted to u32.
    int_comptime_int_i32,
    /// An integer value that fits in 32 bits with an explicitly provided type.
    /// data is extra index of `IntSmall`.
    int_small,
    /// A positive integer value.
    /// data is a limbs index to `Int`.
    int_positive,
    /// A negative integer value.
    /// data is a limbs index to `Int`.
    int_negative,
    /// The ABI alignment of a lazy type.
    /// data is extra index of `IntLazy`.
    int_lazy_align,
    /// The ABI size of a lazy type.
    /// data is extra index of `IntLazy`.
    int_lazy_size,
    /// An error value.
    /// data is extra index of `Key.Error`.
    error_set_error,
    /// An error union error.
    /// data is extra index of `Key.Error`.
    error_union_error,
    /// An error union payload.
    /// data is extra index of `TypeValue`.
    error_union_payload,
    /// An enum literal value.
    /// data is `NullTerminatedString` of the error name.
    enum_literal,
    /// An enum tag value.
    /// data is extra index of `EnumTag`.
    enum_tag,
    /// An f16 value.
    /// data is float value bitcasted to u16 and zero-extended.
    float_f16,
    /// An f32 value.
    /// data is float value bitcasted to u32.
    float_f32,
    /// An f64 value.
    /// data is extra index to Float64.
    float_f64,
    /// An f80 value.
    /// data is extra index to Float80.
    float_f80,
    /// An f128 value.
    /// data is extra index to Float128.
    float_f128,
    /// A c_longdouble value of 80 bits.
    /// data is extra index to Float80.
    /// This is used when a c_longdouble value is provided as an f80, because f80 has unnormalized
    /// values which cannot be losslessly represented as f128. It should only be used when the type
    /// underlying c_longdouble for the target is 80 bits.
    float_c_longdouble_f80,
    /// A c_longdouble value of 128 bits.
    /// data is extra index to Float128.
    /// This is used when a c_longdouble value is provided as any type other than an f80, since all
    /// other float types can be losslessly converted to and from f128.
    float_c_longdouble_f128,
    /// A comptime_float value.
    /// data is extra index to Float128.
    float_comptime_float,
    /// A global variable.
    /// data is extra index to Variable.
    variable,
    /// An extern function or variable.
    /// data is extra index to Extern.
    /// Some parts of the key are stored in `owner_nav`.
    @"extern",
    /// A non-extern function corresponding directly to the AST node from whence it originated.
    /// data is extra index to `FuncDecl`.
    /// Only the owner Decl is used for hashing and equality because the other
    /// fields can get patched up during incremental compilation.
    func_decl,
    /// A generic function instantiation.
    /// data is extra index to `FuncInstance`.
    func_instance,
    /// A `func_decl` or a `func_instance` that has been coerced to a different type.
    /// data is extra index to `FuncCoerced`.
    func_coerced,
    /// This represents the only possible value for *some* types which have
    /// only one possible value. Not all only-possible-values are encoded this way;
    /// for example structs which have all comptime fields are not encoded this way.
    /// The set of values that are encoded this way is:
    /// * An array or vector which has length 0.
    /// * A struct which has all fields comptime-known.
    /// * An empty enum or union. TODO: this value's existence is strange, because such a type in reality has no values. See #15909
    /// data is Index of the type, which is known to be zero bits at runtime.
    only_possible_value,
    /// data is extra index to Key.Union.
    union_value,
    /// An array of bytes.
    /// data is extra index to `Bytes`.
    bytes,
    /// An instance of a struct, array, or vector.
    /// data is extra index to `Aggregate`.
    aggregate,
    /// An instance of an array or vector with every element being the same value.
    /// data is extra index to `Repeated`.
    repeated,

    /// A memoized comptime function call result.
    /// data is extra index to `MemoizedCall`
    memoized_call,

    const ErrorUnionType = Key.ErrorUnionType;
    const TypeValue = Key.TypeValue;
    const Error = Key.Error;
    const EnumTag = Key.EnumTag;
    const Union = Key.Union;
    const TypePointer = Key.PtrType;

    const enum_explicit_encoding = .{
        .summary = .@"{.payload.name%summary#\"}",
        .payload = EnumExplicit,
        .trailing = struct {
            owner_union: Index,
            captures: ?[]CaptureValue,
            type_hash: ?u64,
            field_names: []NullTerminatedString,
            tag_values: []Index,
        },
        .config = .{
            .@"trailing.owner_union.?" = .@"payload.zir_index == .none",
            .@"trailing.cau.?" = .@"payload.zir_index != .none",
            .@"trailing.captures.?" = .@"payload.captures_len < 0xffffffff",
            .@"trailing.captures.?.len" = .@"payload.captures_len",
            .@"trailing.type_hash.?" = .@"payload.captures_len == 0xffffffff",
            .@"trailing.field_names.len" = .@"payload.fields_len",
            .@"trailing.tag_values.len" = .@"payload.fields_len",
        },
    };
    const encodings = .{
        .removed = .{},

        .type_int_signed = .{ .summary = .@"i{.data%value}", .data = u32 },
        .type_int_unsigned = .{ .summary = .@"u{.data%value}", .data = u32 },
        .type_array_big = .{
            .summary = .@"[{.payload.len1%value} << 32 | {.payload.len0%value}:{.payload.sentinel%summary}]{.payload.child%summary}",
            .payload = Array,
        },
        .type_array_small = .{ .summary = .@"[{.payload.len%value}]{.payload.child%summary}", .payload = Vector },
        .type_vector = .{ .summary = .@"@Vector({.payload.len%value}, {.payload.child%summary})", .payload = Vector },
        .type_pointer = .{ .summary = .@"*... {.payload.child%summary}", .payload = TypePointer },
        .type_slice = .{ .summary = .@"[]... {.data.unwrapped.payload.child%summary}", .data = Index },
        .type_optional = .{ .summary = .@"?{.data%summary}", .data = Index },
        .type_anyframe = .{ .summary = .@"anyframe->{.data%summary}", .data = Index },
        .type_error_union = .{
            .summary = .@"{.payload.error_set_type%summary}!{.payload.payload_type%summary}",
            .payload = ErrorUnionType,
        },
        .type_anyerror_union = .{ .summary = .@"anyerror!{.data%summary}", .data = Index },
        .type_error_set = .{ .summary = .@"error{...}", .payload = ErrorSet },
        .type_inferred_error_set = .{
            .summary = .@"@typeInfo(@typeInfo(@TypeOf({.data%summary})).@\"fn\".return_type.?).error_union.error_set",
            .data = Index,
        },
        .type_enum_auto = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = EnumAuto,
            .trailing = struct {
                owner_union: ?Index,
                captures: ?[]CaptureValue,
                type_hash: ?u64,
                field_names: []NullTerminatedString,
            },
            .config = .{
                .@"trailing.owner_union.?" = .@"payload.zir_index == .none",
                .@"trailing.cau.?" = .@"payload.zir_index != .none",
                .@"trailing.captures.?" = .@"payload.captures_len < 0xffffffff",
                .@"trailing.captures.?.len" = .@"payload.captures_len",
                .@"trailing.type_hash.?" = .@"payload.captures_len == 0xffffffff",
                .@"trailing.field_names.len" = .@"payload.fields_len",
            },
        },
        .type_enum_explicit = enum_explicit_encoding,
        .type_enum_nonexhaustive = enum_explicit_encoding,
        .simple_type = .{ .summary = .@"{.index%value#.}", .index = SimpleType },
        .type_opaque = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = TypeOpaque,
            .trailing = struct { captures: []CaptureValue },
            .config = .{ .@"trailing.captures.len" = .@"payload.captures_len" },
        },
        .type_struct = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = TypeStruct,
            .trailing = struct {
                captures_len: ?u32,
                captures: ?[]CaptureValue,
                type_hash: ?u64,
                field_types: []Index,
                field_names_map: OptionalMapIndex,
                field_names: []NullTerminatedString,
                field_inits: ?[]Index,
                field_aligns: ?[]Alignment,
                field_is_comptime_bits: ?[]u32,
                field_index: ?[]LoadedStructType.RuntimeOrder,
                field_offset: []u32,
            },
            .config = .{
                .@"trailing.captures_len.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?.len" = .@"trailing.captures_len.?",
                .@"trailing.type_hash.?" = .@"payload.flags.is_reified",
                .@"trailing.field_types.len" = .@"payload.fields_len",
                .@"trailing.field_names.len" = .@"payload.fields_len",
                .@"trailing.field_inits.?" = .@"payload.flags.any_default_inits",
                .@"trailing.field_inits.?.len" = .@"payload.fields_len",
                .@"trailing.field_aligns.?" = .@"payload.flags.any_aligned_fields",
                .@"trailing.field_aligns.?.len" = .@"payload.fields_len",
                .@"trailing.field_is_comptime_bits.?" = .@"payload.flags.any_comptime_fields",
                .@"trailing.field_is_comptime_bits.?.len" = .@"(payload.fields_len + 31) / 32",
                .@"trailing.field_index.?" = .@"!payload.flags.is_extern",
                .@"trailing.field_index.?.len" = .@"payload.fields_len",
                .@"trailing.field_offset.len" = .@"payload.fields_len",
            },
        },
        .type_struct_packed = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = TypeStructPacked,
            .trailing = struct {
                captures_len: ?u32,
                captures: ?[]CaptureValue,
                type_hash: ?u64,
                field_types: []Index,
                field_names: []NullTerminatedString,
            },
            .config = .{
                .@"trailing.captures_len.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?.len" = .@"trailing.captures_len.?",
                .@"trailing.type_hash.?" = .@"payload.is_flags.is_reified",
                .@"trailing.field_types.len" = .@"payload.fields_len",
                .@"trailing.field_names.len" = .@"payload.fields_len",
            },
        },
        .type_struct_packed_inits = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = TypeStructPacked,
            .trailing = struct {
                captures_len: ?u32,
                captures: ?[]CaptureValue,
                type_hash: ?u64,
                field_types: []Index,
                field_names: []NullTerminatedString,
                field_inits: []Index,
            },
            .config = .{
                .@"trailing.captures_len.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?.len" = .@"trailing.captures_len.?",
                .@"trailing.type_hash.?" = .@"payload.is_flags.is_reified",
                .@"trailing.field_types.len" = .@"payload.fields_len",
                .@"trailing.field_names.len" = .@"payload.fields_len",
                .@"trailing.field_inits.len" = .@"payload.fields_len",
            },
        },
        .type_tuple = .{
            .summary = .@"struct {...}",
            .payload = TypeTuple,
            .trailing = struct {
                field_types: []Index,
                field_values: []Index,
            },
            .config = .{
                .@"trailing.field_types.len" = .@"payload.fields_len",
                .@"trailing.field_values.len" = .@"payload.fields_len",
            },
        },
        .type_union = .{
            .summary = .@"{.payload.name%summary#\"}",
            .payload = TypeUnion,
            .trailing = struct {
                captures_len: ?u32,
                captures: ?[]CaptureValue,
                type_hash: ?u64,
                field_types: []Index,
                field_aligns: []Alignment,
            },
            .config = .{
                .@"trailing.captures_len.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?" = .@"payload.flags.any_captures",
                .@"trailing.captures.?.len" = .@"trailing.captures_len.?",
                .@"trailing.type_hash.?" = .@"payload.is_flags.is_reified",
                .@"trailing.field_types.len" = .@"payload.fields_len",
                .@"trailing.field_aligns.len" = .@"payload.fields_len",
            },
        },
        .type_function = .{
            .summary = .@"fn (...) ... {.payload.return_type%summary}",
            .payload = TypeFunction,
            .trailing = struct {
                param_comptime_bits: ?[]u32,
                param_noalias_bits: ?[]u32,
                param_type: []Index,
            },
            .config = .{
                .@"trailing.param_comptime_bits.?" = .@"payload.flags.has_comptime_bits",
                .@"trailing.param_comptime_bits.?.len" = .@"(payload.params_len + 31) / 32",
                .@"trailing.param_noalias_bits.?" = .@"payload.flags.has_noalias_bits",
                .@"trailing.param_noalias_bits.?.len" = .@"(payload.params_len + 31) / 32",
                .@"trailing.param_type.len" = .@"payload.params_len",
            },
        },

        .undef = .{ .summary = .@"@as({.data%summary}, undefined)", .data = Index },
        .simple_value = .{ .summary = .@"{.index%value#.}", .index = SimpleValue },
        .ptr_nav = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.nav.fqn%summary#\"}) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrNav,
        },
        .ptr_comptime_alloc = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&comptime_allocs[{.payload.index%summary}]) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrComptimeAlloc,
        },
        .ptr_uav = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.val%summary}) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrUav,
        },
        .ptr_uav_aligned = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(@as({.payload.orig_ty%summary}, &{.payload.val%summary})) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrUavAligned,
        },
        .ptr_comptime_field = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.field_val%summary}) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrComptimeField,
        },
        .ptr_int = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value}))",
            .payload = PtrInt,
        },
        .ptr_eu_payload = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&({.payload.base%summary} catch unreachable)) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrBase,
        },
        .ptr_opt_payload = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.base%summary}.?) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrBase,
        },
        .ptr_elem = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.base%summary}[{.payload.index%summary}]) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrBaseIndex,
        },
        .ptr_field = .{
            .summary = .@"@as({.payload.ty%summary}, @ptrFromInt(@intFromPtr(&{.payload.base%summary}[{.payload.index%summary}]) + ({.payload.byte_offset_a%value} << 32 | {.payload.byte_offset_b%value})))",
            .payload = PtrBaseIndex,
        },
        .ptr_slice = .{
            .summary = .@"{.payload.ptr%summary}[0..{.payload.len%summary}]",
            .payload = PtrSlice,
        },
        .opt_payload = .{ .summary = .@"@as({.payload.ty%summary}, {.payload.val%summary})", .payload = TypeValue },
        .opt_null = .{ .summary = .@"@as({.data%summary}, null)", .data = Index },
        .int_u8 = .{ .summary = .@"@as(u8, {.data%value})", .data = u8 },
        .int_u16 = .{ .summary = .@"@as(u16, {.data%value})", .data = u16 },
        .int_u32 = .{ .summary = .@"@as(u32, {.data%value})", .data = u32 },
        .int_i32 = .{ .summary = .@"@as(i32, {.data%value})", .data = i32 },
        .int_usize = .{ .summary = .@"@as(usize, {.data%value})", .data = u32 },
        .int_comptime_int_u32 = .{ .summary = .@"{.data%value}", .data = u32 },
        .int_comptime_int_i32 = .{ .summary = .@"{.data%value}", .data = i32 },
        .int_small = .{ .summary = .@"@as({.payload.ty%summary}, {.payload.value%value})", .payload = IntSmall },
        .int_positive = .{},
        .int_negative = .{},
        .int_lazy_align = .{ .summary = .@"@as({.payload.ty%summary}, @alignOf({.payload.lazy_ty%summary}))", .payload = IntLazy },
        .int_lazy_size = .{ .summary = .@"@as({.payload.ty%summary}, @sizeOf({.payload.lazy_ty%summary}))", .payload = IntLazy },
        .error_set_error = .{ .summary = .@"@as({.payload.ty%summary}, error.@{.payload.name%summary})", .payload = Error },
        .error_union_error = .{ .summary = .@"@as({.payload.ty%summary}, error.@{.payload.name%summary})", .payload = Error },
        .error_union_payload = .{ .summary = .@"@as({.payload.ty%summary}, {.payload.val%summary})", .payload = TypeValue },
        .enum_literal = .{ .summary = .@".@{.data%summary}", .data = NullTerminatedString },
        .enum_tag = .{ .summary = .@"@as({.payload.ty%summary}, @enumFromInt({.payload.int%summary}))", .payload = EnumTag },
        .float_f16 = .{ .summary = .@"@as(f16, {.data%value})", .data = f16 },
        .float_f32 = .{ .summary = .@"@as(f32, {.data%value})", .data = f32 },
        .float_f64 = .{ .summary = .@"@as(f64, {.payload%value})", .payload = f64 },
        .float_f80 = .{ .summary = .@"@as(f80, {.payload%value})", .payload = f80 },
        .float_f128 = .{ .summary = .@"@as(f128, {.payload%value})", .payload = f128 },
        .float_c_longdouble_f80 = .{ .summary = .@"@as(c_longdouble, {.payload%value})", .payload = f80 },
        .float_c_longdouble_f128 = .{ .summary = .@"@as(c_longdouble, {.payload%value})", .payload = f128 },
        .float_comptime_float = .{ .summary = .@"{.payload%value}", .payload = f128 },
        .variable = .{ .summary = .@"{.payload.owner_nav.fqn%summary#\"}", .payload = Variable },
        .@"extern" = .{ .summary = .@"{.payload.owner_nav.fqn%summary#\"}", .payload = Extern },
        .func_decl = .{
            .summary = .@"{.payload.owner_nav.fqn%summary#\"}",
            .payload = FuncDecl,
            .trailing = struct { inferred_error_set: ?Index },
            .config = .{ .@"trailing.inferred_error_set.?" = .@"payload.analysis.inferred_error_set" },
        },
        .func_instance = .{
            .summary = .@"{.payload.owner_nav.fqn%summary#\"}",
            .payload = FuncInstance,
            .trailing = struct {
                inferred_error_set: ?Index,
                param_values: []Index,
            },
            .config = .{
                .@"trailing.inferred_error_set.?" = .@"payload.analysis.inferred_error_set",
                .@"trailing.param_values.len" = .@"payload.ty.payload.params_len",
            },
        },
        .func_coerced = .{
            .summary = .@"@as(*const {.payload.ty%summary}, @ptrCast(&{.payload.func%summary})).*",
            .payload = FuncCoerced,
        },
        .only_possible_value = .{ .summary = .@"@as({.data%summary}, undefined)", .data = Index },
        .union_value = .{ .summary = .@"@as({.payload.ty%summary}, {})", .payload = Union },
        .bytes = .{ .summary = .@"@as({.payload.ty%summary}, {.payload.bytes%summary}.*)", .payload = Bytes },
        .aggregate = .{
            .summary = .@"@as({.payload.ty%summary}, .{...})",
            .payload = Aggregate,
            .trailing = struct { elements: []Index },
            .config = .{ .@"trailing.elements.len" = .@"payload.ty.payload.fields_len" },
        },
        .repeated = .{ .summary = .@"@as({.payload.ty%summary}, @splat({.payload.elem_val%summary}))", .payload = Repeated },

        .memoized_call = .{
            .summary = .@"@memoize({.payload.func%summary})",
            .payload = MemoizedCall,
            .trailing = struct { arg_values: []Index },
            .config = .{ .@"trailing.arg_values.len" = .@"payload.args_len" },
        },
    };
    fn Payload(comptime tag: Tag) type {
        return @field(encodings, @tagName(tag)).payload;
    }

    pub const Variable = struct {
        ty: Index,
        /// May be `none`.
        init: Index,
        owner_nav: Nav.Index,
        flags: Flags,

        pub const Flags = packed struct(u32) {
            is_const: bool,
            is_threadlocal: bool,
            is_weak_linkage: bool,
            is_dll_import: bool,
            _: u28 = 0,
        };
    };

    pub const Extern = struct {
        // name, alignment, addrspace come from `owner_nav`.
        ty: Index,
        lib_name: OptionalNullTerminatedString,
        flags: Variable.Flags,
        owner_nav: Nav.Index,
        zir_index: TrackedInst.Index,
    };

    /// Trailing:
    /// 0. element: Index for each len
    /// len is determined by the aggregate type.
    pub const Aggregate = struct {
        /// The type of the aggregate.
        ty: Index,
    };

    /// Trailing:
    /// 0. If `analysis.inferred_error_set` is `true`, `Index` of an `error_set` which
    ///    is a regular error set corresponding to the finished inferred error set.
    ///    A `none` value marks that the inferred error set is not resolved yet.
    pub const FuncDecl = struct {
        analysis: FuncAnalysis,
        owner_nav: Nav.Index,
        ty: Index,
        zir_body_inst: TrackedInst.Index,
        lbrace_line: u32,
        rbrace_line: u32,
        lbrace_column: u32,
        rbrace_column: u32,
    };

    /// Trailing:
    /// 0. If `analysis.inferred_error_set` is `true`, `Index` of an `error_set` which
    ///    is a regular error set corresponding to the finished inferred error set.
    ///    A `none` value marks that the inferred error set is not resolved yet.
    /// 1. For each parameter of generic_owner: `Index` if comptime, otherwise `none`
    pub const FuncInstance = struct {
        analysis: FuncAnalysis,
        // Needed by the linker for codegen. Not part of hashing or equality.
        owner_nav: Nav.Index,
        ty: Index,
        branch_quota: u32,
        /// Points to a `FuncDecl`.
        generic_owner: Index,
    };

    pub const FuncCoerced = struct {
        ty: Index,
        func: Index,
    };

    /// Trailing:
    /// 0. name: NullTerminatedString for each names_len
    pub const ErrorSet = struct {
        names_len: u32,
        /// Maps error names to declaration index.
        names_map: MapIndex,
    };

    /// Trailing:
    /// 0. comptime_bits: u32, // if has_comptime_bits
    /// 1. noalias_bits: u32, // if has_noalias_bits
    /// 2. param_type: Index for each params_len
    pub const TypeFunction = struct {
        params_len: u32,
        return_type: Index,
        flags: Flags,

        pub const Flags = packed struct(u32) {
            cc: PackedCallingConvention,
            is_var_args: bool,
            is_generic: bool,
            has_comptime_bits: bool,
            has_noalias_bits: bool,
            is_noinline: bool,
            _: u9 = 0,
        };
    };

    /// Trailing:
    /// 0. captures_len: u32 // if `any_captures`
    /// 1. capture: CaptureValue // for each `captures_len`
    /// 2. type_hash: PackedU64 // if `is_reified`
    /// 3. field type: Index for each field; declaration order
    /// 4. field align: Alignment for each field; declaration order
    pub const TypeUnion = struct {
        name: NullTerminatedString,
        flags: Flags,
        /// This could be provided through the tag type, but it is more convenient
        /// to store it directly. This is also necessary for `dumpStatsFallible` to
        /// work on unresolved types.
        fields_len: u32,
        /// Only valid after .have_layout
        size: u32,
        /// Only valid after .have_layout
        padding: u32,
        namespace: NamespaceIndex,
        /// The enum that provides the list of field names and values.
        tag_ty: Index,
        zir_index: TrackedInst.Index,

        pub const Flags = packed struct(u32) {
            any_captures: bool,
            runtime_tag: LoadedUnionType.RuntimeTag,
            /// If false, the field alignment trailing data is omitted.
            any_aligned_fields: bool,
            layout: std.builtin.Type.ContainerLayout,
            status: LoadedUnionType.Status,
            requires_comptime: RequiresComptime,
            assumed_runtime_bits: bool,
            assumed_pointer_aligned: bool,
            alignment: Alignment,
            is_reified: bool,
            _: u12 = 0,
        };
    };

    /// Trailing:
    /// 0. captures_len: u32 // if `any_captures`
    /// 1. capture: CaptureValue // for each `captures_len`
    /// 2. type_hash: PackedU64 // if `is_reified`
    /// 3. type: Index for each fields_len
    /// 4. name: NullTerminatedString for each fields_len
    /// 5. init: Index for each fields_len // if tag is type_struct_packed_inits
    pub const TypeStructPacked = struct {
        name: NullTerminatedString,
        zir_index: TrackedInst.Index,
        fields_len: u32,
        namespace: NamespaceIndex,
        backing_int_ty: Index,
        names_map: MapIndex,
        flags: Flags,

        pub const Flags = packed struct(u32) {
            any_captures: bool = false,
            /// Dependency loop detection when resolving field inits.
            field_inits_wip: bool = false,
            inits_resolved: bool = false,
            is_reified: bool = false,
            _: u28 = 0,
        };
    };

    /// At first I thought of storing the denormalized data externally, such as...
    ///
    /// * runtime field order
    /// * calculated field offsets
    /// * size and alignment of the struct
    ///
    /// ...since these can be computed based on the other data here. However,
    /// this data does need to be memoized, and therefore stored in memory
    /// while the compiler is running, in order to avoid O(N^2) logic in many
    /// places. Since the data can be stored compactly in the InternPool
    /// representation, it is better for memory usage to store denormalized data
    /// here, and potentially also better for performance as well. It's also simpler
    /// than coming up with some other scheme for the data.
    ///
    /// Trailing:
    /// 0. captures_len: u32 // if `any_captures`
    /// 1. capture: CaptureValue // for each `captures_len`
    /// 2. type_hash: PackedU64 // if `is_reified`
    /// 3. type: Index for each field in declared order
    /// 4. if any_default_inits:
    ///    init: Index // for each field in declared order
    /// 5. if any_aligned_fields:
    ///    align: Alignment // for each field in declared order
    /// 6. if any_comptime_fields:
    ///    field_is_comptime_bits: u32 // minimal number of u32s needed, LSB is field 0
    /// 7. if not is_extern:
    ///    field_index: RuntimeOrder // for each field in runtime order
    /// 8. field_offset: u32 // for each field in declared order, undef until layout_resolved
    pub const TypeStruct = struct {
        name: NullTerminatedString,
        zir_index: TrackedInst.Index,
        namespace: NamespaceIndex,
        fields_len: u32,
        flags: Flags,
        size: u32,

        pub const Flags = packed struct(u32) {
            any_captures: bool = false,
            is_extern: bool = false,
            known_non_opv: bool = false,
            requires_comptime: RequiresComptime = @enumFromInt(0),
            assumed_runtime_bits: bool = false,
            assumed_pointer_aligned: bool = false,
            any_comptime_fields: bool = false,
            any_default_inits: bool = false,
            any_aligned_fields: bool = false,
            /// `.none` until layout_resolved
            alignment: Alignment = @enumFromInt(0),
            /// Dependency loop detection when resolving struct alignment.
            alignment_wip: bool = false,
            /// Dependency loop detection when resolving field types.
            field_types_wip: bool = false,
            /// Dependency loop detection when resolving struct layout.
            layout_wip: bool = false,
            /// Indicates whether `size`, `alignment`, runtime field order, and
            /// field offets are populated.
            layout_resolved: bool = false,
            /// Dependency loop detection when resolving field inits.
            field_inits_wip: bool = false,
            /// Indicates whether `field_inits` has been resolved.
            inits_resolved: bool = false,
            // The types and all its fields have had their layout resolved. Even through pointer = false,
            // which `layout_resolved` does not ensure.
            fully_resolved: bool = false,
            is_reified: bool = false,
            _: u8 = 0,
        };
    };

    /// Trailing:
    /// 0. capture: CaptureValue // for each `captures_len`
    pub const TypeOpaque = struct {
        name: NullTerminatedString,
        /// Contains the declarations inside this opaque.
        namespace: NamespaceIndex,
        /// The index of the `opaque_decl` instruction.
        zir_index: TrackedInst.Index,
        /// `std.math.maxInt(u32)` indicates this type is reified.
        captures_len: u32,
    };
}```
嗯，我现在要总结这段Zig代码的思维模型。首先，我需要理解这段代码的结构和用途。看起来这是一个枚举类型的定义，可能用于某种类型系统或者内存池的管理。枚举的每个成员都有详细的注释，说明它们的作用和相关数据。

首先，我注意到`pub const Tag = enum(u8) { ... }`定义了一个公共的枚举类型，每个成员都有一个u8类型的值。每个枚举成员都有注释，解释它们的用途和数据的含义。例如，`removed`表示被移除的值，`type_int_signed`和`type_int_unsigned`表示有符号和无符号整数类型，数据是位数。接下来有很多类型相关的成员，比如数组类型、指针类型、切片类型等等，每个都有不同的数据结构和用途。

接下来，代码中定义了一些结构体，如`ErrorUnionType`、`TypeValue`、`Error`、`EnumTag`、`Union`、`TypePointer`等，这些可能用于存储不同类型的具体信息。然后有一个`enum_explicit_encoding`的常量，包含了一些配置信息，可能用于序列化或编码枚举类型。

在`encodings`这个结构体中，每个枚举成员对应一个配置，包括`summary`格式字符串和`data`或`payload`的类型。例如，`type_int_signed`的`summary`是`i{.data%value}`，表示格式化输出时有符号整数的数据值。这可能是用于生成类型或值的字符串表示。

接下来是各种结构体的定义，比如`Variable`、`Extern`、`Aggregate`、`FuncDecl`、`FuncInstance`等，这些结构体可能用于存储不同类型的具体数据，如变量、外部函数、聚合类型、函数声明、函数实例等。每个结构体都有详细的字段，包括类型索引、标志位、命名空间索引等，显示了复杂的数据管理和类型系统结构。

在`TypeFunction`结构体中，有参数长度、返回类型和标志位，标志位包括调用约定、是否可变参数、是否泛型等，显示了函数类型的详细配置。`TypeUnion`和`TypeStruct`等结构体也展示了联合类型和结构体的复杂配置，包括字段数量、对齐方式、捕获信息等。

最后，还有一些辅助函数，如`Payload`，用于根据枚举标签获取对应的payload类型。

总结起来，这段代码定义了一个复杂的类型系统，用于管理不同类型的表示和存储，包括基本类型、复合类型、函数类型、错误处理等。每个类型都有对应的数据结构和编码方式，支持类型检查、内存管理和序列化等功能。枚举类型`Tag`作为核心，区分不同的类型，而各种结构体则存储具体类型的详细信息。整个设计注重灵活性和扩展性，能够处理复杂的类型关系和编译时计算。
================================================
这段代码定义了一个复杂的类型系统与内存池管理模型，主要用于编译器或语言运行时对类型和值的内部表示、存储及查询。以下是其核心思维模型的总结：

---

### **核心结构**
1. **枚举 `Tag`**  
   作为类型和值的唯一标识符，每个枚举成员对应一种**类型**或**值**的类别（如整数类型、指针、函数、错误类型等）。  
   - 每个成员携带元数据（如 `data` 或 `payload`），用于存储具体信息（如位数、关联类型索引、编码配置等）。
   - 示例：
     - `type_int_signed`: 有符号整数类型，`data` 存储位数。
     - `ptr_slice`: 切片指针，`payload` 存储底层指针和长度。
     - `error_union_error`: 错误联合类型，`payload` 存储错误名称。

2. **编码配置 (`encodings`)**  
   为每个 `Tag` 成员定义其**序列化规则**和**数据存储方式**：
   - `summary`: 格式字符串，用于生成人类可读的表示（如 `@as(u8, {.data%value})`）。
   - `data`/`payload`: 指定存储的数据类型（如 `u32`、自定义结构体）。
   - `trailing` 和 `config`: 定义动态附加数据的布局（如字段列表、捕获值），支持复杂类型的动态扩展。

---

### **数据类型与存储**
1. **基础类型**  
   - 整型（`int_u8`、`int_comptime_int_u32`）、浮点型（`float_f32`）、布尔值等直接通过 `data` 存储值。
   - 复杂类型（如数组、结构体、联合体）通过 `payload` 关联到额外结构体（如 `Array`、`TypeStruct`），存储字段、对齐、命名空间等信息。

2. **复合类型**  
   - **函数类型 (`TypeFunction`)**  
     包含参数类型列表、返回类型、调用约定等标志位（如是否泛型、是否可变参数）。
   - **结构体 (`TypeStruct`)**  
     存储字段类型、对齐、初始化值、布局信息（如大小、偏移量），支持编译时和运行时的布局解析。
   - **错误处理 (`error_union_error`、`error_set_error`)**  
     通过关联的错误集合（`ErrorSet`）和名称映射实现错误类型的唯一性检查。

3. **指针与内存管理**  
   - 多种指针类型（如 `ptr_uav`、`ptr_slice`）通过 `payload` 存储基地址、偏移量、对齐信息，支持动态内存操作。
   - 特殊指针（如 `ptr_comptime_alloc`）用于编译时分配的内存地址管理。

---

### **关键机制**
1. **动态数据附加 (`trailing`)**  
   允许类型在固定 `payload` 外附加动态数据（如函数参数列表、结构体字段），通过 `config` 定义数据长度和条件（如 `param_values.len = payload.params_len`）。

2. **标志位与状态管理**  
   - 结构体（如 `TypeStruct.Flags`）使用位字段标记状态（如 `is_extern`、`layout_resolved`），管理类型解析、布局计算等过程。
   - 错误处理依赖状态标志（如 `inferred_error_set`）跟踪未解析的错误集合。

3. **命名空间与符号管理**  
   - `namespace` 字段关联类型到其所属的命名空间，支持模块化符号解析。
   - `owner_nav` 用于追踪声明来源（如函数、变量的 AST 节点位置）。

---

### **功能目标**
- **类型唯一化 (Interning)**  
  通过 `InternPool` 管理类型的唯一实例，避免重复存储，加速类型比较和查询。
- **编译时计算**  
  支持 `comptime` 特性（如 `int_lazy_align` 延迟计算类型对齐），依赖类型系统的元编程能力。
- **跨平台兼容**  
  处理不同目标平台的特性（如 `float_c_longdouble_f80` 适配 80 位浮点格式）。

---

### **总结**
这段代码实现了一个高度结构化的类型系统，通过枚举 `Tag` 分类所有可能的类型和值，结合动态数据附加和标志位管理，支持复杂类型的表示、内存布局计算、编译时优化及错误处理。其设计注重内存效率（通过唯一化存储）和扩展性（通过动态 `trailing` 数据），是编译器或语言运行时实现类型安全、元编程和跨平台兼容的核心基础设施。