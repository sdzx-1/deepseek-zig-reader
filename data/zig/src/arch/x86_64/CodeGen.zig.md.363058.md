```zig
fn genBinOp(
    self: *CodeGen,
    maybe_inst: ?Air.Inst.Index,
    air_tag: Air.Inst.Tag,
    lhs_air: Air.Inst.Ref,
    rhs_air: Air.Inst.Ref,
) !MCValue {
    const pt = self.pt;
    const zcu = pt.zcu;
    const lhs_ty = self.typeOf(lhs_air);
    const rhs_ty = self.typeOf(rhs_air);
    const abi_size: u32 = @intCast(lhs_ty.abiSize(zcu));

    if (lhs_ty.isRuntimeFloat()) libcall: {
        const float_bits = lhs_ty.floatBits(self.target.*);
        const type_needs_libcall = switch (float_bits) {
            16 => !self.hasFeature(.f16c),
            32, 64 => false,
            80, 128 => true,
            else => unreachable,
        };
        switch (air_tag) {
            .rem, .mod => {},
            else => if (!type_needs_libcall) break :libcall,
        }
        var callee_buf: ["__mod?f3".len]u8 = undefined;
        const callee = switch (air_tag) {
            .add,
            .sub,
            .mul,
            .div_float,
            .div_trunc,
            .div_floor,
            .div_exact,
            => std.fmt.bufPrint(&callee_buf, "__{s}{c}f3", .{
                @tagName(air_tag)[0..3],
                floatCompilerRtAbiName(float_bits),
            }),
            .rem, .mod, .min, .max => std.fmt.bufPrint(&callee_buf, "{s}f{s}{s}", .{
                floatLibcAbiPrefix(lhs_ty),
                switch (air_tag) {
                    .rem, .mod => "mod",
                    .min => "min",
                    .max => "max",
                    else => unreachable,
                },
                floatLibcAbiSuffix(lhs_ty),
            }),
            else => return self.fail("TODO implement genBinOp for {s} {}", .{
                @tagName(air_tag), lhs_ty.fmt(pt),
            }),
        } catch unreachable;
        const result = try self.genCall(.{ .lib = .{
            .return_type = lhs_ty.toIntern(),
            .param_types = &.{ lhs_ty.toIntern(), rhs_ty.toIntern() },
            .callee = callee,
        } }, &.{ lhs_ty, rhs_ty }, &.{ .{ .air_ref = lhs_air }, .{ .air_ref = rhs_air } }, .{});
        return switch (air_tag) {
            .mod => result: {
                const adjusted: MCValue = if (type_needs_libcall) adjusted: {
                    var add_callee_buf: ["__add?f3".len]u8 = undefined;
                    break :adjusted try self.genCall(.{ .lib = .{
                        .return_type = lhs_ty.toIntern(),
                        .param_types = &.{
                            lhs_ty.toIntern(),
                            rhs_ty.toIntern(),
                        },
                        .callee = std.fmt.bufPrint(&add_callee_buf, "__add{c}f3", .{
                            floatCompilerRtAbiName(float_bits),
                        }) catch unreachable,
                    } }, &.{ lhs_ty, rhs_ty }, &.{ result, .{ .air_ref = rhs_air } }, .{});
                } else switch (float_bits) {
                    16, 32, 64 => adjusted: {
                        const dst_reg = switch (result) {
                            .register => |reg| reg,
                            else => if (maybe_inst) |inst|
                                (try self.copyToRegisterWithInstTracking(inst, lhs_ty, result)).register
                            else
                                try self.copyToTmpRegister(lhs_ty, result),
                        };
                        const dst_lock = self.register_manager.lockReg(dst_reg);
                        defer if (dst_lock) |lock| self.register_manager.unlockReg(lock);

                        const rhs_mcv = try self.resolveInst(rhs_air);
                        const src_mcv: MCValue = if (float_bits == 16) src: {
                            assert(self.hasFeature(.f16c));
                            const tmp_reg = (try self.register_manager.allocReg(
                                null,
                                abi.RegisterClass.sse,
                            )).to128();
                            const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                            defer self.register_manager.unlockReg(tmp_lock);

                            if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                                .{ .vp_w, .insr },
                                dst_reg,
                                dst_reg,
                                try rhs_mcv.mem(self, .{ .size = .word }),
                                .u(1),
                            ) else try self.asmRegisterRegisterRegister(
                                .{ .vp_, .unpcklwd },
                                dst_reg,
                                dst_reg,
                                (if (rhs_mcv.isRegister())
                                    rhs_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, rhs_mcv)).to128(),
                            );
                            try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg, dst_reg);
                            try self.asmRegisterRegister(.{ .v_, .movshdup }, tmp_reg, dst_reg);
                            break :src .{ .register = tmp_reg };
                        } else rhs_mcv;

                        if (self.hasFeature(.avx)) {
                            const mir_tag: Mir.Inst.FixedTag = switch (float_bits) {
                                16, 32 => .{ .v_ss, .add },
                                64 => .{ .v_sd, .add },
                                else => unreachable,
                            };
                            if (src_mcv.isBase()) try self.asmRegisterRegisterMemory(
                                mir_tag,
                                dst_reg,
                                dst_reg,
                                try src_mcv.mem(self, .{ .size = .fromBitSize(float_bits) }),
                            ) else try self.asmRegisterRegisterRegister(
                                mir_tag,
                                dst_reg,
                                dst_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                        } else {
                            const mir_tag: Mir.Inst.FixedTag = switch (float_bits) {
                                32 => .{ ._ss, .add },
                                64 => .{ ._sd, .add },
                                else => unreachable,
                            };
                            if (src_mcv.isBase()) try self.asmRegisterMemory(
                                mir_tag,
                                dst_reg,
                                try src_mcv.mem(self, .{ .size = .fromBitSize(float_bits) }),
                            ) else try self.asmRegisterRegister(
                                mir_tag,
                                dst_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                        }

                        if (float_bits == 16) try self.asmRegisterRegisterImmediate(
                            .{ .v_, .cvtps2ph },
                            dst_reg,
                            dst_reg,
                            bits.RoundMode.imm(.{}),
                        );
                        break :adjusted .{ .register = dst_reg };
                    },
                    80, 128 => return self.fail("TODO implement genBinOp for {s} of {}", .{
                        @tagName(air_tag), lhs_ty.fmt(pt),
                    }),
                    else => unreachable,
                };
                break :result try self.genCall(.{ .lib = .{
                    .return_type = lhs_ty.toIntern(),
                    .param_types = &.{ lhs_ty.toIntern(), rhs_ty.toIntern() },
                    .callee = callee,
                } }, &.{ lhs_ty, rhs_ty }, &.{ adjusted, .{ .air_ref = rhs_air } }, .{});
            },
            .div_trunc, .div_floor => try self.genRoundLibcall(lhs_ty, result, .{
                .direction = switch (air_tag) {
                    .div_trunc => .zero,
                    .div_floor => .down,
                    else => unreachable,
                },
                .precision = .inexact,
            }),
            else => result,
        };
    }

    const sse_op = switch (lhs_ty.zigTypeTag(zcu)) {
        else => false,
        .float => true,
        .vector => switch (lhs_ty.childType(zcu).toIntern()) {
            .bool_type, .u1_type => false,
            else => true,
        },
    };
    if (sse_op and ((lhs_ty.scalarType(zcu).isRuntimeFloat() and
        lhs_ty.scalarType(zcu).floatBits(self.target.*) == 80) or
        lhs_ty.abiSize(zcu) > self.vectorSize(.float)))
        return self.fail("TODO implement genBinOp for {s} {}", .{ @tagName(air_tag), lhs_ty.fmt(pt) });

    const maybe_mask_reg = switch (air_tag) {
        else => null,
        .rem, .mod => unreachable,
        .max, .min => if (lhs_ty.scalarType(zcu).isRuntimeFloat()) registerAlias(
            if (!self.hasFeature(.avx) and self.hasFeature(.sse4_1)) mask: {
                try self.register_manager.getKnownReg(.xmm0, null);
                break :mask .xmm0;
            } else try self.register_manager.allocReg(null, abi.RegisterClass.sse),
            abi_size,
        ) else null,
    };
    const mask_lock =
        if (maybe_mask_reg) |mask_reg| self.register_manager.lockRegAssumeUnused(mask_reg) else null;
    defer if (mask_lock) |lock| self.register_manager.unlockReg(lock);

    const ordered_air: [2]Air.Inst.Ref = if (lhs_ty.isVector(zcu) and
        switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
            .bool => false,
            .int => switch (air_tag) {
                .cmp_lt, .cmp_gte => true,
                else => false,
            },
            .float => switch (air_tag) {
                .cmp_gte, .cmp_gt => true,
                else => false,
            },
            else => unreachable,
        }) .{ rhs_air, lhs_air } else .{ lhs_air, rhs_air };

    if (lhs_ty.isAbiInt(zcu)) for (ordered_air) |op_air| {
        switch (try self.resolveInst(op_air)) {
            .register => |op_reg| switch (op_reg.class()) {
                .sse => try self.register_manager.getReg(op_reg, null),
                else => {},
            },
            else => {},
        }
    };

    const lhs_mcv = try self.resolveInst(ordered_air[0]);
    var rhs_mcv = try self.resolveInst(ordered_air[1]);
    switch (lhs_mcv) {
        .immediate => |imm| switch (imm) {
            0 => switch (air_tag) {
                .sub, .sub_wrap => return self.genUnOp(maybe_inst, .neg, ordered_air[1]),
                else => {},
            },
            else => {},
        },
        else => {},
    }

    const is_commutative = switch (air_tag) {
        .add,
        .add_wrap,
        .mul,
        .bool_or,
        .bit_or,
        .bool_and,
        .bit_and,
        .xor,
        .min,
        .max,
        .cmp_eq,
        .cmp_neq,
        => true,

        else => false,
    };

    const lhs_locks: [2]?RegisterLock = switch (lhs_mcv) {
        .register => |lhs_reg| .{ self.register_manager.lockRegAssumeUnused(lhs_reg), null },
        .register_pair => |lhs_regs| locks: {
            const locks = self.register_manager.lockRegsAssumeUnused(2, lhs_regs);
            break :locks .{ locks[0], locks[1] };
        },
        else => @splat(null),
    };
    defer for (lhs_locks) |lhs_lock| if (lhs_lock) |lock| self.register_manager.unlockReg(lock);

    const rhs_locks: [2]?RegisterLock = switch (rhs_mcv) {
        .register => |rhs_reg| .{ self.register_manager.lockReg(rhs_reg), null },
        .register_pair => |rhs_regs| self.register_manager.lockRegs(2, rhs_regs),
        else => @splat(null),
    };
    defer for (rhs_locks) |rhs_lock| if (rhs_lock) |lock| self.register_manager.unlockReg(lock);

    var flipped = false;
    var copied_to_dst = true;
    const dst_mcv: MCValue = dst: {
        const tracked_inst = switch (air_tag) {
            else => maybe_inst,
            .cmp_lt, .cmp_lte, .cmp_eq, .cmp_gte, .cmp_gt, .cmp_neq => null,
        };
        if (maybe_inst) |inst| {
            if ((!sse_op or lhs_mcv.isRegister()) and
                self.reuseOperandAdvanced(inst, ordered_air[0], 0, lhs_mcv, tracked_inst))
                break :dst lhs_mcv;
            if (is_commutative and (!sse_op or rhs_mcv.isRegister()) and
                self.reuseOperandAdvanced(inst, ordered_air[1], 1, rhs_mcv, tracked_inst))
            {
                flipped = true;
                break :dst rhs_mcv;
            }
        }
        const dst_mcv = try self.allocRegOrMemAdvanced(lhs_ty, tracked_inst, true);
        if (sse_op and lhs_mcv.isRegister() and self.hasFeature(.avx))
            copied_to_dst = false
        else
            try self.genCopy(lhs_ty, dst_mcv, lhs_mcv, .{});
        rhs_mcv = try self.resolveInst(ordered_air[1]);
        break :dst dst_mcv;
    };
    const dst_locks: [2]?RegisterLock = switch (dst_mcv) {
        .register => |dst_reg| .{ self.register_manager.lockReg(dst_reg), null },
        .register_pair => |dst_regs| self.register_manager.lockRegs(2, dst_regs),
        else => @splat(null),
    };
    defer for (dst_locks) |dst_lock| if (dst_lock) |lock| self.register_manager.unlockReg(lock);

    const unmat_src_mcv = if (flipped) lhs_mcv else rhs_mcv;
    const src_mcv: MCValue = if (maybe_mask_reg) |mask_reg|
        if (self.hasFeature(.avx) and unmat_src_mcv.isRegister() and maybe_inst != null and
            self.liveness.operandDies(maybe_inst.?, if (flipped) 0 else 1)) unmat_src_mcv else src: {
            try self.genSetReg(mask_reg, rhs_ty, unmat_src_mcv, .{});
            break :src .{ .register = mask_reg };
        }
    else
        unmat_src_mcv;
    const src_locks: [2]?RegisterLock = switch (src_mcv) {
        .register => |src_reg| .{ self.register_manager.lockReg(src_reg), null },
        .register_pair => |src_regs| self.register_manager.lockRegs(2, src_regs),
        else => @splat(null),
    };
    defer for (src_locks) |src_lock| if (src_lock) |lock| self.register_manager.unlockReg(lock);

    if (!sse_op) {
        switch (air_tag) {
            .add,
            .add_wrap,
            => try self.genBinOpMir(.{ ._, .add }, lhs_ty, dst_mcv, src_mcv),

            .sub,
            .sub_wrap,
            => try self.genBinOpMir(.{ ._, .sub }, lhs_ty, dst_mcv, src_mcv),

            .ptr_add,
            .ptr_sub,
            => {
                const tmp_reg = try self.copyToTmpRegister(rhs_ty, src_mcv);
                const tmp_mcv = MCValue{ .register = tmp_reg };
                const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                defer self.register_manager.unlockReg(tmp_lock);

                const elem_size = lhs_ty.elemType2(zcu).abiSize(zcu);
                try self.genIntMulComplexOpMir(rhs_ty, tmp_mcv, .{ .immediate = elem_size });
                try self.genBinOpMir(
                    switch (air_tag) {
                        .ptr_add => .{ ._, .add },
                        .ptr_sub => .{ ._, .sub },
                        else => unreachable,
                    },
                    lhs_ty,
                    dst_mcv,
                    tmp_mcv,
                );
            },

            .bool_or,
            .bit_or,
            => try self.genBinOpMir(.{ ._, .@"or" }, lhs_ty, dst_mcv, src_mcv),

            .bool_and,
            .bit_and,
            => try self.genBinOpMir(.{ ._, .@"and" }, lhs_ty, dst_mcv, src_mcv),

            .xor => try self.genBinOpMir(.{ ._, .xor }, lhs_ty, dst_mcv, src_mcv),

            .min,
            .max,
            => {
                const resolved_src_mcv = switch (src_mcv) {
                    else => src_mcv,
                    .air_ref => |src_ref| try self.resolveInst(src_ref),
                };

                if (abi_size > 8) {
                    const dst_regs = switch (dst_mcv) {
                        .register_pair => |dst_regs| dst_regs,
                        else => dst: {
                            const dst_regs = try self.register_manager.allocRegs(2, @splat(null), abi.RegisterClass.gp);
                            const dst_regs_locks = self.register_manager.lockRegsAssumeUnused(2, dst_regs);
                            defer for (dst_regs_locks) |lock| self.register_manager.unlockReg(lock);

                            try self.genCopy(lhs_ty, .{ .register_pair = dst_regs }, dst_mcv, .{});
                            break :dst dst_regs;
                        },
                    };
                    const dst_regs_locks = self.register_manager.lockRegs(2, dst_regs);
                    defer for (dst_regs_locks) |dst_lock| if (dst_lock) |lock|
                        self.register_manager.unlockReg(lock);

                    const tmp_reg = try self.register_manager.allocReg(null, abi.RegisterClass.gp);
                    const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                    defer self.register_manager.unlockReg(tmp_lock);

                    const signed = lhs_ty.isSignedInt(zcu);
                    const cc: Condition = switch (air_tag) {
                        .min => if (signed) .nl else .nb,
                        .max => if (signed) .nge else .nae,
                        else => unreachable,
                    };

                    try self.asmRegisterRegister(.{ ._, .mov }, tmp_reg, dst_regs[1]);
                    if (src_mcv.isBase()) {
                        try self.asmRegisterMemory(
                            .{ ._, .cmp },
                            dst_regs[0],
                            try src_mcv.mem(self, .{ .size = .qword }),
                        );
                        try self.asmRegisterMemory(
                            .{ ._, .sbb },
                            tmp_reg,
                            try src_mcv.address().offset(8).deref().mem(self, .{ .size = .qword }),
                        );
                        try self.asmCmovccRegisterMemory(
                            cc,
                            dst_regs[0],
                            try src_mcv.mem(self, .{ .size = .qword }),
                        );
                        try self.asmCmovccRegisterMemory(
                            cc,
                            dst_regs[1],
                            try src_mcv.address().offset(8).deref().mem(self, .{ .size = .qword }),
                        );
                    } else {
                        try self.asmRegisterRegister(
                            .{ ._, .cmp },
                            dst_regs[0],
                            src_mcv.register_pair[0],
                        );
                        try self.asmRegisterRegister(
                            .{ ._, .sbb },
                            tmp_reg,
                            src_mcv.register_pair[1],
                        );
                        try self.asmCmovccRegisterRegister(cc, dst_regs[0], src_mcv.register_pair[0]);
                        try self.asmCmovccRegisterRegister(cc, dst_regs[1], src_mcv.register_pair[1]);
                    }
                    try self.genCopy(lhs_ty, dst_mcv, .{ .register_pair = dst_regs }, .{});
                } else {
                    const mat_src_mcv: MCValue = if (switch (resolved_src_mcv) {
                        .immediate,
                        .eflags,
                        .register_offset,
                        .load_symbol,
                        .lea_symbol,
                        .load_direct,
                        .lea_direct,
                        .load_got,
                        .lea_got,
                        .load_tlv,
                        .lea_tlv,
                        .lea_frame,
                        => true,
                        .memory => |addr| std.math.cast(i32, @as(i64, @bitCast(addr))) == null,
                        else => false,
                        .register_pair,
                        .register_overflow,
                        => unreachable,
                    })
                        .{ .register = try self.copyToTmpRegister(rhs_ty, resolved_src_mcv) }
                    else
                        resolved_src_mcv;
                    const mat_mcv_lock = switch (mat_src_mcv) {
                        .register => |reg| self.register_manager.lockReg(reg),
                        else => null,
                    };
                    defer if (mat_mcv_lock) |lock| self.register_manager.unlockReg(lock);

                    try self.genBinOpMir(.{ ._, .cmp }, lhs_ty, dst_mcv, mat_src_mcv);

                    const int_info = lhs_ty.intInfo(zcu);
                    const cc: Condition = switch (int_info.signedness) {
                        .unsigned => switch (air_tag) {
                            .min => .a,
                            .max => .b,
                            else => unreachable,
                        },
                        .signed => switch (air_tag) {
                            .min => .g,
                            .max => .l,
                            else => unreachable,
                        },
                    };

                    const cmov_abi_size = @max(@as(u32, @intCast(lhs_ty.abiSize(zcu))), 2);
                    const tmp_reg = switch (dst_mcv) {
                        .register => |reg| reg,
                        else => try self.copyToTmpRegister(lhs_ty, dst_mcv),
                    };
                    const tmp_lock = self.register_manager.lockReg(tmp_reg);
                    defer if (tmp_lock) |lock| self.register_manager.unlockReg(lock);
                    switch (mat_src_mcv) {
                        .none,
                        .unreach,
                        .dead,
                        .undef,
                        .immediate,
                        .eflags,
                        .register_pair,
                        .register_triple,
                        .register_quadruple,
                        .register_offset,
                        .register_overflow,
                        .register_mask,
                        .load_symbol,
                        .lea_symbol,
                        .load_direct,
                        .lea_direct,
                        .load_got,
                        .lea_got,
                        .load_tlv,
                        .lea_tlv,
                        .lea_frame,
                        .elementwise_regs_then_frame,
                        .reserved_frame,
                        .air_ref,
                        => unreachable,
                        .register => |src_reg| try self.asmCmovccRegisterRegister(
                            cc,
                            registerAlias(tmp_reg, cmov_abi_size),
                            registerAlias(src_reg, cmov_abi_size),
                        ),
                        .memory, .indirect, .load_frame => try self.asmCmovccRegisterMemory(
                            cc,
                            registerAlias(tmp_reg, cmov_abi_size),
                            switch (mat_src_mcv) {
                                .memory => |addr| .{
                                    .base = .{ .reg = .ds },
                                    .mod = .{ .rm = .{
                                        .size = .fromSize(cmov_abi_size),
                                        .disp = @intCast(@as(i64, @bitCast(addr))),
                                    } },
                                },
                                .indirect => |reg_off| .{
                                    .base = .{ .reg = reg_off.reg },
                                    .mod = .{ .rm = .{
                                        .size = .fromSize(cmov_abi_size),
                                        .disp = reg_off.off,
                                    } },
                                },
                                .load_frame => |frame_addr| .{
                                    .base = .{ .frame = frame_addr.index },
                                    .mod = .{ .rm = .{
                                        .size = .fromSize(cmov_abi_size),
                                        .disp = frame_addr.off,
                                    } },
                                },
                                else => unreachable,
                            },
                        ),
                    }
                    try self.genCopy(lhs_ty, dst_mcv, .{ .register = tmp_reg }, .{});
                }
            },

            .cmp_eq, .cmp_neq => {
                assert(lhs_ty.isVector(zcu) and lhs_ty.childType(zcu).toIntern() == .bool_type);
                try self.genBinOpMir(.{ ._, .xor }, lhs_ty, dst_mcv, src_mcv);
                switch (air_tag) {
                    .cmp_eq => try self.genUnOpMir(.{ ._, .not }, lhs_ty, dst_mcv),
                    .cmp_neq => {},
                    else => unreachable,
                }
            },

            else => return self.fail("TODO implement genBinOp for {s} {}", .{
                @tagName(air_tag), lhs_ty.fmt(pt),
            }),
        }
        return dst_mcv;
    }

    const dst_reg = registerAlias(dst_mcv.getReg().?, abi_size);
    const mir_tag = @as(?Mir.Inst.FixedTag, switch (lhs_ty.zigTypeTag(zcu)) {
        else => unreachable,
        .float => switch (lhs_ty.floatBits(self.target.*)) {
            16 => {
                assert(self.hasFeature(.f16c));
                const lhs_reg = if (copied_to_dst) dst_reg else registerAlias(lhs_mcv.getReg().?, abi_size);

                const tmp_reg = (try self.register_manager.allocReg(null, abi.RegisterClass.sse)).to128();
                const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                defer self.register_manager.unlockReg(tmp_lock);

                if (src_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                    .{ .vp_w, .insr },
                    dst_reg,
                    lhs_reg,
                    try src_mcv.mem(self, .{ .size = .word }),
                    .u(1),
                ) else try self.asmRegisterRegisterRegister(
                    .{ .vp_, .unpcklwd },
                    dst_reg,
                    lhs_reg,
                    (if (src_mcv.isRegister())
                        src_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                );
                try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg, dst_reg);
                try self.asmRegisterRegister(.{ .v_, .movshdup }, tmp_reg, dst_reg);
                try self.asmRegisterRegisterRegister(
                    switch (air_tag) {
                        .add => .{ .v_ss, .add },
                        .sub => .{ .v_ss, .sub },
                        .mul => .{ .v_ss, .mul },
                        .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ss, .div },
                        .max => .{ .v_ss, .max },
                        .min => .{ .v_ss, .min },
                        else => unreachable,
                    },
                    dst_reg,
                    dst_reg,
                    tmp_reg,
                );
                switch (air_tag) {
                    .div_trunc, .div_floor => try self.asmRegisterRegisterRegisterImmediate(
                        .{ .v_ss, .round },
                        dst_reg,
                        dst_reg,
                        dst_reg,
                        bits.RoundMode.imm(.{
                            .direction = switch (air_tag) {
                                .div_trunc => .zero,
                                .div_floor => .down,
                                else => unreachable,
                            },
                            .precision = .inexact,
                        }),
                    ),
                    else => {},
                }
                try self.asmRegisterRegisterImmediate(
                    .{ .v_, .cvtps2ph },
                    dst_reg,
                    dst_reg,
                    bits.RoundMode.imm(.{}),
                );
                return dst_mcv;
            },
            32 => switch (air_tag) {
                .add => if (self.hasFeature(.avx)) .{ .v_ss, .add } else .{ ._ss, .add },
                .sub => if (self.hasFeature(.avx)) .{ .v_ss, .sub } else .{ ._ss, .sub },
                .mul => if (self.hasFeature(.avx)) .{ .v_ss, .mul } else .{ ._ss, .mul },
                .div_float,
                .div_trunc,
                .div_floor,
                .div_exact,
                => if (self.hasFeature(.avx)) .{ .v_ss, .div } else .{ ._ss, .div },
                .max => if (self.hasFeature(.avx)) .{ .v_ss, .max } else .{ ._ss, .max },
                .min => if (self.hasFeature(.avx)) .{ .v_ss, .min } else .{ ._ss, .min },
                else => unreachable,
            },
            64 => switch (air_tag) {
                .add => if (self.hasFeature(.avx)) .{ .v_sd, .add } else .{ ._sd, .add },
                .sub => if (self.hasFeature(.avx)) .{ .v_sd, .sub } else .{ ._sd, .sub },
                .mul => if (self.hasFeature(.avx)) .{ .v_sd, .mul } else .{ ._sd, .mul },
                .div_float,
                .div_trunc,
                .div_floor,
                .div_exact,
                => if (self.hasFeature(.avx)) .{ .v_sd, .div } else .{ ._sd, .div },
                .max => if (self.hasFeature(.avx)) .{ .v_sd, .max } else .{ ._sd, .max },
                .min => if (self.hasFeature(.avx)) .{ .v_sd, .min } else .{ ._sd, .min },
                else => unreachable,
            },
            80, 128 => null,
            else => unreachable,
        },
        .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
            else => null,
            .int => switch (lhs_ty.childType(zcu).intInfo(zcu).bits) {
                8 => switch (lhs_ty.vectorLen(zcu)) {
                    1...16 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_b, .add } else .{ .p_b, .add },
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_b, .sub } else .{ .p_b, .sub },
                        .bit_and => if (self.hasFeature(.avx))
                            .{ .vp_, .@"and" }
                        else
                            .{ .p_, .@"and" },
                        .bit_or => if (self.hasFeature(.avx)) .{ .vp_, .@"or" } else .{ .p_, .@"or" },
                        .xor => if (self.hasFeature(.avx)) .{ .vp_, .xor } else .{ .p_, .xor },
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_b, .mins }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_b, .mins }
                            else
                                null,
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_b, .minu }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_b, .minu }
                            else
                                null,
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_b, .maxs }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_b, .maxs }
                            else
                                null,
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_b, .maxu }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_b, .maxu }
                            else
                                null,
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_b, .cmpgt }
                            else
                                .{ .p_b, .cmpgt },
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_b, .cmpeq } else .{ .p_b, .cmpeq },
                        else => null,
                    },
                    17...32 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_b, .add } else null,
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_b, .sub } else null,
                        .bit_and => if (self.hasFeature(.avx2)) .{ .vp_, .@"and" } else null,
                        .bit_or => if (self.hasFeature(.avx2)) .{ .vp_, .@"or" } else null,
                        .xor => if (self.hasFeature(.avx2)) .{ .vp_, .xor } else null,
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_b, .mins } else null,
                            .unsigned => if (self.hasFeature(.avx)) .{ .vp_b, .minu } else null,
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_b, .maxs } else null,
                            .unsigned => if (self.hasFeature(.avx2)) .{ .vp_b, .maxu } else null,
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx)) .{ .vp_b, .cmpgt } else null,
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_b, .cmpeq } else null,
                        else => null,
                    },
                    else => null,
                },
                16 => switch (lhs_ty.vectorLen(zcu)) {
                    1...8 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_w, .add } else .{ .p_w, .add },
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_w, .sub } else .{ .p_w, .sub },
                        .mul,
                        .mul_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_w, .mull } else .{ .p_d, .mull },
                        .bit_and => if (self.hasFeature(.avx))
                            .{ .vp_, .@"and" }
                        else
                            .{ .p_, .@"and" },
                        .bit_or => if (self.hasFeature(.avx)) .{ .vp_, .@"or" } else .{ .p_, .@"or" },
                        .xor => if (self.hasFeature(.avx)) .{ .vp_, .xor } else .{ .p_, .xor },
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_w, .mins }
                            else
                                .{ .p_w, .mins },
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_w, .minu }
                            else
                                .{ .p_w, .minu },
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_w, .maxs }
                            else
                                .{ .p_w, .maxs },
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_w, .maxu }
                            else
                                .{ .p_w, .maxu },
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_w, .cmpgt }
                            else
                                .{ .p_w, .cmpgt },
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_w, .cmpeq } else .{ .p_w, .cmpeq },
                        else => null,
                    },
                    9...16 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_w, .add } else null,
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_w, .sub } else null,
                        .mul,
                        .mul_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_w, .mull } else null,
                        .bit_and => if (self.hasFeature(.avx2)) .{ .vp_, .@"and" } else null,
                        .bit_or => if (self.hasFeature(.avx2)) .{ .vp_, .@"or" } else null,
                        .xor => if (self.hasFeature(.avx2)) .{ .vp_, .xor } else null,
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_w, .mins } else null,
                            .unsigned => if (self.hasFeature(.avx)) .{ .vp_w, .minu } else null,
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_w, .maxs } else null,
                            .unsigned => if (self.hasFeature(.avx2)) .{ .vp_w, .maxu } else null,
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx)) .{ .vp_w, .cmpgt } else null,
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_w, .cmpeq } else null,
                        else => null,
                    },
                    else => null,
                },
                32 => switch (lhs_ty.vectorLen(zcu)) {
                    1...4 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_d, .add } else .{ .p_d, .add },
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_d, .sub } else .{ .p_d, .sub },
                        .mul,
                        .mul_wrap,
                        => if (self.hasFeature(.avx))
                            .{ .vp_d, .mull }
                        else if (self.hasFeature(.sse4_1))
                            .{ .p_d, .mull }
                        else
                            null,
                        .bit_and => if (self.hasFeature(.avx))
                            .{ .vp_, .@"and" }
                        else
                            .{ .p_, .@"and" },
                        .bit_or => if (self.hasFeature(.avx)) .{ .vp_, .@"or" } else .{ .p_, .@"or" },
                        .xor => if (self.hasFeature(.avx)) .{ .vp_, .xor } else .{ .p_, .xor },
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_d, .mins }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_d, .mins }
                            else
                                null,
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_d, .minu }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_d, .minu }
                            else
                                null,
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_d, .maxs }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_d, .maxs }
                            else
                                null,
                            .unsigned => if (self.hasFeature(.avx))
                                .{ .vp_d, .maxu }
                            else if (self.hasFeature(.sse4_1))
                                .{ .p_d, .maxu }
                            else
                                null,
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_d, .cmpgt }
                            else
                                .{ .p_d, .cmpgt },
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_d, .cmpeq } else .{ .p_d, .cmpeq },
                        else => null,
                    },
                    5...8 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_d, .add } else null,
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_d, .sub } else null,
                        .mul,
                        .mul_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_d, .mull } else null,
                        .bit_and => if (self.hasFeature(.avx2)) .{ .vp_, .@"and" } else null,
                        .bit_or => if (self.hasFeature(.avx2)) .{ .vp_, .@"or" } else null,
                        .xor => if (self.hasFeature(.avx2)) .{ .vp_, .xor } else null,
                        .min => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_d, .mins } else null,
                            .unsigned => if (self.hasFeature(.avx)) .{ .vp_d, .minu } else null,
                        },
                        .max => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx2)) .{ .vp_d, .maxs } else null,
                            .unsigned => if (self.hasFeature(.avx2)) .{ .vp_d, .maxu } else null,
                        },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx)) .{ .vp_d, .cmpgt } else null,
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_d, .cmpeq } else null,
                        else => null,
                    },
                    else => null,
                },
                64 => switch (lhs_ty.vectorLen(zcu)) {
                    1...2 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_q, .add } else .{ .p_q, .add },
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx)) .{ .vp_q, .sub } else .{ .p_q, .sub },
                        .bit_and => if (self.hasFeature(.avx))
                            .{ .vp_, .@"and" }
                        else
                            .{ .p_, .@"and" },
                        .bit_or => if (self.hasFeature(.avx)) .{ .vp_, .@"or" } else .{ .p_, .@"or" },
                        .xor => if (self.hasFeature(.avx)) .{ .vp_, .xor } else .{ .p_, .xor },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gte,
                        .cmp_gt,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx))
                                .{ .vp_q, .cmpgt }
                            else if (self.hasFeature(.sse4_2))
                                .{ .p_q, .cmpgt }
                            else
                                null,
                            .unsigned => null,
                        },
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx))
                            .{ .vp_q, .cmpeq }
                        else if (self.hasFeature(.sse4_1))
                            .{ .p_q, .cmpeq }
                        else
                            null,
                        else => null,
                    },
                    3...4 => switch (air_tag) {
                        .add,
                        .add_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_q, .add } else null,
                        .sub,
                        .sub_wrap,
                        => if (self.hasFeature(.avx2)) .{ .vp_q, .sub } else null,
                        .bit_and => if (self.hasFeature(.avx2)) .{ .vp_, .@"and" } else null,
                        .bit_or => if (self.hasFeature(.avx2)) .{ .vp_, .@"or" } else null,
                        .xor => if (self.hasFeature(.avx2)) .{ .vp_, .xor } else null,
                        .cmp_eq,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .vp_d, .cmpeq } else null,
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_gt,
                        .cmp_gte,
                        => switch (lhs_ty.childType(zcu).intInfo(zcu).signedness) {
                            .signed => if (self.hasFeature(.avx)) .{ .vp_d, .cmpgt } else null,
                            .unsigned => null,
                        },
                        else => null,
                    },
                    else => null,
                },
                else => null,
            },
            .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                16 => tag: {
                    assert(self.hasFeature(.f16c));
                    const lhs_reg = if (copied_to_dst) dst_reg else registerAlias(lhs_mcv.getReg().?, abi_size);
                    switch (lhs_ty.vectorLen(zcu)) {
                        1 => {
                            const tmp_reg =
                                (try self.register_manager.allocReg(null, abi.RegisterClass.sse)).to128();
                            const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                            defer self.register_manager.unlockReg(tmp_lock);

                            if (src_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                                .{ .vp_w, .insr },
                                dst_reg,
                                lhs_reg,
                                try src_mcv.mem(self, .{ .size = .word }),
                                .u(1),
                            ) else try self.asmRegisterRegisterRegister(
                                .{ .vp_, .unpcklwd },
                                dst_reg,
                                lhs_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                            try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg, dst_reg);
                            try self.asmRegisterRegister(.{ .v_, .movshdup }, tmp_reg, dst_reg);
                            try self.asmRegisterRegisterRegister(
                                switch (air_tag) {
                                    .add => .{ .v_ss, .add },
                                    .sub => .{ .v_ss, .sub },
                                    .mul => .{ .v_ss, .mul },
                                    .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ss, .div },
                                    .max => .{ .v_ss, .max },
                                    .min => .{ .v_ss, .max },
                                    else => unreachable,
                                },
                                dst_reg,
                                dst_reg,
                                tmp_reg,
                            );
                            try self.asmRegisterRegisterImmediate(
                                .{ .v_, .cvtps2ph },
                                dst_reg,
                                dst_reg,
                                bits.RoundMode.imm(.{}),
                            );
                            return dst_mcv;
                        },
                        2 => {
                            const tmp_reg = (try self.register_manager.allocReg(
                                null,
                                abi.RegisterClass.sse,
                            )).to128();
                            const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                            defer self.register_manager.unlockReg(tmp_lock);

                            if (src_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                                .{ .vp_d, .insr },
                                dst_reg,
                                lhs_reg,
                                try src_mcv.mem(self, .{ .size = .dword }),
                                .u(1),
                            ) else try self.asmRegisterRegisterRegister(
                                .{ .v_ps, .unpckl },
                                dst_reg,
                                lhs_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                            try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg, dst_reg);
                            try self.asmRegisterRegisterRegister(
                                .{ .v_ps, .movhl },
                                tmp_reg,
                                dst_reg,
                                dst_reg,
                            );
                            try self.asmRegisterRegisterRegister(
                                switch (air_tag) {
                                    .add => .{ .v_ps, .add },
                                    .sub => .{ .v_ps, .sub },
                                    .mul => .{ .v_ps, .mul },
                                    .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ps, .div },
                                    .max => .{ .v_ps, .max },
                                    .min => .{ .v_ps, .max },
                                    else => unreachable,
                                },
                                dst_reg,
                                dst_reg,
                                tmp_reg,
                            );
                            try self.asmRegisterRegisterImmediate(
                                .{ .v_, .cvtps2ph },
                                dst_reg,
                                dst_reg,
                                bits.RoundMode.imm(.{}),
                            );
                            return dst_mcv;
                        },
                        3...4 => {
                            const tmp_reg = (try self.register_manager.allocReg(
                                null,
                                abi.RegisterClass.sse,
                            )).to128();
                            const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                            defer self.register_manager.unlockReg(tmp_lock);

                            try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg, lhs_reg);
                            if (src_mcv.isBase()) try self.asmRegisterMemory(
                                .{ .v_ps, .cvtph2 },
                                tmp_reg,
                                try src_mcv.mem(self, .{ .size = .qword }),
                            ) else try self.asmRegisterRegister(
                                .{ .v_ps, .cvtph2 },
                                tmp_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                            try self.asmRegisterRegisterRegister(
                                switch (air_tag) {
                                    .add => .{ .v_ps, .add },
                                    .sub => .{ .v_ps, .sub },
                                    .mul => .{ .v_ps, .mul },
                                    .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ps, .div },
                                    .max => .{ .v_ps, .max },
                                    .min => .{ .v_ps, .max },
                                    else => unreachable,
                                },
                                dst_reg,
                                dst_reg,
                                tmp_reg,
                            );
                            try self.asmRegisterRegisterImmediate(
                                .{ .v_, .cvtps2ph },
                                dst_reg,
                                dst_reg,
                                bits.RoundMode.imm(.{}),
                            );
                            return dst_mcv;
                        },
                        5...8 => {
                            const tmp_reg = (try self.register_manager.allocReg(
                                null,
                                abi.RegisterClass.sse,
                            )).to256();
                            const tmp_lock = self.register_manager.lockRegAssumeUnused(tmp_reg);
                            defer self.register_manager.unlockReg(tmp_lock);

                            try self.asmRegisterRegister(.{ .v_ps, .cvtph2 }, dst_reg.to256(), lhs_reg);
                            if (src_mcv.isBase()) try self.asmRegisterMemory(
                                .{ .v_ps, .cvtph2 },
                                tmp_reg,
                                try src_mcv.mem(self, .{ .size = .xword }),
                            ) else try self.asmRegisterRegister(
                                .{ .v_ps, .cvtph2 },
                                tmp_reg,
                                (if (src_mcv.isRegister())
                                    src_mcv.getReg().?
                                else
                                    try self.copyToTmpRegister(rhs_ty, src_mcv)).to128(),
                            );
                            try self.asmRegisterRegisterRegister(
                                switch (air_tag) {
                                    .add => .{ .v_ps, .add },
                                    .sub => .{ .v_ps, .sub },
                                    .mul => .{ .v_ps, .mul },
                                    .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ps, .div },
                                    .max => .{ .v_ps, .max },
                                    .min => .{ .v_ps, .max },
                                    else => unreachable,
                                },
                                dst_reg.to256(),
                                dst_reg.to256(),
                                tmp_reg,
                            );
                            try self.asmRegisterRegisterImmediate(
                                .{ .v_, .cvtps2ph },
                                dst_reg,
                                dst_reg.to256(),
                                bits.RoundMode.imm(.{}),
                            );
                            return dst_mcv;
                        },
                        else => break :tag null,
                    }
                },
                32 => switch (lhs_ty.vectorLen(zcu)) {
                    1 => switch (air_tag) {
                        .add => if (self.hasFeature(.avx)) .{ .v_ss, .add } else .{ ._ss, .add },
                        .sub => if (self.hasFeature(.avx)) .{ .v_ss, .sub } else .{ ._ss, .sub },
                        .mul => if (self.hasFeature(.avx)) .{ .v_ss, .mul } else .{ ._ss, .mul },
                        .div_float,
                        .div_trunc,
                        .div_floor,
                        .div_exact,
                        => if (self.hasFeature(.avx)) .{ .v_ss, .div } else .{ ._ss, .div },
                        .max => if (self.hasFeature(.avx)) .{ .v_ss, .max } else .{ ._ss, .max },
                        .min => if (self.hasFeature(.avx)) .{ .v_ss, .min } else .{ ._ss, .min },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_eq,
                        .cmp_gte,
                        .cmp_gt,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .v_ss, .cmp } else .{ ._ss, .cmp },
                        else => unreachable,
                    },
                    2...4 => switch (air_tag) {
                        .add => if (self.hasFeature(.avx)) .{ .v_ps, .add } else .{ ._ps, .add },
                        .sub => if (self.hasFeature(.avx)) .{ .v_ps, .sub } else .{ ._ps, .sub },
                        .mul => if (self.hasFeature(.avx)) .{ .v_ps, .mul } else .{ ._ps, .mul },
                        .div_float,
                        .div_trunc,
                        .div_floor,
                        .div_exact,
                        => if (self.hasFeature(.avx)) .{ .v_ps, .div } else .{ ._ps, .div },
                        .max => if (self.hasFeature(.avx)) .{ .v_ps, .max } else .{ ._ps, .max },
                        .min => if (self.hasFeature(.avx)) .{ .v_ps, .min } else .{ ._ps, .min },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_eq,
                        .cmp_gte,
                        .cmp_gt,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .v_ps, .cmp } else .{ ._ps, .cmp },
                        else => unreachable,
                    },
                    5...8 => if (self.hasFeature(.avx)) switch (air_tag) {
                        .add => .{ .v_ps, .add },
                        .sub => .{ .v_ps, .sub },
                        .mul => .{ .v_ps, .mul },
                        .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_ps, .div },
                        .max => .{ .v_ps, .max },
                        .min => .{ .v_ps, .min },
                        .cmp_lt, .cmp_lte, .cmp_eq, .cmp_gte, .cmp_gt, .cmp_neq => .{ .v_ps, .cmp },
                        else => unreachable,
                    } else null,
                    else => null,
                },
                64 => switch (lhs_ty.vectorLen(zcu)) {
                    1 => switch (air_tag) {
                        .add => if (self.hasFeature(.avx)) .{ .v_sd, .add } else .{ ._sd, .add },
                        .sub => if (self.hasFeature(.avx)) .{ .v_sd, .sub } else .{ ._sd, .sub },
                        .mul => if (self.hasFeature(.avx)) .{ .v_sd, .mul } else .{ ._sd, .mul },
                        .div_float,
                        .div_trunc,
                        .div_floor,
                        .div_exact,
                        => if (self.hasFeature(.avx)) .{ .v_sd, .div } else .{ ._sd, .div },
                        .max => if (self.hasFeature(.avx)) .{ .v_sd, .max } else .{ ._sd, .max },
                        .min => if (self.hasFeature(.avx)) .{ .v_sd, .min } else .{ ._sd, .min },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_eq,
                        .cmp_gte,
                        .cmp_gt,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .v_sd, .cmp } else .{ ._sd, .cmp },
                        else => unreachable,
                    },
                    2 => switch (air_tag) {
                        .add => if (self.hasFeature(.avx)) .{ .v_pd, .add } else .{ ._pd, .add },
                        .sub => if (self.hasFeature(.avx)) .{ .v_pd, .sub } else .{ ._pd, .sub },
                        .mul => if (self.hasFeature(.avx)) .{ .v_pd, .mul } else .{ ._pd, .mul },
                        .div_float,
                        .div_trunc,
                        .div_floor,
                        .div_exact,
                        => if (self.hasFeature(.avx)) .{ .v_pd, .div } else .{ ._pd, .div },
                        .max => if (self.hasFeature(.avx)) .{ .v_pd, .max } else .{ ._pd, .max },
                        .min => if (self.hasFeature(.avx)) .{ .v_pd, .min } else .{ ._pd, .min },
                        .cmp_lt,
                        .cmp_lte,
                        .cmp_eq,
                        .cmp_gte,
                        .cmp_gt,
                        .cmp_neq,
                        => if (self.hasFeature(.avx)) .{ .v_pd, .cmp } else .{ ._pd, .cmp },
                        else => unreachable,
                    },
                    3...4 => if (self.hasFeature(.avx)) switch (air_tag) {
                        .add => .{ .v_pd, .add },
                        .sub => .{ .v_pd, .sub },
                        .mul => .{ .v_pd, .mul },
                        .div_float, .div_trunc, .div_floor, .div_exact => .{ .v_pd, .div },
                        .max => .{ .v_pd, .max },
                        .cmp_lt, .cmp_lte, .cmp_eq, .cmp_gte, .cmp_gt, .cmp_neq => .{ .v_pd, .cmp },
                        .min => .{ .v_pd, .min },
                        else => unreachable,
                    } else null,
                    else => null,
                },
                80, 128 => null,
                else => unreachable,
            },
        },
    }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
        @tagName(air_tag), lhs_ty.fmt(pt),
    });

    const lhs_copy_reg = if (maybe_mask_reg) |_| registerAlias(
        if (copied_to_dst) try self.copyToTmpRegister(lhs_ty, dst_mcv) else lhs_mcv.getReg().?,
        abi_size,
    ) else null;
    const lhs_copy_lock = if (lhs_copy_reg) |reg| self.register_manager.lockReg(reg) else null;
    defer if (lhs_copy_lock) |lock| self.register_manager.unlockReg(lock);

    switch (mir_tag[1]) {
        else => if (self.hasFeature(.avx)) {
            const lhs_reg = if (copied_to_dst) dst_reg else registerAlias(lhs_mcv.getReg().?, abi_size);
            if (src_mcv.isBase()) try self.asmRegisterRegisterMemory(
                mir_tag,
                dst_reg,
                lhs_reg,
                try src_mcv.mem(self, .{ .size = switch (lhs_ty.zigTypeTag(zcu)) {
                    else => .fromSize(abi_size),
                    .vector => .fromBitSize(dst_reg.bitSize()),
                } }),
            ) else try self.asmRegisterRegisterRegister(
                mir_tag,
                dst_reg,
                lhs_reg,
                registerAlias(if (src_mcv.isRegister())
                    src_mcv.getReg().?
                else
                    try self.copyToTmpRegister(rhs_ty, src_mcv), abi_size),
            );
        } else {
            assert(copied_to_dst);
            if (src_mcv.isBase()) try self.asmRegisterMemory(
                mir_tag,
                dst_reg,
                try src_mcv.mem(self, .{ .size = switch (lhs_ty.zigTypeTag(zcu)) {
                    else => .fromSize(abi_size),
                    .vector => .fromBitSize(dst_reg.bitSize()),
                } }),
            ) else try self.asmRegisterRegister(
                mir_tag,
                dst_reg,
                registerAlias(if (src_mcv.isRegister())
                    src_mcv.getReg().?
                else
                    try self.copyToTmpRegister(rhs_ty, src_mcv), abi_size),
            );
        },
        .cmp => {
            const imm: Immediate = .u(switch (air_tag) {
                .cmp_eq => 0,
                .cmp_lt, .cmp_gt => 1,
                .cmp_lte, .cmp_gte => 2,
                .cmp_neq => 4,
                else => unreachable,
            });
            if (self.hasFeature(.avx)) {
                const lhs_reg =
                    if (copied_to_dst) dst_reg else registerAlias(lhs_mcv.getReg().?, abi_size);
                if (src_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                    mir_tag,
                    dst_reg,
                    lhs_reg,
                    try src_mcv.mem(self, .{ .size = switch (lhs_ty.zigTypeTag(zcu)) {
                        else => .fromSize(abi_size),
                        .vector => .fromBitSize(dst_reg.bitSize()),
                    } }),
                    imm,
                ) else try self.asmRegisterRegisterRegisterImmediate(
                    mir_tag,
                    dst_reg,
                    lhs_reg,
                    registerAlias(if (src_mcv.isRegister())
                        src_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(rhs_ty, src_mcv), abi_size),
                    imm,
                );
            } else {
                assert(copied_to_dst);
                if (src_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                    mir_tag,
                    dst_reg,
                    try src_mcv.mem(self, .{ .size = switch (lhs_ty.zigTypeTag(zcu)) {
                        else => .fromSize(abi_size),
                        .vector => .fromBitSize(dst_reg.bitSize()),
                    } }),
                    imm,
                ) else try self.asmRegisterRegisterImmediate(
                    mir_tag,
                    dst_reg,
                    registerAlias(if (src_mcv.isRegister())
                        src_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(rhs_ty, src_mcv), abi_size),
                    imm,
                );
            }
        },
    }

    switch (air_tag) {
        .add, .add_wrap, .sub, .sub_wrap, .mul, .mul_wrap, .div_float, .div_exact => {},
        .div_trunc, .div_floor => try self.genRound(lhs_ty, dst_reg, .{ .register = dst_reg }, .{
            .direction = switch (air_tag) {
                .div_trunc => .zero,
                .div_floor => .down,
                else => unreachable,
            },
            .precision = .inexact,
        }),
        .bit_and, .bit_or, .xor => {},
        .max, .min => if (maybe_mask_reg) |mask_reg| if (self.hasFeature(.avx)) {
            const rhs_copy_reg = registerAlias(src_mcv.getReg().?, abi_size);

            try self.asmRegisterRegisterRegisterImmediate(
                @as(?Mir.Inst.FixedTag, switch (lhs_ty.zigTypeTag(zcu)) {
                    .float => switch (lhs_ty.floatBits(self.target.*)) {
                        32 => .{ .v_ss, .cmp },
                        64 => .{ .v_sd, .cmp },
                        16, 80, 128 => null,
                        else => unreachable,
                    },
                    .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                        .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                            32 => switch (lhs_ty.vectorLen(zcu)) {
                                1 => .{ .v_ss, .cmp },
                                2...8 => .{ .v_ps, .cmp },
                                else => null,
                            },
                            64 => switch (lhs_ty.vectorLen(zcu)) {
                                1 => .{ .v_sd, .cmp },
                                2...4 => .{ .v_pd, .cmp },
                                else => null,
                            },
                            16, 80, 128 => null,
                            else => unreachable,
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
                    @tagName(air_tag), lhs_ty.fmt(pt),
                }),
                mask_reg,
                rhs_copy_reg,
                rhs_copy_reg,
                bits.VexFloatPredicate.imm(.unord),
            );
            try self.asmRegisterRegisterRegisterRegister(
                @as(?Mir.Inst.FixedTag, switch (lhs_ty.zigTypeTag(zcu)) {
                    .float => switch (lhs_ty.floatBits(self.target.*)) {
                        32 => .{ .v_ps, .blendv },
                        64 => .{ .v_pd, .blendv },
                        16, 80, 128 => null,
                        else => unreachable,
                    },
                    .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                        .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                            32 => switch (lhs_ty.vectorLen(zcu)) {
                                1...8 => .{ .v_ps, .blendv },
                                else => null,
                            },
                            64 => switch (lhs_ty.vectorLen(zcu)) {
                                1...4 => .{ .v_pd, .blendv },
                                else => null,
                            },
                            16, 80, 128 => null,
                            else => unreachable,
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
                    @tagName(air_tag), lhs_ty.fmt(pt),
                }),
                dst_reg,
                dst_reg,
                lhs_copy_reg.?,
                mask_reg,
            );
        } else {
            const has_blend = self.hasFeature(.sse4_1);
            try self.asmRegisterRegisterImmediate(
                @as(?Mir.Inst.FixedTag, switch (lhs_ty.zigTypeTag(zcu)) {
                    .float => switch (lhs_ty.floatBits(self.target.*)) {
                        32 => .{ ._ss, .cmp },
                        64 => .{ ._sd, .cmp },
                        16, 80, 128 => null,
                        else => unreachable,
                    },
                    .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                        .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                            32 => switch (lhs_ty.vectorLen(zcu)) {
                                1 => .{ ._ss, .cmp },
                                2...4 => .{ ._ps, .cmp },
                                else => null,
                            },
                            64 => switch (lhs_ty.vectorLen(zcu)) {
                                1 => .{ ._sd, .cmp },
                                2 => .{ ._pd, .cmp },
                                else => null,
                            },
                            16, 80, 128 => null,
                            else => unreachable,
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
                    @tagName(air_tag), lhs_ty.fmt(pt),
                }),
                mask_reg,
                mask_reg,
                bits.SseFloatPredicate.imm(if (has_blend) .unord else .ord),
            );
            if (has_blend) try self.asmRegisterRegisterRegister(
                @as(?Mir.Inst.FixedTag, switch (lhs_ty.zigTypeTag(zcu)) {
                    .float => switch (lhs_ty.floatBits(self.target.*)) {
                        32 => .{ ._ps, .blendv },
                        64 => .{ ._pd, .blendv },
                        16, 80, 128 => null,
                        else => unreachable,
                    },
                    .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                        .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                            32 => switch (lhs_ty.vectorLen(zcu)) {
                                1...4 => .{ ._ps, .blendv },
                                else => null,
                            },
                            64 => switch (lhs_ty.vectorLen(zcu)) {
                                1...2 => .{ ._pd, .blendv },
                                else => null,
                            },
                            16, 80, 128 => null,
                            else => unreachable,
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
                    @tagName(air_tag), lhs_ty.fmt(pt),
                }),
                dst_reg,
                lhs_copy_reg.?,
                mask_reg,
            ) else {
                const mir_fixes = @as(?Mir.Inst.Fixes, switch (lhs_ty.zigTypeTag(zcu)) {
                    .float => switch (lhs_ty.floatBits(self.target.*)) {
                        32 => ._ps,
                        64 => ._pd,
                        16, 80, 128 => null,
                        else => unreachable,
                    },
                    .vector => switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                        .float => switch (lhs_ty.childType(zcu).floatBits(self.target.*)) {
                            32 => switch (lhs_ty.vectorLen(zcu)) {
                                1...4 => ._ps,
                                else => null,
                            },
                            64 => switch (lhs_ty.vectorLen(zcu)) {
                                1...2 => ._pd,
                                else => null,
                            },
                            16, 80, 128 => null,
                            else => unreachable,
                        },
                        else => unreachable,
                    },
                    else => unreachable,
                }) orelse return self.fail("TODO implement genBinOp for {s} {}", .{
                    @tagName(air_tag), lhs_ty.fmt(pt),
                });
                try self.asmRegisterRegister(.{ mir_fixes, .@"and" }, dst_reg, mask_reg);
                try self.asmRegisterRegister(.{ mir_fixes, .andn }, mask_reg, lhs_copy_reg.?);
                try self.asmRegisterRegister(.{ mir_fixes, .@"or" }, dst_reg, mask_reg);
            }
        },
        .cmp_lt, .cmp_lte, .cmp_eq, .cmp_gte, .cmp_gt, .cmp_neq => {
            switch (lhs_ty.childType(zcu).zigTypeTag(zcu)) {
                .int => switch (air_tag) {
                    .cmp_lt,
                    .cmp_eq,
                    .cmp_gt,
                    => {},
                    .cmp_lte,
                    .cmp_gte,
                    .cmp_neq,
                    => {
                        const unsigned_ty = try lhs_ty.toUnsigned(pt);
                        const not_mcv = try self.genTypedValue(try unsigned_ty.maxInt(pt, unsigned_ty));
                        const not_mem: Memory = if (not_mcv.isBase())
                            try not_mcv.mem(self, .{ .size = .fromSize(abi_size) })
                        else
                            .{ .base = .{
                                .reg = try self.copyToTmpRegister(.usize, not_mcv.address()),
                            }, .mod = .{ .rm = .{ .size = .fromSize(abi_size) } } };
                        switch (mir_tag[0]) {
                            .vp_b, .vp_d, .vp_q, .vp_w => try self.asmRegisterRegisterMemory(
                                .{ .vp_, .xor },
                                dst_reg,
                                dst_reg,
                                not_mem,
                            ),
                            .p_b, .p_d, .p_q, .p_w => try self.asmRegisterMemory(
                                .{ .p_, .xor },
                                dst_reg,
                                not_mem,
                            ),
                            else => unreachable,
                        }
                    },
                    else => unreachable,
                },
                .float => {},
                else => unreachable,
            }

            const gp_reg = try self.register_manager.allocReg(maybe_inst, abi.RegisterClass.gp);
            const gp_lock = self.register_manager.lockRegAssumeUnused(gp_reg);
            defer self.register_manager.unlockReg(gp_lock);

            try self.asmRegisterRegister(switch (mir_tag[0]) {
                ._pd, ._sd, .p_q => .{ ._pd, .movmsk },
                ._ps, ._ss, .p_d => .{ ._ps, .movmsk },
                .p_b => .{ .p_b, .movmsk },
                .p_w => movmsk: {
                    try self.asmRegisterRegister(.{ .p_b, .ackssw }, dst_reg, dst_reg);
                    break :movmsk .{ .p_b, .movmsk };
                },
                .v_pd, .v_sd, .vp_q => .{ .v_pd, .movmsk },
                .v_ps, .v_ss, .vp_d => .{ .v_ps, .movmsk },
                .vp_b => .{ .vp_b, .movmsk },
                .vp_w => movmsk: {
                    try self.asmRegisterRegisterRegister(
                        .{ .vp_b, .ackssw },
                        dst_reg,
                        dst_reg,
                        dst_reg,
                    );
                    break :movmsk .{ .vp_b, .movmsk };
                },
                else => unreachable,
            }, gp_reg.to32(), dst_reg);
            return .{ .register = gp_reg };
        },
        else => unreachable,
    }

    return dst_mcv;
}
```
嗯，我现在要试着理解这段Zig代码。看起来这是一个生成二进制操作的函数，可能属于某种编译器后端，比如代码生成部分。函数名是genBinOp，接受一些参数，包括CodeGen的指针、可能的指令索引、操作类型（air_tag）、左右操作数的引用等等。返回类型是MCValue，可能表示生成的机器码值。

首先，我需要理清代码的结构。函数开始部分处理浮点类型的情况，尤其是需要调用库函数（libcall）的情况。这里有很多条件判断，根据不同的浮点位数（16、32、64、80、128）和操作类型（比如rem、mod、add、sub等）生成不同的库函数调用名称，比如__addhf3之类的。然后调用genCall来生成实际的调用代码。

接下来，代码处理了SSE指令的情况，根据不同的向量类型和操作类型选择合适的SSE指令。这里有很多条件分支，根据不同的向量元素类型（如int、float）和操作（如add、sub、mul等）选择不同的指令，比如vp_add、vp_sub等。还考虑了不同的向量长度（如1-16、17-32等）以及不同的指令集支持（如AVX、SSE4.1等）。

在处理整数操作时，代码使用了不同的指令，比如add、sub、and、or、xor等，并根据操作类型生成相应的机器指令。这里还涉及到寄存器的分配和管理，比如使用register_manager来分配和锁定寄存器，避免冲突。

代码中还有大量的错误处理，比如使用try来处理可能的错误，并在无法处理的情况下返回错误信息，比如“TODO implement genBinOp for ...”。

我注意到代码中有很多条件判断，根据不同的类型和操作选择不同的指令路径。例如，当处理浮点数的加减乘除时，会根据浮点数的位数（16、32、64）和是否支持特定的指令集（如f16c）来决定是生成内联指令还是调用库函数。对于向量操作，会根据向量元素的类型和长度选择合适的SIMD指令。

此外，代码中处理了不同寄存器的分配和锁定，确保在生成指令时寄存器不会被意外覆盖。例如，使用register_manager来分配和锁定寄存器，并在使用后解锁。

在生成实际指令时，代码使用了不同的asm函数，如asmRegisterRegister、asmRegisterMemory等，根据操作类型和操作数的位置生成相应的汇编指令。例如，当操作数是内存地址时，生成带有内存操作数的指令；如果是寄存器，则生成寄存器操作数的指令。

在处理比较操作（如cmp_eq、cmp_lt等）时，代码生成相应的比较指令，并根据结果设置条件码，然后可能使用cmovcc指令来选择结果。对于浮点比较，还考虑了无序（NaN）情况，生成相应的比较指令。

总的来说，这段代码的主要逻辑是根据输入的操作类型和操作数的类型，生成相应的机器指令或库函数调用，处理各种边界条件和目标架构的支持情况，同时管理寄存器的分配和释放，确保生成的代码正确高效。

可能的难点包括理解不同指令集（如AVX、SSE4.1）对指令生成的影响，以及如何处理不同数据类型（如不同位数的浮点数、向量类型）的差异。此外，错误处理和TODO部分表明有些情况尚未实现，需要后续扩展。

总结一下，这个函数的思维模型是：根据操作类型和操作数的类型，通过条件分支选择合适的指令生成路径，管理寄存器使用，处理可能的不支持情况，最终生成正确的机器指令或调用库函数。
================================================
这段代码实现了一个编译器后端中生成二进制操作（如加减乘除等）的复杂逻辑，其核心思维模型可概括如下：

---

### **核心思维模型**

1. **类型与指令分发**  
   - **浮点运算**：根据浮点位数（16/32/64/80/128位）和硬件支持（如f16c、AVX）决定使用内联指令或调用库函数（如`__addhf3`）。特殊操作（如取模、min/max）需要定制库函数名。
   - **整数与向量运算**：根据类型（标量/向量）和元素类型（整型/浮点）选择SSE/AVX指令（如`vp_add`、`v_ss`），同时考虑向量长度和指令集支持（如SSE4.1、AVX2）。

2. **寄存器管理**  
   - **动态分配与锁定**：通过`register_manager`分配和锁定寄存器（如SSE寄存器用于浮点，通用寄存器用于整数），避免操作数冲突。
   - **寄存器重用**：尽可能复用操作数寄存器（如通过`reuseOperandAdvanced`优化寄存器使用）。

3. **指令生成策略**  
   - **内联指令生成**：对支持的指令直接生成汇编操作（如`asmRegisterRegister`生成寄存器间操作）。
   - **复杂操作降级**：对不支持的操作（如80位浮点）降级为库函数调用（`genCall`）或生成错误（`fail`）。
   - **条件码处理**：比较操作（如`cmp_lt`）生成条件码，结合`cmovcc`指令实现条件赋值。

4. **特殊操作处理**  
   - **取模与除法**：通过调整库函数调用结果（如调用`__add?f3`修正模运算结果）。
   - **向量操作优化**：根据向量元素类型和长度选择SIMD指令（如`v_ps.add`用于32位浮点向量加法）。
   - **NaN处理**：浮点比较时处理无序情况（如生成`.cmp.unord`指令）。

5. **错误与扩展性**  
   - **边界条件处理**：对未实现的操作返回`TODO`错误（如`genBinOp for .div_floor`）。
   - **指令集特性检测**：通过`hasFeature`动态判断硬件支持（如AVX是否可用）。

---

### **关键代码路径示例**
1. **浮点加法（32位）**  
   - 检查是否为运行时浮点类型 → 确认无需库调用 → 选择`v_ss.add`（AVX）或`_ss.add`（SSE）指令 → 生成寄存器/内存操作。

2. **向量整数最小值（16位无符号）**  
   - 判断向量长度和元素类型 → 选择`vp_b.minu`（AVX）或`p_b.minu`（SSE4.1） → 处理寄存器分配 → 生成指令。

3. **取模运算（80位浮点）**  
   - 检测到类型需要库调用 → 生成`fmodl`库函数名 → 调用`genCall` → 调整结果。

---

### **总结**
该函数通过精细的类型分发、硬件特性适配和寄存器管理，将高级操作映射到底层机器指令或库调用，同时处理大量边界条件和优化路径，体现了编译器后端代码生成的高度复杂性。