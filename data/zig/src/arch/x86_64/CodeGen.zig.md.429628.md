```zig
const Select = struct {
    cg: *CodeGen,
    types: [@intFromEnum(Select.Operand.Ref.none)]Type,
    temps: [@intFromEnum(Select.Operand.Ref.none)]Temp,
    labels: [@intFromEnum(Label._)]struct {
        backward: ?Mir.Inst.Index,
        forward: [1]?Mir.Inst.Index,
    },
    top: u3,

    const Error = InnerError || error{SelectFailed};

    fn emitLabel(s: *Select, label_index: Label) void {
        assert(@intFromEnum(label_index) < @intFromEnum(Label._));
        const label = &s.labels[@intFromEnum(label_index)];
        for (&label.forward) |*reloc| {
            if (reloc.*) |r| s.cg.performReloc(r);
            reloc.* = null;
        }
        label.backward = @intCast(s.cg.mir_instructions.len);
    }

    fn emit(s: *Select, inst: Instruction) InnerError!void {
        const mir_tag: Mir.Inst.FixedTag = .{ inst[1], inst[2] };
        pseudo: {
            switch (inst[0]) {
                .@"0:", .@"1:", .@"2:", .@"3:", .@"4:" => |label| s.emitLabel(label),
                ._ => {},
                .pseudo => break :pseudo,
            }
            var mir_ops: [4]CodeGen.Operand = undefined;
            inline for (&mir_ops, 3..) |*mir_op, inst_index| mir_op.* = try inst[inst_index].lower(s);
            s.cg.asmOps(mir_tag, mir_ops) catch |err| switch (err) {
                error.InvalidInstruction => {
                    const fixes = @tagName(mir_tag[0]);
                    const fixes_blank = std.mem.indexOfScalar(u8, fixes, '_').?;
                    return s.cg.fail(
                        "invalid instruction: '{s}{s}{s} {s} {s} {s} {s}'",
                        .{
                            fixes[0..fixes_blank],
                            @tagName(mir_tag[1]),
                            fixes[fixes_blank + 1 ..],
                            @tagName(mir_ops[0]),
                            @tagName(mir_ops[1]),
                            @tagName(mir_ops[2]),
                            @tagName(mir_ops[3]),
                        },
                    );
                },
                else => |e| return e,
            };
        }
        switch (mir_tag[0]) {
            .f_ => switch (mir_tag[1]) {
                .@"2xm1",
                .abs,
                .add,
                .chs,
                .clex,
                .com,
                .comi,
                .cos,
                .div,
                .divr,
                .free,
                .mul,
                .nop,
                .prem,
                .rndint,
                .scale,
                .sin,
                .sqrt,
                .st,
                .sub,
                .subr,
                .tst,
                .ucom,
                .ucomi,
                .wait,
                .xam,
                .xch,
                => {},
                .init, .save => s.top = 0,
                .ld, .ptan, .sincos, .xtract => s.top -%= 1,
                .patan, .yl2x => s.top +%= 1,
                .rstor => unreachable,
                else => unreachable,
            },
            .f_1 => switch (mir_tag[1]) {
                .ld => s.top -%= 1,
                .prem => {},
                else => unreachable,
            },
            .f_b, .f_be, .f_e, .f_nb, .f_nbe, .f_ne, .f_nu, .f_u => switch (mir_tag[1]) {
                .cmov => {},
                else => unreachable,
            },
            .f_cw, .f_env, .f_sw => switch (mir_tag[1]) {
                .ld, .st => {},
                else => unreachable,
            },
            .f_p1 => switch (mir_tag[1]) {
                .yl2x => s.top +%= 1,
                else => unreachable,
            },
            .f_cstp => switch (mir_tag[1]) {
                .de => s.top -%= 1,
                .in => s.top +%= 1,
                else => unreachable,
            },
            .f_l2e, .f_l2t, .f_lg2, .f_ln2, .f_pi, .f_z => switch (mir_tag[1]) {
                .ld => s.top -%= 1,
                else => unreachable,
            },
            .f_p => switch (mir_tag[1]) {
                .add, .com, .comi, .div, .divr, .mul, .st, .sub, .subr, .ucom, .ucomi => s.top +%= 1,
                else => {
                    const fixes = @tagName(mir_tag[0]);
                    const fixes_blank = std.mem.indexOfScalar(u8, fixes, '_').?;
                    std.debug.panic("{s}: {s}{s}{s}\n", .{
                        @src().fn_name,
                        fixes[0..fixes_blank],
                        @tagName(mir_tag[1]),
                        fixes[fixes_blank + 1 ..],
                    });
                },
            },
            .f_pp => switch (mir_tag[1]) {
                .com, .ucom => s.top +%= 2,
                else => unreachable,
            },
            .fb_ => switch (mir_tag[1]) {
                .ld => s.top -%= 1,
                else => unreachable,
            },
            .fb_p => switch (mir_tag[1]) {
                .st => s.top +%= 1,
                else => unreachable,
            },
            .fi_ => switch (mir_tag[1]) {
                .add, .com, .div, .divr, .mul, .st, .stt, .sub, .subr => {},
                .ld => s.top -%= 1,
                else => unreachable,
            },
            .fi_p => switch (mir_tag[1]) {
                .com, .st, .stt => s.top +%= 1,
                else => unreachable,
            },
            .fn_ => switch (mir_tag[1]) {
                .clex => {},
                .init, .save => s.top = 0,
                else => unreachable,
            },
            .fn_cw, .fn_env, .fn_sw => switch (mir_tag[1]) {
                .st => {},
                else => unreachable,
            },
            .fx_ => switch (mir_tag[1]) {
                .rstor => unreachable,
                .save => {},
                else => unreachable,
            },
            else => {},
        }
    }

    fn lowerReg(s: *const Select, reg: Register) Register {
        if (reg.class() != .x87) return reg;
        return @enumFromInt(@intFromEnum(Register.st0) + (@as(u3, @intCast(reg.enc())) -% s.top));
    }

    const Case = struct {
        required_features: [4]?std.Target.x86.Feature = @splat(null),
        src_constraints: [@intFromEnum(Select.Operand.Ref.none) - @intFromEnum(Select.Operand.Ref.src0)]Constraint = @splat(.any),
        dst_constraints: [@intFromEnum(Select.Operand.Ref.src0) - @intFromEnum(Select.Operand.Ref.dst0)]Constraint = @splat(.any),
        patterns: []const Select.Pattern,
        call_frame: packed struct(u16) { size: u10 = 0, alignment: InternPool.Alignment } = .{ .size = 0, .alignment = .none },
        extra_temps: [@intFromEnum(Select.Operand.Ref.dst0) - @intFromEnum(Select.Operand.Ref.tmp0)]TempSpec = @splat(.unused),
        dst_temps: [@intFromEnum(Select.Operand.Ref.src0) - @intFromEnum(Select.Operand.Ref.dst0)]TempSpec.Kind = @splat(.unused),
        clobbers: packed struct {
            eflags: bool = false,
            caller_preserved: CallConv = .none,
        } = .{},
        each: union(enum) {
            once: []const Instruction,
        },

        const CallConv = enum(u2) { none, ccc, zigcc };
    };

    const Constraint = union(enum) {
        any,
        any_bool_vec,
        any_int,
        any_signed_int,
        any_unsigned_int,
        any_scalar_int,
        any_scalar_signed_int,
        any_scalar_unsigned_int,
        any_float,
        po2_any,
        bool_vec: Memory.Size,
        vec: Memory.Size,
        signed_int_vec: Memory.Size,
        signed_int_or_full_vec: Memory.Size,
        unsigned_int_vec: Memory.Size,
        size: Memory.Size,
        multiple_size: Memory.Size,
        int: Memory.Size,
        scalar_int_is: Memory.Size,
        scalar_signed_int_is: Memory.Size,
        scalar_unsigned_int_is: Memory.Size,
        scalar_int: OfIsSizes,
        scalar_signed_int: OfIsSizes,
        scalar_unsigned_int: OfIsSizes,
        scalar_signed_or_exclusive_int: OfIsSizes,
        scalar_exact_int: struct { of: Memory.Size, is: u16 },
        scalar_exact_signed_int: struct { of: Memory.Size, is: u16 },
        scalar_exact_unsigned_int: struct { of: Memory.Size, is: u16 },
        multiple_scalar_int: OfIsSizes,
        multiple_scalar_signed_int: OfIsSizes,
        multiple_scalar_unsigned_int: OfIsSizes,
        multiple_scalar_signed_or_exclusive_int: OfIsSizes,
        multiple_scalar_exact_int: struct { of: Memory.Size, is: u16 },
        multiple_scalar_exact_signed_int: struct { of: Memory.Size, is: u16 },
        multiple_scalar_exact_unsigned_int: struct { of: Memory.Size, is: u16 },
        scalar_remainder_int: OfIsSizes,
        scalar_remainder_signed_int: OfIsSizes,
        scalar_remainder_unsigned_int: OfIsSizes,
        scalar_exact_remainder_int: OfIsSizes,
        scalar_exact_remainder_signed_int: OfIsSizes,
        scalar_exact_remainder_unsigned_int: OfIsSizes,
        float: Memory.Size,
        scalar_any_float: Memory.Size,
        scalar_float: OfIsSizes,
        multiple_scalar_any_float: Memory.Size,
        multiple_scalar_float: OfIsSizes,
        exact_int: u16,
        exact_signed_int: u16,
        exact_unsigned_int: u16,
        signed_or_exact_int: Memory.Size,
        unsigned_or_exact_int: Memory.Size,
        signed_or_exclusive_int: Memory.Size,
        po2_int: Memory.Size,
        signed_po2_int: Memory.Size,
        unsigned_po2_or_exact_int: Memory.Size,
        remainder_int: OfIsSizes,
        remainder_signed_int: OfIsSizes,
        remainder_unsigned_int: OfIsSizes,
        exact_remainder_int: OfIsSizes,
        exact_remainder_signed_int: OfIsSizes,
        exact_remainder_unsigned_int: OfIsSizes,
        signed_or_exact_remainder_int: OfIsSizes,
        unsigned_or_exact_remainder_int: OfIsSizes,
        signed_int: Memory.Size,
        unsigned_int: Memory.Size,
        elem_size_is: u8,
        po2_elem_size,
        elem_int: Memory.Size,

        const OfIsSizes = struct { of: Memory.Size, is: Memory.Size };

        fn accepts(constraint: Constraint, ty: Type, cg: *CodeGen) bool {
            const zcu = cg.pt.zcu;
            return switch (constraint) {
                .any => true,
                .any_bool_vec => ty.isVector(zcu) and ty.childType(zcu).toIntern() == .bool_type,
                .any_int => cg.intInfo(ty) != null,
                .any_signed_int => if (cg.intInfo(ty)) |int_info| int_info.signedness == .signed else false,
                .any_unsigned_int => if (cg.intInfo(ty)) |int_info| int_info.signedness == .unsigned else false,
                .any_scalar_int => cg.intInfo(ty.scalarType(zcu)) != null,
                .any_scalar_signed_int => if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed else false,
                .any_scalar_unsigned_int => if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned else false,
                .any_float => ty.isRuntimeFloat(),
                .po2_any => std.math.isPowerOfTwo(ty.abiSize(zcu)),
                .bool_vec => |size| ty.isVector(zcu) and ty.scalarType(zcu).toIntern() == .bool_type and
                    size.bitSize(cg.target) >= ty.vectorLen(zcu),
                .vec => |size| ty.isVector(zcu) and ty.scalarType(zcu).toIntern() != .bool_type and
                    size.bitSize(cg.target) >= ty.abiSize(zcu),
                .signed_int_vec => |size| ty.isVector(zcu) and @divExact(size.bitSize(cg.target), 8) >= ty.abiSize(zcu) and
                    if (cg.intInfo(ty.childType(zcu))) |int_info| int_info.signedness == .signed else false,
                .signed_int_or_full_vec => |size| ty.isVector(zcu) and @divExact(size.bitSize(cg.target), 8) >= ty.abiSize(zcu) and
                    if (cg.intInfo(ty.childType(zcu))) |int_info| switch (int_info.signedness) {
                        .signed => true,
                        .unsigned => int_info.bits >= 8 and std.math.isPowerOfTwo(int_info.bits),
                    } else false,
                .unsigned_int_vec => |size| ty.isVector(zcu) and @divExact(size.bitSize(cg.target), 8) >= ty.abiSize(zcu) and
                    if (cg.intInfo(ty.childType(zcu))) |int_info| int_info.signedness == .unsigned else false,
                .size => |size| @divExact(size.bitSize(cg.target), 8) >= ty.abiSize(zcu),
                .multiple_size => |size| ty.abiSize(zcu) % @divExact(size.bitSize(cg.target), 8) == 0,
                .int => |size| if (cg.intInfo(ty)) |int_info| size.bitSize(cg.target) >= int_info.bits else false,
                .scalar_int_is => |size| if (cg.intInfo(ty.scalarType(zcu))) |int_info|
                    size.bitSize(cg.target) >= int_info.bits
                else
                    false,
                .scalar_signed_int_is => |size| if (cg.intInfo(ty.scalarType(zcu))) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) >= int_info.bits,
                    .unsigned => false,
                } else false,
                .scalar_unsigned_int_is => |size| if (cg.intInfo(ty.scalarType(zcu))) |int_info| switch (int_info.signedness) {
                    .signed => false,
                    .unsigned => size.bitSize(cg.target) >= int_info.bits,
                } else false,
                .scalar_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .scalar_signed_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                        of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .scalar_unsigned_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                        of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .scalar_signed_or_exclusive_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                        .signed => of_is.is.bitSize(cg.target) >= int_info.bits,
                        .unsigned => of_is.is.bitSize(cg.target) > int_info.bits,
                    } else false,
                .scalar_exact_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| of_is.is == int_info.bits else false,
                .scalar_exact_signed_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                        of_is.is == int_info.bits else false,
                .scalar_exact_unsigned_int => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                        of_is.is == int_info.bits else false,
                .multiple_scalar_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .multiple_scalar_signed_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                        of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .multiple_scalar_unsigned_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                        of_is.is.bitSize(cg.target) >= int_info.bits else false,
                .multiple_scalar_signed_or_exclusive_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| switch (int_info.signedness) {
                        .signed => of_is.is.bitSize(cg.target) >= int_info.bits,
                        .unsigned => of_is.is.bitSize(cg.target) > int_info.bits,
                    } else false,
                .multiple_scalar_exact_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| of_is.is == int_info.bits else false,
                .multiple_scalar_exact_signed_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                        of_is.is == int_info.bits else false,
                .multiple_scalar_exact_unsigned_int => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                        of_is.is == int_info.bits else false,
                .scalar_remainder_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info|
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1
                else
                    false,
                .scalar_remainder_signed_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .scalar_remainder_unsigned_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .scalar_exact_remainder_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info|
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1
                else
                    false,
                .scalar_exact_remainder_signed_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .signed and
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .scalar_exact_remainder_unsigned_int => |of_is| if (cg.intInfo(ty.scalarType(zcu))) |int_info| int_info.signedness == .unsigned and
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .float => |size| if (cg.floatBits(ty)) |float_bits| size.bitSize(cg.target) == float_bits else false,
                .scalar_any_float => |size| @divExact(size.bitSize(cg.target), 8) >= ty.abiSize(zcu) and
                    cg.floatBits(ty.scalarType(zcu)) != null,
                .scalar_float => |of_is| @divExact(of_is.of.bitSize(cg.target), 8) >= cg.unalignedSize(ty) and
                    if (cg.floatBits(ty.scalarType(zcu))) |float_bits| of_is.is.bitSize(cg.target) == float_bits else false,
                .multiple_scalar_any_float => |size| ty.abiSize(zcu) % @divExact(size.bitSize(cg.target), 8) == 0 and
                    cg.floatBits(ty.scalarType(zcu)) != null,
                .multiple_scalar_float => |of_is| ty.abiSize(zcu) % @divExact(of_is.of.bitSize(cg.target), 8) == 0 and
                    if (cg.floatBits(ty.scalarType(zcu))) |float_bits| of_is.is.bitSize(cg.target) == float_bits else false,
                .exact_int => |bit_size| if (cg.intInfo(ty)) |int_info| bit_size == int_info.bits else false,
                .exact_signed_int => |bit_size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => bit_size == int_info.bits,
                    .unsigned => false,
                } else false,
                .exact_unsigned_int => |bit_size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => false,
                    .unsigned => bit_size == int_info.bits,
                } else false,
                .signed_or_exact_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) >= int_info.bits,
                    .unsigned => size.bitSize(cg.target) == int_info.bits,
                } else false,
                .unsigned_or_exact_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) == int_info.bits,
                    .unsigned => size.bitSize(cg.target) >= int_info.bits,
                } else false,
                .signed_or_exclusive_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) >= int_info.bits,
                    .unsigned => size.bitSize(cg.target) > int_info.bits,
                } else false,
                .po2_int => |size| if (cg.intInfo(ty)) |int_info|
                    std.math.isPowerOfTwo(int_info.bits) and size.bitSize(cg.target) >= int_info.bits
                else
                    false,
                .signed_po2_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => std.math.isPowerOfTwo(int_info.bits) and size.bitSize(cg.target) >= int_info.bits,
                    .unsigned => false,
                } else false,
                .unsigned_po2_or_exact_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) == int_info.bits,
                    .unsigned => std.math.isPowerOfTwo(int_info.bits) and size.bitSize(cg.target) >= int_info.bits,
                } else false,
                .remainder_int => |of_is| if (cg.intInfo(ty)) |int_info|
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1
                else
                    false,
                .remainder_signed_int => |of_is| if (cg.intInfo(ty)) |int_info| int_info.signedness == .signed and
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .remainder_unsigned_int => |of_is| if (cg.intInfo(ty)) |int_info| int_info.signedness == .unsigned and
                    of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .exact_remainder_int => |of_is| if (cg.intInfo(ty)) |int_info|
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1
                else
                    false,
                .exact_remainder_signed_int => |of_is| if (cg.intInfo(ty)) |int_info| int_info.signedness == .signed and
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .exact_remainder_unsigned_int => |of_is| if (cg.intInfo(ty)) |int_info| int_info.signedness == .unsigned and
                    of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1 else false,
                .signed_or_exact_remainder_int => |of_is| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1,
                    .unsigned => of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1,
                } else false,
                .unsigned_or_exact_remainder_int => |of_is| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => of_is.is.bitSize(cg.target) == (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1,
                    .unsigned => of_is.is.bitSize(cg.target) >= (int_info.bits - 1) % of_is.of.bitSize(cg.target) + 1,
                } else false,
                .signed_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => size.bitSize(cg.target) >= int_info.bits,
                    .unsigned => false,
                } else false,
                .unsigned_int => |size| if (cg.intInfo(ty)) |int_info| switch (int_info.signedness) {
                    .signed => false,
                    .unsigned => size.bitSize(cg.target) >= int_info.bits,
                } else false,
                .elem_size_is => |size| size == ty.elemType2(zcu).abiSize(zcu),
                .po2_elem_size => std.math.isPowerOfTwo(ty.elemType2(zcu).abiSize(zcu)),
                .elem_int => |size| if (cg.intInfo(ty.elemType2(zcu))) |elem_int_info|
                    size.bitSize(cg.target) >= elem_int_info.bits
                else
                    false,
            };
        }
    };

    const Pattern = struct {
        src: [@intFromEnum(Select.Operand.Ref.none) - @intFromEnum(Select.Operand.Ref.src0)]Src,
        commute: struct { u8, u8 } = .{ 0, 0 },

        const Src = union(enum) {
            none,
            imm8,
            imm16,
            imm32,
            simm32,
            to_reg: Register,
            to_reg_pair: [2]Register,
            to_param_gpr: TempSpec.Kind.CallConvRegSpec,
            to_param_gpr_pair: TempSpec.Kind.CallConvRegSpec,
            to_ret_gpr: TempSpec.Kind.CallConvRegSpec,
            to_ret_gpr_pair: TempSpec.Kind.CallConvRegSpec,
            mem,
            to_mem,
            mut_mem,
            to_mut_mem,
            gpr,
            to_gpr,
            mut_gpr,
            to_mut_gpr,
            x87,
            to_x87,
            mut_x87,
            to_mut_x87,
            mmx,
            to_mmx,
            mut_mmx,
            to_mut_mmx,
            mm,
            to_mm,
            mut_mm,
            to_mut_mm,
            sse,
            to_sse,
            mut_sse,
            to_mut_sse,
            xmm,
            to_xmm,
            mut_xmm,
            to_mut_xmm,
            ymm,
            to_ymm,
            mut_ymm,
            to_mut_ymm,

            fn matches(src: Src, temp: Temp, cg: *CodeGen) bool {
                return switch (src) {
                    .none => temp.tracking(cg).short == .none,
                    .imm8 => switch (temp.tracking(cg).short) {
                        .immediate => |imm| std.math.cast(u8, imm) != null,
                        else => false,
                    },
                    .imm16 => switch (temp.tracking(cg).short) {
                        .immediate => |imm| std.math.cast(u16, imm) != null,
                        else => false,
                    },
                    .imm32 => switch (temp.tracking(cg).short) {
                        .immediate => |imm| std.math.cast(u32, imm) != null,
                        else => false,
                    },
                    .simm32 => switch (temp.tracking(cg).short) {
                        .immediate => |imm| std.math.cast(i32, @as(i64, @bitCast(imm))) != null,
                        else => false,
                    },
                    .mem => temp.tracking(cg).short.isMemory(),
                    .to_mem, .to_mut_mem => true,
                    .mut_mem => temp.isMut(cg) and temp.tracking(cg).short.isMemory(),
                    .to_reg, .to_reg_pair, .to_param_gpr, .to_param_gpr_pair, .to_ret_gpr, .to_ret_gpr_pair => true,
                    .gpr => temp.typeOf(cg).abiSize(cg.pt.zcu) <= 8 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .general_purpose,
                        .register_offset => |reg_off| reg_off.reg.class() == .general_purpose and reg_off.off == 0,
                        else => false,
                    },
                    .mut_gpr => temp.isMut(cg) and temp.typeOf(cg).abiSize(cg.pt.zcu) <= 8 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .general_purpose,
                        .register_offset => |reg_off| reg_off.reg.class() == .general_purpose and reg_off.off == 0,
                        else => false,
                    },
                    .to_gpr, .to_mut_gpr => temp.typeOf(cg).abiSize(cg.pt.zcu) <= 8,
                    .x87 => switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .x87,
                        .register_offset => |reg_off| reg_off.reg.class() == .x87 and reg_off.off == 0,
                        else => false,
                    },
                    .mut_x87 => temp.isMut(cg) and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .x87,
                        .register_offset => |reg_off| reg_off.reg.class() == .x87 and reg_off.off == 0,
                        else => false,
                    },
                    .to_x87, .to_mut_x87 => true,
                    .mmx => switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .mmx,
                        .register_offset => |reg_off| reg_off.reg.class() == .mmx and reg_off.off == 0,
                        else => false,
                    },
                    .mut_mmx => temp.isMut(cg) and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .mmx,
                        .register_offset => |reg_off| reg_off.reg.class() == .mmx and reg_off.off == 0,
                        else => false,
                    },
                    .to_mmx, .to_mut_mmx => true,
                    .mm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 8 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .mmx,
                        .register_offset => |reg_off| reg_off.reg.class() == .mmx and reg_off.off == 0,
                        else => false,
                    },
                    .mut_mm => temp.isMut(cg) and temp.typeOf(cg).abiSize(cg.pt.zcu) == 8 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .mmx,
                        .register_offset => |reg_off| reg_off.reg.class() == .mmx and reg_off.off == 0,
                        else => false,
                    },
                    .to_mm, .to_mut_mm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 8,
                    .sse => switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .mut_sse => temp.isMut(cg) and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .to_sse, .to_mut_sse => true,
                    .xmm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 16 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .mut_xmm => temp.isMut(cg) and temp.typeOf(cg).abiSize(cg.pt.zcu) == 16 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .to_xmm, .to_mut_xmm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 16,
                    .ymm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 32 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .mut_ymm => temp.isMut(cg) and temp.typeOf(cg).abiSize(cg.pt.zcu) == 32 and switch (temp.tracking(cg).short) {
                        .register => |reg| reg.class() == .sse,
                        .register_offset => |reg_off| reg_off.reg.class() == .sse and reg_off.off == 0,
                        else => false,
                    },
                    .to_ymm, .to_mut_ymm => temp.typeOf(cg).abiSize(cg.pt.zcu) == 32,
                };
            }

            fn convert(src: Src, temp: *Temp, cg: *CodeGen) InnerError!bool {
                return switch (src) {
                    .none, .imm8, .imm16, .imm32, .simm32 => false,
                    .mem, .to_mem => try temp.toBase(false, cg),
                    .mut_mem, .to_mut_mem => try temp.toBase(true, cg),
                    .to_reg => |reg| try temp.toReg(reg, cg),
                    .to_reg_pair => |regs| try temp.toRegPair(regs, cg),
                    .to_param_gpr => |param_spec| try temp.toReg(abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index], cg),
                    .to_param_gpr_pair => |param_spec| try temp.toRegPair(abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index..][0..2].*, cg),
                    .to_ret_gpr => |ret_spec| try temp.toReg(abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index], cg),
                    .to_ret_gpr_pair => |ret_spec| try temp.toRegPair(abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index..][0..2].*, cg),
                    .gpr, .to_gpr => try temp.toRegClass(false, .general_purpose, cg),
                    .mut_gpr, .to_mut_gpr => try temp.toRegClass(true, .general_purpose, cg),
                    .x87, .to_x87 => try temp.toRegClass(false, .x87, cg),
                    .mut_x87, .to_mut_x87 => try temp.toRegClass(true, .x87, cg),
                    .mmx, .to_mmx, .mm, .to_mm => try temp.toRegClass(false, .mmx, cg),
                    .mut_mmx, .to_mut_mmx, .mut_mm, .to_mut_mm => try temp.toRegClass(true, .mmx, cg),
                    .sse, .to_sse, .xmm, .to_xmm, .ymm, .to_ymm => try temp.toRegClass(false, .sse, cg),
                    .mut_sse, .to_mut_sse, .mut_xmm, .to_mut_xmm, .mut_ymm, .to_mut_ymm => try temp.toRegClass(true, .sse, cg),
                };
            }
        };
    };

    const TempSpec = struct {
        type: Type = .noreturn,
        kind: Kind,

        const unused: TempSpec = .{ .kind = .unused };

        const Kind = union(enum) {
            unused,
            any,
            cc: Condition,
            ref: Select.Operand.Ref,
            reg: Register,
            reg_pair: [2]Register,
            param_gpr: CallConvRegSpec,
            param_gpr_pair: CallConvRegSpec,
            ret_gpr: CallConvRegSpec,
            ret_gpr_pair: CallConvRegSpec,
            rc: Register.Class,
            rc_pair: Register.Class,
            mut_rc: struct { ref: Select.Operand.Ref, rc: Register.Class },
            ref_mask: struct { ref: Select.Operand.Ref, info: MaskInfo },
            rc_mask: struct { rc: Register.Class, info: MaskInfo },
            mut_rc_mask: struct { ref: Select.Operand.Ref, rc: Register.Class, info: MaskInfo },
            mem,
            mem_of_type: Select.Operand.Ref,
            smin_mem: ConstSpec,
            smax_mem: ConstSpec,
            umin_mem: ConstSpec,
            umax_mem: ConstSpec,
            @"0x1p63_mem": ConstSpec,
            f64_0x1p52_0x1p84_mem,
            u32_0x1p52_hi_0x1p84_hi_0_0_mem,
            f32_0_0x1p64_mem,
            pshufb_trunc_mem: struct { from: Memory.Size, to: Memory.Size },
            pshufb_bswap_mem: struct { repeat: u4 = 1, size: Memory.Size, smear: u4 = 1 },
            forward_bits,
            reverse_bits,
            frame: FrameIndex,
            lazy_symbol: struct { kind: link.File.LazySymbol.Kind, ref: Select.Operand.Ref = .none },
            symbol: *const struct { lib: ?[]const u8 = null, name: []const u8 },

            const ConstSpec = struct {
                ref: Select.Operand.Ref = .none,
                to_signedness: ?std.builtin.Signedness = null,
                vectorize_to: ?Memory.Size = null,
            };

            const CallConvRegSpec = struct {
                cc: Case.CallConv,
                index: u2,

                fn tag(spec: CallConvRegSpec, cg: *const CodeGen) std.builtin.CallingConvention.Tag {
                    return switch (spec.cc) {
                        .none => unreachable,
                        .ccc => cg.target.cCallingConvention().?,
                        .zigcc => .auto,
                    };
                }
            };

            fn lock(kind: Kind, cg: *CodeGen) ![2]?RegisterLock {
                var reg_locks: [2]?RegisterLock = @splat(null);
                const regs: [2]Register = switch (kind) {
                    else => return reg_locks,
                    .reg => |reg| .{ reg, .none },
                    .reg_pair => |regs| regs,
                    .param_gpr => |param_spec| abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index..][0..1].* ++ .{.none},
                    .param_gpr_pair => |param_spec| abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index..][0..2].*,
                    .ret_gpr => |ret_spec| abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index..][0..1].* ++ .{.none},
                    .ret_gpr_pair => |ret_spec| abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index..][0..2].*,
                };
                for (regs, &reg_locks) |reg, *reg_lock| {
                    if (reg == .none) continue;
                    const reg_index = RegisterManager.indexOfRegIntoTracked(reg) orelse continue;
                    try cg.register_manager.getRegIndex(reg_index, null);
                    reg_lock.* = cg.register_manager.lockRegIndex(reg_index);
                }
                return reg_locks;
            }

            fn finish(kind: Kind, temp: Temp, cg: *CodeGen) void {
                switch (kind) {
                    else => {},
                    inline .rc_mask, .mut_rc_mask, .ref_mask => |mask| temp.asMask(mask.info, cg),
                }
            }
        };

        fn create(spec: TempSpec, s: *const Select) InnerError!struct { Temp, bool } {
            const cg = s.cg;
            const pt = cg.pt;
            return switch (spec.kind) {
                .unused => .{ undefined, false },
                .any => .{ try cg.tempAlloc(spec.type), true },
                .cc => |cc| .{ try cg.tempInit(spec.type, .{ .eflags = cc }), true },
                .ref => |ref| .{ ref.tempOf(s), false },
                .reg => |reg| .{ try cg.tempInit(spec.type, .{ .register = reg }), true },
                .reg_pair => |regs| .{ try cg.tempInit(spec.type, .{ .register_pair = regs }), true },
                .param_gpr => |param_spec| .{ try cg.tempInit(spec.type, .{
                    .register = abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index],
                }), true },
                .param_gpr_pair => |param_spec| .{ try cg.tempInit(spec.type, .{
                    .register_pair = abi.getCAbiIntParamRegs(param_spec.tag(cg))[param_spec.index..][0..2].*,
                }), true },
                .ret_gpr => |ret_spec| .{ try cg.tempInit(spec.type, .{
                    .register = abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index],
                }), true },
                .ret_gpr_pair => |ret_spec| .{ try cg.tempInit(spec.type, .{
                    .register_pair = abi.getCAbiIntReturnRegs(ret_spec.tag(cg))[ret_spec.index..][0..2].*,
                }), true },
                .rc => |rc| .{ try cg.tempAllocReg(spec.type, regSetForRegClass(rc)), true },
                .rc_pair => |rc| .{ try cg.tempAllocRegPair(spec.type, regSetForRegClass(rc)), true },
                .mut_rc => |ref_rc| {
                    const temp = ref_rc.ref.tempOf(s);
                    if (temp.isMut(cg)) switch (temp.tracking(cg).short) {
                        .register => |reg| if (reg.class() == ref_rc.rc) return .{ temp, false },
                        .register_offset => |reg_off| if (reg_off.off == 0 and reg_off.reg.class() == ref_rc.rc) return .{ temp, false },
                        else => {},
                    };
                    return .{ try cg.tempAllocReg(spec.type, regSetForRegClass(ref_rc.rc)), true };
                },
                .ref_mask => |ref_mask| .{ ref_mask.ref.tempOf(s), false },
                .rc_mask => |rc_mask| .{ try cg.tempAllocReg(spec.type, regSetForRegClass(rc_mask.rc)), true },
                .mut_rc_mask => |ref_rc_mask| {
                    const temp = ref_rc_mask.ref.tempOf(s);
                    if (temp.isMut(cg)) switch (temp.tracking(cg).short) {
                        .register => |reg| if (reg.class() == ref_rc_mask.rc) return .{ temp, false },
                        .register_offset => |reg_off| if (reg_off.off == 0 and reg_off.reg.class() == ref_rc_mask.rc) return .{ temp, false },
                        else => {},
                    };
                    return .{ try cg.tempAllocReg(spec.type, regSetForRegClass(ref_rc_mask.rc)), true };
                },
                .mem => .{ try cg.tempAllocMem(spec.type), true },
                .mem_of_type => |ref| .{ try cg.tempAllocMem(ref.typeOf(s)), true },
                .smin_mem, .smax_mem, .umin_mem, .umax_mem, .@"0x1p63_mem" => |const_spec| {
                    const zcu = pt.zcu;
                    const ip = &zcu.intern_pool;
                    const ty = if (const_spec.ref == .none) spec.type else const_spec.ref.typeOf(s);
                    const vector_len, const scalar_ty: Type = switch (ip.indexToKey(ty.toIntern())) {
                        else => .{ null, ty },
                        .vector_type => |vector_type| .{ vector_type.len, .fromInterned(vector_type.child) },
                    };
                    const res_vector_len: ?u32 = if (const_spec.vectorize_to) |vectorize_to| switch (vectorize_to) {
                        .none => null,
                        else => @intCast(@divExact(@divExact(vectorize_to.bitSize(cg.target), 8), scalar_ty.abiSize(pt.zcu))),
                    } else vector_len;
                    const res_scalar_ty, const res_scalar_val: Value = res_scalar: switch (scalar_ty.toIntern()) {
                        .bool_type => .{
                            scalar_ty,
                            .fromInterned(switch (spec.kind) {
                                else => unreachable,
                                .smin_mem, .umax_mem => .bool_true,
                                .smax_mem, .umin_mem => .bool_false,
                            }),
                        },
                        else => {
                            const scalar_info: std.builtin.Type.Int = cg.intInfo(scalar_ty) orelse .{
                                .signedness = .signed,
                                .bits = cg.floatBits(scalar_ty).?,
                            };
                            const scalar_signedness = const_spec.to_signedness orelse scalar_info.signedness;
                            const scalar_int_ty = try pt.intType(scalar_signedness, scalar_info.bits);

                            if (scalar_info.bits <= 64) {
                                const int_val: i64 = switch (spec.kind) {
                                    else => unreachable,
                                    .smin_mem => std.math.minInt(i64),
                                    .smax_mem => std.math.maxInt(i64),
                                    .umin_mem => 0,
                                    .umax_mem => -1,
                                    .@"0x1p63_mem" => switch (scalar_info.bits) {
                                        else => unreachable,
                                        16 => @as(i64, @as(i16, @bitCast(@as(f16, 0x1p63)))) << 64 - 16,
                                        32 => @as(i64, @as(i32, @bitCast(@as(f32, 0x1p63)))) << 64 - 32,
                                        64 => @as(i64, @as(i64, @bitCast(@as(f64, 0x1p63)))) << 64 - 64,
                                    },
                                };
                                const shift: u6 = @intCast(64 - scalar_info.bits);
                                break :res_scalar .{ scalar_int_ty, switch (scalar_signedness) {
                                    .signed => try pt.intValue_i64(scalar_int_ty, int_val >> shift),
                                    .unsigned => try pt.intValue_u64(scalar_int_ty, @as(u64, @bitCast(int_val)) >> shift),
                                } };
                            }

                            const ExpectedContents = [std.math.big.int.calcTwosCompLimbCount(1 << 10)]std.math.big.Limb;
                            var stack align(@max(@alignOf(ExpectedContents), @alignOf(std.heap.StackFallbackAllocator(0)))) =
                                std.heap.stackFallback(@sizeOf(ExpectedContents), cg.gpa);
                            const allocator = stack.get();
                            var big_int: std.math.big.int.Mutable = .{
                                .limbs = try allocator.alloc(
                                    std.math.big.Limb,
                                    std.math.big.int.calcTwosCompLimbCount(scalar_info.bits),
                                ),
                                .len = undefined,
                                .positive = undefined,
                            };
                            defer allocator.free(big_int.limbs);
                            switch (spec.kind) {
                                else => unreachable,
                                .smin_mem, .smax_mem, .umin_mem, .umax_mem => big_int.setTwosCompIntLimit(switch (spec.kind) {
                                    else => unreachable,
                                    .smin_mem, .umin_mem => .min,
                                    .smax_mem, .umax_mem => .max,
                                }, switch (spec.kind) {
                                    else => unreachable,
                                    .smin_mem, .smax_mem => .signed,
                                    .umin_mem, .umax_mem => .unsigned,
                                }, scalar_info.bits),
                                .@"0x1p63_mem" => switch (scalar_info.bits) {
                                    else => unreachable,
                                    80 => big_int.set(@as(u80, @bitCast(@as(f80, 0x1p63)))),
                                    128 => big_int.set(@as(u128, @bitCast(@as(f128, 0x1p63)))),
                                },
                            }
                            big_int.truncate(big_int.toConst(), scalar_signedness, scalar_info.bits);
                            break :res_scalar .{ scalar_int_ty, try pt.intValue_big(scalar_int_ty, big_int.toConst()) };
                        },
                    };
                    const res_val: Value = if (res_vector_len) |len| .fromInterned(try pt.intern(.{ .aggregate = .{
                        .ty = (try pt.vectorType(.{
                            .len = len,
                            .child = res_scalar_ty.toIntern(),
                        })).toIntern(),
                        .storage = .{ .repeated_elem = res_scalar_val.toIntern() },
                    } })) else res_scalar_val;
                    return .{ try cg.tempMemFromValue(res_val), true };
                },
                .f64_0x1p52_0x1p84_mem => .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                    .ty = (try pt.vectorType(.{ .len = 2, .child = .f64_type })).toIntern(),
                    .storage = .{ .elems = &.{
                        (try pt.floatValue(.f64, @as(f64, 0x1p52))).toIntern(),
                        (try pt.floatValue(.f64, @as(f64, 0x1p84))).toIntern(),
                    } },
                } }))), true },
                .u32_0x1p52_hi_0x1p84_hi_0_0_mem => .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                    .ty = (try pt.vectorType(.{ .len = 4, .child = .u32_type })).toIntern(),
                    .storage = .{ .elems = &(.{
                        (try pt.intValue(.u32, @as(u64, @bitCast(@as(f64, 0x1p52))) >> 32)).toIntern(),
                        (try pt.intValue(.u32, @as(u64, @bitCast(@as(f64, 0x1p84))) >> 32)).toIntern(),
                    } ++ .{(try pt.intValue(.u32, 0)).toIntern()} ** 2) },
                } }))), true },
                .f32_0_0x1p64_mem => .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                    .ty = (try pt.vectorType(.{ .len = 2, .child = .f32_type })).toIntern(),
                    .storage = .{ .elems = &.{
                        (try pt.floatValue(.f32, @as(f32, 0))).toIntern(),
                        (try pt.floatValue(.f32, @as(f32, 0x1p64))).toIntern(),
                    } },
                } }))), true },
                .pshufb_trunc_mem => |trunc_spec| {
                    const zcu = pt.zcu;
                    assert(spec.type.isVector(zcu));
                    assert(spec.type.childType(zcu).toIntern() == .u8_type);
                    var bytes: [16]u8 = @splat(1 << 7);
                    const from_bytes: u32 = @intCast(@divExact(trunc_spec.from.bitSize(cg.target), 8));
                    const to_bytes: u32 = @intCast(@divExact(trunc_spec.to.bitSize(cg.target), 8));
                    var from_index: u32 = 0;
                    var to_index: u32 = 0;
                    while (from_index < bytes.len) {
                        for (0..to_bytes) |byte_off| bytes[to_index + byte_off] = @intCast(from_index + byte_off);
                        from_index += from_bytes;
                        to_index += to_bytes;
                    }
                    const elems = bytes[0..spec.type.vectorLen(zcu)];
                    return .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                        .ty = spec.type.toIntern(),
                        .storage = .{ .bytes = try zcu.intern_pool.getOrPutString(zcu.gpa, pt.tid, elems, .maybe_embedded_nulls) },
                    } }))), true };
                },
                .pshufb_bswap_mem => |bswap_spec| {
                    const zcu = pt.zcu;
                    assert(spec.type.isVector(zcu));
                    assert(spec.type.childType(zcu).toIntern() == .u8_type);
                    var bytes: [32]u8 = @splat(1 << 7);
                    const len: usize = @intCast(@divExact(bswap_spec.size.bitSize(cg.target), 8));
                    var to_index: u32 = 0;
                    for (0..bswap_spec.repeat) |_| for (0..len) |from_index| {
                        @memset(bytes[to_index..][0..bswap_spec.smear], @intCast(len - 1 - from_index));
                        to_index += bswap_spec.smear;
                    };
                    const elems = bytes[0..spec.type.vectorLen(zcu)];
                    return .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                        .ty = spec.type.toIntern(),
                        .storage = .{ .bytes = try zcu.intern_pool.getOrPutString(zcu.gpa, pt.tid, elems, .maybe_embedded_nulls) },
                    } }))), true };
                },
                .forward_bits, .reverse_bits => {
                    const zcu = pt.zcu;
                    assert(spec.type.isVector(zcu));
                    assert(spec.type.childType(zcu).toIntern() == .u8_type);
                    var bytes: [32]u8 = undefined;
                    const elems = bytes[0..spec.type.vectorLen(zcu)];
                    for (elems, 0..) |*elem, index| elem.* = switch (spec.kind) {
                        else => unreachable,
                        .forward_bits => @as(u8, 1 << 0) << @truncate(index),
                        .reverse_bits => @as(u8, 1 << 7) >> @truncate(index),
                    };
                    return .{ try cg.tempMemFromValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                        .ty = spec.type.toIntern(),
                        .storage = .{ .bytes = try zcu.intern_pool.getOrPutString(zcu.gpa, pt.tid, elems, .maybe_embedded_nulls) },
                    } }))), true };
                },
                .frame => |frame_index| .{ try cg.tempInit(spec.type, .{ .load_frame = .{ .index = frame_index } }), true },
                .lazy_symbol => |lazy_symbol_spec| {
                    const ty = if (lazy_symbol_spec.ref == .none) spec.type else lazy_symbol_spec.ref.typeOf(s);
                    const lazy_symbol: link.File.LazySymbol = .{
                        .kind = lazy_symbol_spec.kind,
                        .ty = ty.toIntern(),
                    };
                    return .{ try cg.tempInit(.usize, .{ .lea_symbol = .{
                        .sym_index = if (cg.bin_file.cast(.elf)) |elf_file|
                            elf_file.zigObjectPtr().?.getOrCreateMetadataForLazySymbol(elf_file, pt, lazy_symbol) catch |err|
                                return cg.fail("{s} creating lazy symbol", .{@errorName(err)})
                        else if (cg.bin_file.cast(.macho)) |macho_file|
                            macho_file.getZigObject().?.getOrCreateMetadataForLazySymbol(macho_file, pt, lazy_symbol) catch |err|
                                return cg.fail("{s} creating lazy symbol", .{@errorName(err)})
                        else if (cg.bin_file.cast(.coff)) |coff_file|
                            coff_file.getAtom(coff_file.getOrCreateAtomForLazySymbol(pt, lazy_symbol) catch |err|
                                return cg.fail("{s} creating lazy symbol", .{@errorName(err)})).getSymbolIndex().?
                        else
                            return cg.fail("external symbols unimplemented for {s}", .{@tagName(cg.bin_file.tag)}),
                    } }), true };
                },
                .symbol => |symbol_spec| .{ try cg.tempInit(spec.type, .{ .lea_symbol = .{
                    .sym_index = if (cg.bin_file.cast(.elf)) |elf_file|
                        try elf_file.getGlobalSymbol(symbol_spec.name, symbol_spec.lib)
                    else if (cg.bin_file.cast(.macho)) |macho_file|
                        try macho_file.getGlobalSymbol(symbol_spec.name, symbol_spec.lib)
                    else if (cg.bin_file.cast(.coff)) |coff_file|
                        link.File.Coff.global_symbol_bit | try coff_file.getGlobalSymbol(symbol_spec.name, symbol_spec.lib)
                    else
                        return cg.fail("external symbols unimplemented for {s}", .{@tagName(cg.bin_file.tag)}),
                } }), true },
            };
        }
    };

    const Instruction = struct {
        Label,
        Mir.Inst.Fixes,
        Mir.Inst.Tag,
        Select.Operand,
        Select.Operand,
        Select.Operand,
        Select.Operand,
    };
    const Label = enum { @"0:", @"1:", @"2:", @"3:", @"4:", @"_", pseudo };
    const Operand = struct {
        flags: packed struct(u32) {
            tag: Tag,
            adjust: Adjust = .none,
            base: Ref.Sized = .none,
            index: packed struct(u7) {
                ref: Ref,
                scale: Memory.Scale = .@"1",
            } = .{ .ref = .none },
            unused: u3 = 0,
        },
        imm: i32 = 0,

        const Tag = enum(u3) {
            none,
            backward_label,
            forward_label,
            ref,
            simm,
            uimm,
            lea,
            mem,
        };
        const Adjust = packed struct(u10) {
            sign: enum(u1) { neg, pos },
            lhs: enum(u5) {
                none,
                ptr_size,
                ptr_bit_size,
                size,
                src0_size,
                delta_size,
                delta_elem_size,
                unaligned_size,
                unaligned_size_add_elem_size,
                unaligned_size_sub_elem_size,
                bit_size,
                src0_bit_size,
                @"8_size_sub_bit_size",
                len,
                elem_limbs,
                elem_size,
                src0_elem_size,
                dst0_elem_size,
                src0_elem_size_mul_src1,
                src1,
                log2_src0_elem_size,
                smin,
                smax,
                umax,
                repeat,
            },
            op: enum(u2) { mul, div, div_8_down, rem_8_mul },
            rhs: Memory.Scale,

            const none: Adjust = .{ .sign = .pos, .lhs = .none, .op = .mul, .rhs = .@"1" };
            const sub_ptr_size: Adjust = .{ .sign = .neg, .lhs = .ptr_size, .op = .mul, .rhs = .@"1" };
            const add_ptr_bit_size: Adjust = .{ .sign = .pos, .lhs = .ptr_bit_size, .op = .mul, .rhs = .@"1" };
            const add_8_size: Adjust = .{ .sign = .pos, .lhs = .size, .op = .mul, .rhs = .@"8" };
            const add_size: Adjust = .{ .sign = .pos, .lhs = .size, .op = .mul, .rhs = .@"1" };
            const add_size_div_4: Adjust = .{ .sign = .pos, .lhs = .size, .op = .div, .rhs = .@"4" };
            const add_size_div_8: Adjust = .{ .sign = .pos, .lhs = .size, .op = .div, .rhs = .@"8" };
            const sub_size_div_8: Adjust = .{ .sign = .neg, .lhs = .size, .op = .div, .rhs = .@"8" };
            const sub_size_div_4: Adjust = .{ .sign = .neg, .lhs = .size, .op = .div, .rhs = .@"4" };
            const sub_size: Adjust = .{ .sign = .neg, .lhs = .size, .op = .mul, .rhs = .@"1" };
            const sub_src0_size_div_8: Adjust = .{ .sign = .neg, .lhs = .src0_size, .op = .div, .rhs = .@"8" };
            const sub_src0_size: Adjust = .{ .sign = .neg, .lhs = .src0_size, .op = .mul, .rhs = .@"1" };
            const add_8_src0_size: Adjust = .{ .sign = .pos, .lhs = .src0_size, .op = .mul, .rhs = .@"8" };
            const add_delta_size_div_8: Adjust = .{ .sign = .pos, .lhs = .delta_size, .op = .div, .rhs = .@"8" };
            const add_delta_elem_size: Adjust = .{ .sign = .pos, .lhs = .delta_elem_size, .op = .mul, .rhs = .@"1" };
            const add_delta_elem_size_div_8: Adjust = .{ .sign = .pos, .lhs = .delta_elem_size, .op = .div, .rhs = .@"8" };
            const add_unaligned_size: Adjust = .{ .sign = .pos, .lhs = .unaligned_size, .op = .mul, .rhs = .@"1" };
            const sub_unaligned_size: Adjust = .{ .sign = .neg, .lhs = .unaligned_size, .op = .mul, .rhs = .@"1" };
            const add_unaligned_size_add_elem_size: Adjust = .{ .sign = .pos, .lhs = .unaligned_size_add_elem_size, .op = .mul, .rhs = .@"1" };
            const add_unaligned_size_sub_elem_size: Adjust = .{ .sign = .pos, .lhs = .unaligned_size_sub_elem_size, .op = .mul, .rhs = .@"1" };
            const add_2_bit_size: Adjust = .{ .sign = .pos, .lhs = .bit_size, .op = .mul, .rhs = .@"2" };
            const add_bit_size: Adjust = .{ .sign = .pos, .lhs = .bit_size, .op = .mul, .rhs = .@"1" };
            const add_bit_size_rem_64: Adjust = .{ .sign = .pos, .lhs = .bit_size, .op = .rem_8_mul, .rhs = .@"8" };
            const sub_bit_size_rem_64: Adjust = .{ .sign = .neg, .lhs = .bit_size, .op = .rem_8_mul, .rhs = .@"8" };
            const sub_bit_size: Adjust = .{ .sign = .neg, .lhs = .bit_size, .op = .mul, .rhs = .@"1" };
            const add_src0_bit_size: Adjust = .{ .sign = .pos, .lhs = .src0_bit_size, .op = .mul, .rhs = .@"1" };
            const sub_src0_bit_size: Adjust = .{ .sign = .neg, .lhs = .src0_bit_size, .op = .mul, .rhs = .@"1" };
            const add_bit_size_sub_8_size: Adjust = .{ .sign = .neg, .lhs = .@"8_size_sub_bit_size", .op = .mul, .rhs = .@"1" };
            const add_8_len: Adjust = .{ .sign = .pos, .lhs = .len, .op = .mul, .rhs = .@"8" };
            const add_4_len: Adjust = .{ .sign = .pos, .lhs = .len, .op = .mul, .rhs = .@"4" };
            const add_3_len: Adjust = .{ .sign = .pos, .lhs = .len, .op = .mul, .rhs = .@"3" };
            const add_2_len: Adjust = .{ .sign = .pos, .lhs = .len, .op = .mul, .rhs = .@"2" };
            const add_len: Adjust = .{ .sign = .pos, .lhs = .len, .op = .mul, .rhs = .@"1" };
            const sub_len: Adjust = .{ .sign = .neg, .lhs = .len, .op = .mul, .rhs = .@"1" };
            const add_8_elem_size: Adjust = .{ .sign = .pos, .lhs = .elem_size, .op = .mul, .rhs = .@"8" };
            const add_elem_size: Adjust = .{ .sign = .pos, .lhs = .elem_size, .op = .mul, .rhs = .@"1" };
            const add_elem_size_div_8: Adjust = .{ .sign = .pos, .lhs = .elem_size, .op = .div, .rhs = .@"8" };
            const sub_elem_size_div_8: Adjust = .{ .sign = .neg, .lhs = .elem_size, .op = .div, .rhs = .@"8" };
            const sub_elem_size_div_4: Adjust = .{ .sign = .neg, .lhs = .elem_size, .op = .div, .rhs = .@"4" };
            const add_src0_elem_size: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size, .op = .mul, .rhs = .@"1" };
            const add_2_src0_elem_size: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size, .op = .mul, .rhs = .@"2" };
            const add_4_src0_elem_size: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size, .op = .mul, .rhs = .@"4" };
            const add_8_src0_elem_size: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size, .op = .mul, .rhs = .@"8" };
            const add_src0_elem_size_div_8: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size, .op = .div, .rhs = .@"8" };
            const sub_src0_elem_size_div_8: Adjust = .{ .sign = .neg, .lhs = .src0_elem_size, .op = .div, .rhs = .@"8" };
            const sub_src0_elem_size: Adjust = .{ .sign = .neg, .lhs = .src0_elem_size, .op = .mul, .rhs = .@"1" };
            const add_src0_elem_size_mul_src1: Adjust = .{ .sign = .pos, .lhs = .src0_elem_size_mul_src1, .op = .mul, .rhs = .@"1" };
            const sub_src0_elem_size_mul_src1: Adjust = .{ .sign = .neg, .lhs = .src0_elem_size_mul_src1, .op = .mul, .rhs = .@"1" };
            const add_src1_div_8_down_4: Adjust = .{ .sign = .pos, .lhs = .src1, .op = .div_8_down, .rhs = .@"4" };
            const add_src1_rem_32: Adjust = .{ .sign = .pos, .lhs = .src1, .op = .rem_8_mul, .rhs = .@"4" };
            const add_src1_rem_64: Adjust = .{ .sign = .pos, .lhs = .src1, .op = .rem_8_mul, .rhs = .@"8" };
            const add_log2_src0_elem_size: Adjust = .{ .sign = .pos, .lhs = .log2_src0_elem_size, .op = .mul, .rhs = .@"1" };
            const add_dst0_elem_size: Adjust = .{ .sign = .pos, .lhs = .dst0_elem_size, .op = .mul, .rhs = .@"1" };
            const add_elem_limbs: Adjust = .{ .sign = .pos, .lhs = .elem_limbs, .op = .mul, .rhs = .@"1" };
            const add_smin: Adjust = .{ .sign = .pos, .lhs = .smin, .op = .mul, .rhs = .@"1" };
            const add_umax: Adjust = .{ .sign = .pos, .lhs = .umax, .op = .mul, .rhs = .@"1" };
            const repeat: Adjust = .{ .sign = .pos, .lhs = .repeat, .op = .mul, .rhs = .@"1" };
        };
        const Ref = enum(u5) {
            tmp0,
            tmp1,
            tmp2,
            tmp3,
            tmp4,
            tmp5,
            tmp6,
            tmp7,
            tmp8,
            tmp9,
            tmp10,
            dst0,
            dst1,
            src0,
            src1,
            src2,
            none,

            const Sized = packed struct(u9) {
                ref: Ref,
                size: Memory.Size,

                const none: Sized = .{ .ref = .none, .size = .none };

                const tmp0: Sized = .{ .ref = .tmp0, .size = .none };
                const tmp0b: Sized = .{ .ref = .tmp0, .size = .byte };
                const tmp0w: Sized = .{ .ref = .tmp0, .size = .word };
                const tmp0d: Sized = .{ .ref = .tmp0, .size = .dword };
                const tmp0p: Sized = .{ .ref = .tmp0, .size = .ptr };
                const tmp0g: Sized = .{ .ref = .tmp0, .size = .gpr };
                const tmp0q: Sized = .{ .ref = .tmp0, .size = .qword };
                const tmp0t: Sized = .{ .ref = .tmp0, .size = .tbyte };
                const tmp0x: Sized = .{ .ref = .tmp0, .size = .xword };
                const tmp0y: Sized = .{ .ref = .tmp0, .size = .yword };

                const tmp1: Sized = .{ .ref = .tmp1, .size = .none };
                const tmp1b: Sized = .{ .ref = .tmp1, .size = .byte };
                const tmp1w: Sized = .{ .ref = .tmp1, .size = .word };
                const tmp1d: Sized = .{ .ref = .tmp1, .size = .dword };
                const tmp1p: Sized = .{ .ref = .tmp1, .size = .ptr };
                const tmp1g: Sized = .{ .ref = .tmp1, .size = .gpr };
                const tmp1q: Sized = .{ .ref = .tmp1, .size = .qword };
                const tmp1t: Sized = .{ .ref = .tmp1, .size = .tbyte };
                const tmp1x: Sized = .{ .ref = .tmp1, .size = .xword };
                const tmp1y: Sized = .{ .ref = .tmp1, .size = .yword };

                const tmp2: Sized = .{ .ref = .tmp2, .size = .none };
                const tmp2b: Sized = .{ .ref = .tmp2, .size = .byte };
                const tmp2w: Sized = .{ .ref = .tmp2, .size = .word };
                const tmp2d: Sized = .{ .ref = .tmp2, .size = .dword };
                const tmp2p: Sized = .{ .ref = .tmp2, .size = .ptr };
                const tmp2g: Sized = .{ .ref = .tmp2, .size = .gpr };
                const tmp2q: Sized = .{ .ref = .tmp2, .size = .qword };
                const tmp2t: Sized = .{ .ref = .tmp2, .size = .tbyte };
                const tmp2x: Sized = .{ .ref = .tmp2, .size = .xword };
                const tmp2y: Sized = .{ .ref = .tmp2, .size = .yword };

                const tmp3: Sized = .{ .ref = .tmp3, .size = .none };
                const tmp3b: Sized = .{ .ref = .tmp3, .size = .byte };
                const tmp3w: Sized = .{ .ref = .tmp3, .size = .word };
                const tmp3d: Sized = .{ .ref = .tmp3, .size = .dword };
                const tmp3p: Sized = .{ .ref = .tmp3, .size = .ptr };
                const tmp3g: Sized = .{ .ref = .tmp3, .size = .gpr };
                const tmp3q: Sized = .{ .ref = .tmp3, .size = .qword };
                const tmp3t: Sized = .{ .ref = .tmp3, .size = .tbyte };
                const tmp3x: Sized = .{ .ref = .tmp3, .size = .xword };
                const tmp3y: Sized = .{ .ref = .tmp3, .size = .yword };

                const tmp4: Sized = .{ .ref = .tmp4, .size = .none };
                const tmp4b: Sized = .{ .ref = .tmp4, .size = .byte };
                const tmp4w: Sized = .{ .ref = .tmp4, .size = .word };
                const tmp4d: Sized = .{ .ref = .tmp4, .size = .dword };
                const tmp4p: Sized = .{ .ref = .tmp4, .size = .ptr };
                const tmp4g: Sized = .{ .ref = .tmp4, .size = .gpr };
                const tmp4q: Sized = .{ .ref = .tmp4, .size = .qword };
                const tmp4t: Sized = .{ .ref = .tmp4, .size = .tbyte };
                const tmp4x: Sized = .{ .ref = .tmp4, .size = .xword };
                const tmp4y: Sized = .{ .ref = .tmp4, .size = .yword };

                const tmp5: Sized = .{ .ref = .tmp5, .size = .none };
                const tmp5b: Sized = .{ .ref = .tmp5, .size = .byte };
                const tmp5w: Sized = .{ .ref = .tmp5, .size = .word };
                const tmp5d: Sized = .{ .ref = .tmp5, .size = .dword };
                const tmp5p: Sized = .{ .ref = .tmp5, .size = .ptr };
                const tmp5g: Sized = .{ .ref = .tmp5, .size = .gpr };
                const tmp5q: Sized = .{ .ref = .tmp5, .size = .qword };
                const tmp5t: Sized = .{ .ref = .tmp5, .size = .tbyte };
                const tmp5x: Sized = .{ .ref = .tmp5, .size = .xword };
                const tmp5y: Sized = .{ .ref = .tmp5, .size = .yword };

                const tmp6: Sized = .{ .ref = .tmp6, .size = .none };
                const tmp6b: Sized = .{ .ref = .tmp6, .size = .byte };
                const tmp6w: Sized = .{ .ref = .tmp6, .size = .word };
                const tmp6d: Sized = .{ .ref = .tmp6, .size = .dword };
                const tmp6p: Sized = .{ .ref = .tmp6, .size = .ptr };
                const tmp6g: Sized = .{ .ref = .tmp6, .size = .gpr };
                const tmp6q: Sized = .{ .ref = .tmp6, .size = .qword };
                const tmp6t: Sized = .{ .ref = .tmp6, .size = .tbyte };
                const tmp6x: Sized = .{ .ref = .tmp6, .size = .xword };
                const tmp6y: Sized = .{ .ref = .tmp6, .size = .yword };

                const tmp7: Sized = .{ .ref = .tmp7, .size = .none };
                const tmp7b: Sized = .{ .ref = .tmp7, .size = .byte };
                const tmp7w: Sized = .{ .ref = .tmp7, .size = .word };
                const tmp7d: Sized = .{ .ref = .tmp7, .size = .dword };
                const tmp7p: Sized = .{ .ref = .tmp7, .size = .ptr };
                const tmp7g: Sized = .{ .ref = .tmp7, .size = .gpr };
                const tmp7q: Sized = .{ .ref = .tmp7, .size = .qword };
                const tmp7t: Sized = .{ .ref = .tmp7, .size = .tbyte };
                const tmp7x: Sized = .{ .ref = .tmp7, .size = .xword };
                const tmp7y: Sized = .{ .ref = .tmp7, .size = .yword };

                const tmp8: Sized = .{ .ref = .tmp8, .size = .none };
                const tmp8b: Sized = .{ .ref = .tmp8, .size = .byte };
                const tmp8w: Sized = .{ .ref = .tmp8, .size = .word };
                const tmp8d: Sized = .{ .ref = .tmp8, .size = .dword };
                const tmp8p: Sized = .{ .ref = .tmp8, .size = .ptr };
                const tmp8g: Sized = .{ .ref = .tmp8, .size = .gpr };
                const tmp8q: Sized = .{ .ref = .tmp8, .size = .qword };
                const tmp8t: Sized = .{ .ref = .tmp8, .size = .tbyte };
                const tmp8x: Sized = .{ .ref = .tmp8, .size = .xword };
                const tmp8y: Sized = .{ .ref = .tmp8, .size = .yword };

                const tmp9: Sized = .{ .ref = .tmp9, .size = .none };
                const tmp9b: Sized = .{ .ref = .tmp9, .size = .byte };
                const tmp9w: Sized = .{ .ref = .tmp9, .size = .word };
                const tmp9d: Sized = .{ .ref = .tmp9, .size = .dword };
                const tmp9p: Sized = .{ .ref = .tmp9, .size = .ptr };
                const tmp9g: Sized = .{ .ref = .tmp9, .size = .gpr };
                const tmp9q: Sized = .{ .ref = .tmp9, .size = .qword };
                const tmp9t: Sized = .{ .ref = .tmp9, .size = .tbyte };
                const tmp9x: Sized = .{ .ref = .tmp9, .size = .xword };
                const tmp9y: Sized = .{ .ref = .tmp9, .size = .yword };

                const tmp10: Sized = .{ .ref = .tmp10, .size = .none };
                const tmp10b: Sized = .{ .ref = .tmp10, .size = .byte };
                const tmp10w: Sized = .{ .ref = .tmp10, .size = .word };
                const tmp10d: Sized = .{ .ref = .tmp10, .size = .dword };
                const tmp10p: Sized = .{ .ref = .tmp10, .size = .ptr };
                const tmp10g: Sized = .{ .ref = .tmp10, .size = .gpr };
                const tmp10q: Sized = .{ .ref = .tmp10, .size = .qword };
                const tmp10t: Sized = .{ .ref = .tmp10, .size = .tbyte };
                const tmp10x: Sized = .{ .ref = .tmp10, .size = .xword };
                const tmp10y: Sized = .{ .ref = .tmp10, .size = .yword };

                const dst0: Sized = .{ .ref = .dst0, .size = .none };
                const dst0b: Sized = .{ .ref = .dst0, .size = .byte };
                const dst0w: Sized = .{ .ref = .dst0, .size = .word };
                const dst0d: Sized = .{ .ref = .dst0, .size = .dword };
                const dst0p: Sized = .{ .ref = .dst0, .size = .ptr };
                const dst0g: Sized = .{ .ref = .dst0, .size = .gpr };
                const dst0q: Sized = .{ .ref = .dst0, .size = .qword };
                const dst0t: Sized = .{ .ref = .dst0, .size = .tbyte };
                const dst0x: Sized = .{ .ref = .dst0, .size = .xword };
                const dst0y: Sized = .{ .ref = .dst0, .size = .yword };

                const dst1: Sized = .{ .ref = .dst1, .size = .none };
                const dst1b: Sized = .{ .ref = .dst1, .size = .byte };
                const dst1w: Sized = .{ .ref = .dst1, .size = .word };
                const dst1d: Sized = .{ .ref = .dst1, .size = .dword };
                const dst1p: Sized = .{ .ref = .dst1, .size = .ptr };
                const dst1g: Sized = .{ .ref = .dst1, .size = .gpr };
                const dst1q: Sized = .{ .ref = .dst1, .size = .qword };
                const dst1t: Sized = .{ .ref = .dst1, .size = .tbyte };
                const dst1x: Sized = .{ .ref = .dst1, .size = .xword };
                const dst1y: Sized = .{ .ref = .dst1, .size = .yword };

                const src0: Sized = .{ .ref = .src0, .size = .none };
                const src0b: Sized = .{ .ref = .src0, .size = .byte };
                const src0w: Sized = .{ .ref = .src0, .size = .word };
                const src0d: Sized = .{ .ref = .src0, .size = .dword };
                const src0p: Sized = .{ .ref = .src0, .size = .ptr };
                const src0g: Sized = .{ .ref = .src0, .size = .gpr };
                const src0q: Sized = .{ .ref = .src0, .size = .qword };
                const src0t: Sized = .{ .ref = .src0, .size = .tbyte };
                const src0x: Sized = .{ .ref = .src0, .size = .xword };
                const src0y: Sized = .{ .ref = .src0, .size = .yword };

                const src1: Sized = .{ .ref = .src1, .size = .none };
                const src1b: Sized = .{ .ref = .src1, .size = .byte };
                const src1w: Sized = .{ .ref = .src1, .size = .word };
                const src1d: Sized = .{ .ref = .src1, .size = .dword };
                const src1p: Sized = .{ .ref = .src1, .size = .ptr };
                const src1g: Sized = .{ .ref = .src1, .size = .gpr };
                const src1q: Sized = .{ .ref = .src1, .size = .qword };
                const src1t: Sized = .{ .ref = .src1, .size = .tbyte };
                const src1x: Sized = .{ .ref = .src1, .size = .xword };
                const src1y: Sized = .{ .ref = .src1, .size = .yword };

                const src2: Sized = .{ .ref = .src2, .size = .none };
                const src2b: Sized = .{ .ref = .src2, .size = .byte };
                const src2w: Sized = .{ .ref = .src2, .size = .word };
                const src2d: Sized = .{ .ref = .src2, .size = .dword };
                const src2p: Sized = .{ .ref = .src2, .size = .ptr };
                const src2g: Sized = .{ .ref = .src2, .size = .gpr };
                const src2q: Sized = .{ .ref = .src2, .size = .qword };
                const src2t: Sized = .{ .ref = .src2, .size = .tbyte };
                const src2x: Sized = .{ .ref = .src2, .size = .xword };
                const src2y: Sized = .{ .ref = .src2, .size = .yword };
            };

            fn typeOf(ref: Ref, s: *const Select) Type {
                return s.types[@intFromEnum(ref)];
            }

            fn tempOf(ref: Ref, s: *const Select) Temp {
                return s.temps[@intFromEnum(ref)];
            }

            fn valueOf(ref: Ref, s: *const Select) MCValue {
                return s.temps[@intFromEnum(ref)].tracking(s.cg).short;
            }
        };

        const @"_": Select.Operand = .{ .flags = .{ .tag = .none } };

        const @"0b": Select.Operand = .{ .flags = .{ .tag = .backward_label, .base = .{ .ref = .tmp0, .size = .none } } };
        const @"0f": Select.Operand = .{ .flags = .{ .tag = .forward_label, .base = .{ .ref = .tmp0, .size = .none } } };
        const @"1b": Select.Operand = .{ .flags = .{ .tag = .backward_label, .base = .{ .ref = .tmp1, .size = .none } } };
        const @"1f": Select.Operand = .{ .flags = .{ .tag = .forward_label, .base = .{ .ref = .tmp1, .size = .none } } };
        const @"2b": Select.Operand = .{ .flags = .{ .tag = .backward_label, .base = .{ .ref = .tmp2, .size = .none } } };
        const @"2f": Select.Operand = .{ .flags = .{ .tag = .forward_label, .base = .{ .ref = .tmp2, .size = .none } } };
        const @"3b": Select.Operand = .{ .flags = .{ .tag = .backward_label, .base = .{ .ref = .tmp3, .size = .none } } };
        const @"3f": Select.Operand = .{ .flags = .{ .tag = .forward_label, .base = .{ .ref = .tmp3, .size = .none } } };
        const @"4b": Select.Operand = .{ .flags = .{ .tag = .backward_label, .base = .{ .ref = .tmp4, .size = .none } } };
        const @"4f": Select.Operand = .{ .flags = .{ .tag = .forward_label, .base = .{ .ref = .tmp4, .size = .none } } };

        const tmp0b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0b } };
        const tmp0w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0w } };
        const tmp0d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0d } };
        const tmp0p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0p } };
        const tmp0g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0g } };
        const tmp0q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0q } };
        const tmp0t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0t } };
        const tmp0x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0x } };
        const tmp0y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp0y } };

        const tmp1b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1b } };
        const tmp1w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1w } };
        const tmp1d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1d } };
        const tmp1p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1p } };
        const tmp1g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1g } };
        const tmp1q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1q } };
        const tmp1t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1t } };
        const tmp1x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1x } };
        const tmp1y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp1y } };

        const tmp2b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2b } };
        const tmp2w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2w } };
        const tmp2d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2d } };
        const tmp2p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2p } };
        const tmp2g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2g } };
        const tmp2q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2q } };
        const tmp2t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2t } };
        const tmp2x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2x } };
        const tmp2y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp2y } };

        const tmp3b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3b } };
        const tmp3w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3w } };
        const tmp3d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3d } };
        const tmp3p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3p } };
        const tmp3g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3g } };
        const tmp3q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3q } };
        const tmp3t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3t } };
        const tmp3x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3x } };
        const tmp3y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp3y } };

        const tmp4b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4b } };
        const tmp4w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4w } };
        const tmp4d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4d } };
        const tmp4p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4p } };
        const tmp4g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4g } };
        const tmp4q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4q } };
        const tmp4t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4t } };
        const tmp4x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4x } };
        const tmp4y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp4y } };

        const tmp5b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5b } };
        const tmp5w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5w } };
        const tmp5d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5d } };
        const tmp5p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5p } };
        const tmp5g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5g } };
        const tmp5q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5q } };
        const tmp5t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5t } };
        const tmp5x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5x } };
        const tmp5y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp5y } };

        const tmp6b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6b } };
        const tmp6w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6w } };
        const tmp6d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6d } };
        const tmp6p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6p } };
        const tmp6g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6g } };
        const tmp6q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6q } };
        const tmp6t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6t } };
        const tmp6x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6x } };
        const tmp6y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp6y } };

        const tmp7b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7b } };
        const tmp7w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7w } };
        const tmp7d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7d } };
        const tmp7p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7p } };
        const tmp7g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7g } };
        const tmp7q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7q } };
        const tmp7t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7t } };
        const tmp7x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7x } };
        const tmp7y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp7y } };

        const tmp8b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8b } };
        const tmp8w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8w } };
        const tmp8d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8d } };
        const tmp8p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8p } };
        const tmp8g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8g } };
        const tmp8q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8q } };
        const tmp8t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8t } };
        const tmp8x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8x } };
        const tmp8y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp8y } };

        const tmp9b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9b } };
        const tmp9w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9w } };
        const tmp9d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9d } };
        const tmp9p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9p } };
        const tmp9g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9g } };
        const tmp9q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9q } };
        const tmp9t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9t } };
        const tmp9x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9x } };
        const tmp9y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp9y } };

        const tmp10b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10b } };
        const tmp10w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10w } };
        const tmp10d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10d } };
        const tmp10p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10p } };
        const tmp10g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10g } };
        const tmp10q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10q } };
        const tmp10t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10t } };
        const tmp10x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10x } };
        const tmp10y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .tmp10y } };

        const dst0b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0b } };
        const dst0w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0w } };
        const dst0d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0d } };
        const dst0p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0p } };
        const dst0g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0g } };
        const dst0q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0q } };
        const dst0t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0t } };
        const dst0x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0x } };
        const dst0y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst0y } };

        const dst1b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1b } };
        const dst1w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1w } };
        const dst1d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1d } };
        const dst1p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1p } };
        const dst1g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1g } };
        const dst1q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1q } };
        const dst1t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1t } };
        const dst1x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1x } };
        const dst1y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .dst1y } };

        const src0b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0b } };
        const src0w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0w } };
        const src0d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0d } };
        const src0p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0p } };
        const src0g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0g } };
        const src0q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0q } };
        const src0t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0t } };
        const src0x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0x } };
        const src0y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src0y } };

        const src1b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1b } };
        const src1w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1w } };
        const src1d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1d } };
        const src1p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1p } };
        const src1g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1g } };
        const src1q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1q } };
        const src1t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1t } };
        const src1x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1x } };
        const src1y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src1y } };

        const src2b: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2b } };
        const src2w: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2w } };
        const src2d: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2d } };
        const src2p: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2p } };
        const src2g: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2g } };
        const src2q: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2q } };
        const src2t: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2t } };
        const src2x: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2x } };
        const src2y: Select.Operand = .{ .flags = .{ .tag = .ref, .base = .src2y } };

        fn si(imm: i32) Select.Operand {
            return .{ .flags = .{ .tag = .simm }, .imm = imm };
        }
        fn sa(base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .simm, .adjust = adjust, .base = base } };
        }
        fn sa2(base: Ref.Sized, index: Ref, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .simm, .adjust = adjust, .base = base, .index = .{ .ref = index } } };
        }
        fn sia(imm: i32, base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .simm, .adjust = adjust, .base = base }, .imm = imm };
        }
        fn sia2(imm: i32, base: Ref.Sized, index: Ref, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .simm, .adjust = adjust, .base = base, .index = .{ .ref = index } }, .imm = imm };
        }
        fn ui(imm: u32) Select.Operand {
            return .{ .flags = .{ .tag = .uimm }, .imm = @bitCast(imm) };
        }
        fn ua(base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .uimm, .adjust = adjust, .base = base } };
        }
        fn uia(imm: u32, base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{ .flags = .{ .tag = .uimm, .adjust = adjust, .base = base }, .imm = @bitCast(imm) };
        }

        fn rm(mode: bits.RoundMode) Select.Operand {
            return .{ .flags = .{ .tag = .uimm }, .imm = @intCast(mode.imm().unsigned) };
        }
        fn sp(pred: bits.SseFloatPredicate) Select.Operand {
            return .{ .flags = .{ .tag = .uimm }, .imm = @intCast(pred.imm().unsigned) };
        }
        fn vp(pred: bits.VexFloatPredicate) Select.Operand {
            return .{ .flags = .{ .tag = .uimm }, .imm = @intCast(pred.imm().unsigned) };
        }

        fn lea(base: Ref.Sized) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                },
            };
        }
        fn leaa(base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                },
            };
        }
        fn leaad(base: Ref.Sized, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                },
                .imm = disp,
            };
        }
        fn lead(base: Ref.Sized, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                },
                .imm = disp,
            };
        }
        fn leai(base: Ref.Sized, index: Ref) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                    .index = .{ .ref = index },
                },
            };
        }
        fn leaia(base: Ref.Sized, index: Ref, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index },
                },
            };
        }
        fn leaiad(base: Ref.Sized, index: Ref, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index },
                },
                .imm = disp,
            };
        }
        fn leaid(base: Ref.Sized, index: Ref, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                    .index = .{ .ref = index },
                },
                .imm = disp,
            };
        }
        fn leasi(base: Ref.Sized, scale: Memory.Scale, index: Ref) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
            };
        }
        fn leasia(base: Ref.Sized, scale: Memory.Scale, index: Ref, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
            };
        }
        fn leasid(base: Ref.Sized, scale: Memory.Scale, index: Ref, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
                .imm = disp,
            };
        }
        fn leasiad(base: Ref.Sized, scale: Memory.Scale, index: Ref, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .lea,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
                .imm = disp,
            };
        }

        fn mem(base: Ref.Sized) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                },
            };
        }
        fn memd(base: Ref.Sized, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                },
                .imm = disp,
            };
        }
        fn mema(base: Ref.Sized, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                },
            };
        }
        fn memad(base: Ref.Sized, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                },
                .imm = disp,
            };
        }
        fn memi(base: Ref.Sized, index: Ref) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                    .index = .{ .ref = index },
                },
            };
        }
        fn memia(base: Ref.Sized, index: Ref, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index },
                },
            };
        }
        fn memiad(base: Ref.Sized, index: Ref, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index },
                },
                .imm = disp,
            };
        }
        fn memid(base: Ref.Sized, index: Ref, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                    .index = .{ .ref = index },
                },
                .imm = disp,
            };
        }
        fn memsi(base: Ref.Sized, scale: Memory.Scale, index: Ref) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
            };
        }
        fn memsia(base: Ref.Sized, scale: Memory.Scale, index: Ref, adjust: Adjust) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
            };
        }
        fn memsid(base: Ref.Sized, scale: Memory.Scale, index: Ref, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
                .imm = disp,
            };
        }
        fn memsiad(base: Ref.Sized, scale: Memory.Scale, index: Ref, adjust: Adjust, disp: i32) Select.Operand {
            return .{
                .flags = .{
                    .tag = .mem,
                    .adjust = adjust,
                    .base = base,
                    .index = .{ .ref = index, .scale = scale },
                },
                .imm = disp,
            };
        }

        fn adjustedImm(op: Select.Operand, comptime SignedImm: type, s: *const Select) SignedImm {
            const UnsignedImm = @Type(.{
                .int = .{ .signedness = .unsigned, .bits = @typeInfo(SignedImm).int.bits },
            });
            const lhs: SignedImm = lhs: switch (op.flags.adjust.lhs) {
                .none => 0,
                .ptr_size => @divExact(s.cg.target.ptrBitWidth(), 8),
                .ptr_bit_size => s.cg.target.ptrBitWidth(),
                .size => @intCast(op.flags.base.ref.typeOf(s).abiSize(s.cg.pt.zcu)),
                .src0_size => @intCast(Select.Operand.Ref.src0.typeOf(s).abiSize(s.cg.pt.zcu)),
                .delta_size => @intCast(@as(SignedImm, @intCast(op.flags.base.ref.typeOf(s).abiSize(s.cg.pt.zcu))) -
                    @as(SignedImm, @intCast(op.flags.index.ref.typeOf(s).abiSize(s.cg.pt.zcu)))),
                .delta_elem_size => @intCast(@as(SignedImm, @intCast(op.flags.base.ref.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu))) -
                    @as(SignedImm, @intCast(op.flags.index.ref.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu)))),
                .unaligned_size => @intCast(s.cg.unalignedSize(op.flags.base.ref.typeOf(s))),
                .unaligned_size_add_elem_size => {
                    const ty = op.flags.base.ref.typeOf(s);
                    break :lhs @intCast(s.cg.unalignedSize(ty) + ty.elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu));
                },
                .unaligned_size_sub_elem_size => {
                    const ty = op.flags.base.ref.typeOf(s);
                    break :lhs @intCast(s.cg.unalignedSize(ty) - ty.elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu));
                },
                .bit_size => @intCast(op.flags.base.ref.typeOf(s).scalarType(s.cg.pt.zcu).bitSize(s.cg.pt.zcu)),
                .src0_bit_size => @intCast(Select.Operand.Ref.src0.typeOf(s).scalarType(s.cg.pt.zcu).bitSize(s.cg.pt.zcu)),
                .@"8_size_sub_bit_size" => {
                    const ty = op.flags.base.ref.typeOf(s);
                    break :lhs @intCast(8 * ty.abiSize(s.cg.pt.zcu) - ty.bitSize(s.cg.pt.zcu));
                },
                .len => @intCast(op.flags.base.ref.typeOf(s).vectorLen(s.cg.pt.zcu)),
                .elem_limbs => @intCast(@divExact(
                    op.flags.base.ref.typeOf(s).scalarType(s.cg.pt.zcu).abiSize(s.cg.pt.zcu),
                    @divExact(op.flags.base.size.bitSize(s.cg.target), 8),
                )),
                .elem_size => @intCast(op.flags.base.ref.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu)),
                .src0_elem_size => @intCast(Select.Operand.Ref.src0.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu)),
                .dst0_elem_size => @intCast(Select.Operand.Ref.dst0.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu)),
                .src0_elem_size_mul_src1 => @intCast(Select.Operand.Ref.src0.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu) *
                    Select.Operand.Ref.src1.valueOf(s).immediate),
                .src1 => @intCast(Select.Operand.Ref.src1.valueOf(s).immediate),
                .log2_src0_elem_size => @intCast(std.math.log2(Select.Operand.Ref.src0.typeOf(s).elemType2(s.cg.pt.zcu).abiSize(s.cg.pt.zcu))),
                .smin => @as(SignedImm, std.math.minInt(SignedImm)) >> @truncate(
                    -%op.flags.base.ref.typeOf(s).scalarType(s.cg.pt.zcu).bitSize(s.cg.pt.zcu),
                ),
                .smax => @as(SignedImm, std.math.maxInt(SignedImm)) >> @truncate(
                    -%op.flags.base.ref.typeOf(s).scalarType(s.cg.pt.zcu).bitSize(s.cg.pt.zcu),
                ),
                .umax => @bitCast(@as(UnsignedImm, std.math.maxInt(UnsignedImm)) >> @truncate(
                    -%op.flags.base.ref.typeOf(s).scalarType(s.cg.pt.zcu).bitSize(s.cg.pt.zcu),
                )),
                .repeat => switch (SignedImm) {
                    else => unreachable,
                    i64 => return @as(i64, op.imm) << 32 | @as(u32, @bitCast(op.imm)),
                },
            };
            const rhs = op.flags.adjust.rhs.toLog2();
            const op_res = op_res: switch (op.flags.adjust.op) {
                .mul => {
                    const op_res = @shlWithOverflow(lhs, rhs);
                    assert(op_res[1] == 0);
                    break :op_res op_res[0];
                },
                .div => @shrExact(lhs, rhs),
                .div_8_down => lhs >> 3 & @as(SignedImm, -1) << rhs,
                .rem_8_mul => lhs & (@as(SignedImm, 1) << @intCast(@as(u3, 3) + rhs)) - 1,
            };
            const disp: SignedImm = @bitCast(@as(UnsignedImm, @as(u32, @bitCast(op.imm))));
            return switch (op.flags.adjust.sign) {
                .neg => disp - op_res,
                .pos => disp + op_res,
            };
        }

        fn lower(op: Select.Operand, s: *Select) InnerError!CodeGen.Operand {
            return switch (op.flags.tag) {
                .none => .none,
                .backward_label => .{ .inst = s.labels[@intFromEnum(op.flags.base.ref)].backward.? },
                .forward_label => for (&s.labels[@intFromEnum(op.flags.base.ref)].forward) |*label| {
                    if (label.*) |_| continue;
                    label.* = @intCast(s.cg.mir_instructions.len);
                    break .{ .inst = undefined };
                } else unreachable,
                .ref => switch (op.flags.base.ref.valueOf(s)) {
                    .immediate => |imm| .{ .imm = switch (op.flags.base.size) {
                        .byte => if (std.math.cast(i8, @as(i64, @bitCast(imm)))) |simm| .s(simm) else .u(@as(u8, @intCast(imm))),
                        .word => if (std.math.cast(i16, @as(i64, @bitCast(imm)))) |simm| .s(simm) else .u(@as(u16, @intCast(imm))),
                        .dword => if (std.math.cast(i32, @as(i64, @bitCast(imm)))) |simm| .s(simm) else .u(@as(u32, @intCast(imm))),
                        .qword => if (std.math.cast(i32, @as(i64, @bitCast(imm)))) |simm| .s(simm) else .u(imm),
                        else => unreachable,
                    } },
                    else => |mcv| .{ .mem = try mcv.mem(s.cg, .{ .size = op.flags.base.size }) },
                    .register => |reg| .{ .reg = s.lowerReg(registerAlias(reg, @intCast(@divExact(op.flags.base.size.bitSize(s.cg.target), 8)))) },
                    .lea_symbol => |sym_off| .{ .imm = .rel(sym_off) },
                },
                .simm => .{ .imm = .s(op.adjustedImm(i32, s)) },
                .uimm => .{ .imm = .u(@bitCast(op.adjustedImm(i64, s))) },
                .lea => .{ .mem = .{
                    .base = switch (op.flags.base.ref.valueOf(s)) {
                        else => unreachable,
                        .register => |base_reg| .{ .reg = registerAlias(base_reg, @divExact(s.cg.target.ptrBitWidth(), 8)) },
                        .register_offset => |base_reg_off| .{ .reg = registerAlias(base_reg_off.reg, @divExact(s.cg.target.ptrBitWidth(), 8)) },
                        .lea_symbol => |base_sym_off| .{ .reloc = base_sym_off.sym_index },
                    },
                    .mod = .{ .rm = .{
                        .size = op.flags.base.size,
                        .index = switch (op.flags.index.ref) {
                            else => |index_ref| switch (index_ref.valueOf(s)) {
                                else => unreachable,
                                .register => |index_reg| registerAlias(index_reg, @divExact(s.cg.target.ptrBitWidth(), 8)),
                                .register_offset => |index_reg_off| registerAlias(index_reg_off.reg, @divExact(s.cg.target.ptrBitWidth(), 8)),
                            },
                            .none => .none,
                        },
                        .scale = op.flags.index.scale,
                        .disp = op.adjustedImm(i32, s) + switch (op.flags.base.ref.valueOf(s)) {
                            else => unreachable,
                            .register => 0,
                            .register_offset => |base_reg_off| base_reg_off.off,
                            .lea_symbol => |base_sym_off| base_sym_off.off,
                        } + switch (op.flags.index.ref) {
                            else => |index_ref| switch (index_ref.valueOf(s)) {
                                else => unreachable,
                                .register => 0,
                                .register_offset => |base_reg_off| base_reg_off.off,
                                .lea_symbol => |base_sym_off| base_sym_off.off,
                            },
                            .none => 0,
                        },
                    } },
                } },
                .mem => .{ .mem = try op.flags.base.ref.valueOf(s).mem(s.cg, .{
                    .size = op.flags.base.size,
                    .index = switch (op.flags.index.ref) {
                        else => |index_ref| registerAlias(index_ref.valueOf(s).register, @divExact(s.cg.target.ptrBitWidth(), 8)),
                        .none => .none,
                    },
                    .scale = op.flags.index.scale,
                    .disp = op.adjustedImm(i32, s),
                }) },
            };
        }
    };
}
```
嗯，我现在需要总结这段Zig代码的思维模型。首先，我看到这段代码定义了一个名为Select的结构体，看起来像是用于代码生成或指令选择的部分。结构体内部有很多嵌套的结构体和函数，我需要理清楚它们之间的关系和各自的作用。

首先，Select结构体包含了一些成员变量，比如cg（CodeGen指针）、types、temps、labels和top。这些可能用于跟踪代码生成过程中的类型、临时变量、标签和栈顶位置。接下来有一个Error联合类型，可能用于错误处理。

接下来是emitLabel函数，它似乎处理标签的生成，包括前向和后向跳转的地址重定位。然后是emit函数，这个函数看起来负责生成具体的机器指令。里面有很多switch语句，根据不同的指令标签（如.f_, .f_1等）来处理不同的指令类型，比如算术运算、加载、存储等，同时调整栈顶指针top的值。

lowerReg函数可能用于将寄存器映射到当前栈顶的位置，特别是处理x87浮点寄存器栈。Case结构体定义了指令选择的条件，包括所需特性、源和目标的约束、模式等。Constraint联合类型定义了各种约束条件，比如整数类型、浮点类型、向量类型等的检查规则，这些规则在accepts函数中实现，用于确定某个类型是否符合特定的约束。

Pattern结构体可能用于匹配指令模式，Src联合类型定义了不同的源操作数类型，比如立即数、寄存器、内存地址等。matches函数检查临时变量是否符合特定的源操作数类型，convert函数则尝试将临时变量转换为所需的类型。

TempSpec结构体定义了临时变量的规格，包括类型、种类（如寄存器、内存、符号等），create函数根据规格创建临时变量。Instruction结构体组合了标签、MIR指令的修复、标签、操作数等信息。Operand结构体处理各种操作数的编码，包括标签引用、寄存器、内存地址等，lower函数将操作数转换为目标机器的操作数表示。

总的来说，这段代码是一个指令选择器的一部分，负责将中间表示（IR）转换为具体的机器指令。它处理不同类型的操作数，应用约束条件，生成相应的机器码，并管理临时变量和标签。需要处理各种指令模式、寄存器分配、内存访问以及指令的编码和优化。
================================================
这段代码实现了一个指令选择器（Instruction Selector），用于将中间表示（IR）转换为目标机器的具体指令（如x86）。以下是其核心思维模型的总结：

---

### **1. 核心结构体：`Select`**
- **作用**：管理指令选择过程中的上下文信息。
- **关键成员**：
  - `cg`: 代码生成器（`CodeGen`）的指针，提供底层代码生成功能。
  - `types`/`temps`: 分别存储操作数的类型和临时变量的信息。
  - `labels`: 管理标签（如跳转标签）的前向和后向引用。
  - `top`: 跟踪x87浮点寄存器栈的栈顶位置（用于处理FPU指令）。

---

### **2. 指令生成流程**
#### **`emitLabel` 函数**
- **功能**：处理标签的生成和重定位。
  - 将标签与具体指令地址绑定，处理前向跳转的占位符（`forward`）和后向跳转的地址记录（`backward`）。

#### **`emit` 函数**
- **功能**：生成机器指令的核心逻辑。
  - 解析指令的操作码（如`.f_`, `.f_1`）和操作数，生成对应的MIR（Machine IR）指令。
  - 调整x87栈顶指针（`top`）以管理浮点寄存器栈。
  - 处理特殊指令（如初始化栈、保存/恢复寄存器状态）。

---

### **3. 操作数处理**
#### **`Operand` 结构体**
- **作用**：表示指令的操作数，支持多种类型（立即数、寄存器、内存地址等）。
- **关键字段**：
  - `flags`: 编码操作数的类型（如`.simm`为有符号立即数，`.mem`为内存地址）。
  - `imm`: 存储立即数值或偏移量。
- **动态计算**：通过`adjustedImm`函数结合类型信息动态计算操作数的实际值（如调整对齐、位宽等）。

#### **`lower` 方法**
- **功能**：将抽象的操作数转换为目标机器的具体操作数表示（如寄存器编码、内存地址格式）。

---

### **4. 约束与模式匹配**
#### **`Case` 结构体**
- **作用**：定义指令选择的条件和约束。
  - `required_features`: 目标平台需支持的CPU特性（如SSE、AVX）。
  - `src_constraints`/`dst_constraints`: 源操作数和目标操作数的约束条件（如整数类型、浮点类型、内存对齐等）。

#### **`Constraint` 联合类型**
- **功能**：定义操作数的类型约束（如整数、浮点、向量类型）。
- **`accepts` 方法**：检查给定类型是否满足约束条件（如位宽、符号性、对齐）。

#### **`Pattern` 结构体**
- **作用**：匹配指令模式（如操作数类型组合）。
- **`Src` 联合类型**：定义源操作数的可能形式（如寄存器、立即数、内存地址）。
- **`matches` 方法**：检查临时变量是否符合模式要求。
- **`convert` 方法**：将临时变量转换为目标形式（如绑定到特定寄存器）。

---

### **5. 临时变量管理**
#### **`TempSpec` 结构体**
- **作用**：定义临时变量的属性和分配规则。
  - **`Kind` 联合类型**：支持多种分配策略（如寄存器、内存、符号引用）。
  - **`create` 方法**：根据规格创建临时变量（如分配寄存器或内存空间）。

---

### **6. 关键设计思想**
- **分层抽象**：通过`Select`结构体隔离指令选择逻辑与底层代码生成细节。
- **动态适配**：结合目标平台特性（如寄存器大小、指令集扩展）动态生成指令。
- **类型驱动**：基于类型约束和模式匹配确保生成的指令语义正确。
- **栈管理**：显式管理x87浮点寄存器栈，避免栈溢出或计算错误。

---

### **总结**
这段代码实现了一个复杂的指令选择器，核心目标是将高级IR转换为高效的机器指令。其设计围绕类型约束、模式匹配和动态操作数展开，同时结合底层硬件特性（如x87栈、SSE指令集）进行优化。通过分层抽象和严格的约束检查，确保了生成的代码既正确又高效。