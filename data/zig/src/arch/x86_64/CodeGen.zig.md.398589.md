```zig
fn airShuffle(self: *CodeGen, inst: Air.Inst.Index) !void {
    const pt = self.pt;
    const zcu = pt.zcu;
    const ty_pl = self.air.instructions.items(.data)[@intFromEnum(inst)].ty_pl;
    const extra = self.air.extraData(Air.Shuffle, ty_pl.payload).data;

    const dst_ty = self.typeOfIndex(inst);
    const elem_ty = dst_ty.childType(zcu);
    const elem_abi_size: u16 = @intCast(elem_ty.abiSize(zcu));
    const dst_abi_size: u32 = @intCast(dst_ty.abiSize(zcu));
    const lhs_ty = self.typeOf(extra.a);
    const lhs_abi_size: u32 = @intCast(lhs_ty.abiSize(zcu));
    const rhs_ty = self.typeOf(extra.b);
    const rhs_abi_size: u32 = @intCast(rhs_ty.abiSize(zcu));
    const max_abi_size = @max(dst_abi_size, lhs_abi_size, rhs_abi_size);

    const ExpectedContents = [32]?i32;
    var stack align(@max(@alignOf(ExpectedContents), @alignOf(std.heap.StackFallbackAllocator(0)))) =
        std.heap.stackFallback(@sizeOf(ExpectedContents), self.gpa);
    const allocator = stack.get();

    const mask_elems = try allocator.alloc(?i32, extra.mask_len);
    defer allocator.free(mask_elems);
    for (mask_elems, 0..) |*mask_elem, elem_index| {
        const mask_elem_val =
            Value.fromInterned(extra.mask).elemValue(pt, elem_index) catch unreachable;
        mask_elem.* = if (mask_elem_val.isUndef(zcu))
            null
        else
            @intCast(mask_elem_val.toSignedInt(zcu));
    }

    const has_avx = self.hasFeature(.avx);
    const result = @as(?MCValue, result: {
        for (mask_elems) |mask_elem| {
            if (mask_elem) |_| break;
        } else break :result try self.allocRegOrMem(inst, true);

        for (mask_elems, 0..) |mask_elem, elem_index| {
            if (mask_elem orelse continue != elem_index) break;
        } else {
            const lhs_mcv = try self.resolveInst(extra.a);
            if (self.reuseOperand(inst, extra.a, 0, lhs_mcv)) break :result lhs_mcv;
            const dst_mcv = try self.allocRegOrMem(inst, true);
            try self.genCopy(dst_ty, dst_mcv, lhs_mcv, .{});
            break :result dst_mcv;
        }

        for (mask_elems, 0..) |mask_elem, elem_index| {
            if (~(mask_elem orelse continue) != elem_index) break;
        } else {
            const rhs_mcv = try self.resolveInst(extra.b);
            if (self.reuseOperand(inst, extra.b, 1, rhs_mcv)) break :result rhs_mcv;
            const dst_mcv = try self.allocRegOrMem(inst, true);
            try self.genCopy(dst_ty, dst_mcv, rhs_mcv, .{});
            break :result dst_mcv;
        }

        for ([_]Mir.Inst.Tag{ .unpckl, .unpckh }) |variant| unpck: {
            if (elem_abi_size > 8) break :unpck;
            if (dst_abi_size > self.vectorSize(if (elem_abi_size >= 4) .float else .int)) break :unpck;

            var sources: [2]?u1 = @splat(null);
            for (mask_elems, 0..) |maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index =
                    std.math.cast(u5, if (mask_elem < 0) ~mask_elem else mask_elem) orelse break :unpck;
                const elem_byte = (elem_index >> 1) * elem_abi_size;
                if (mask_elem_index * elem_abi_size != (elem_byte & 0b0111) | @as(u4, switch (variant) {
                    .unpckl => 0b0000,
                    .unpckh => 0b1000,
                    else => unreachable,
                }) | (elem_byte << 1 & 0b10000)) break :unpck;

                const source = @intFromBool(mask_elem < 0);
                if (sources[elem_index & 0b00001]) |prev_source| {
                    if (source != prev_source) break :unpck;
                } else sources[elem_index & 0b00001] = source;
            }
            if (sources[0] orelse break :unpck == sources[1] orelse break :unpck) break :unpck;

            const operands = [2]Air.Inst.Ref{ extra.a, extra.b };
            const operand_tys = [2]Type{ lhs_ty, rhs_ty };
            const lhs_mcv = try self.resolveInst(operands[sources[0].?]);
            const rhs_mcv = try self.resolveInst(operands[sources[1].?]);

            const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                self.reuseOperand(inst, operands[sources[0].?], sources[0].?, lhs_mcv))
                lhs_mcv
            else if (has_avx and lhs_mcv.isRegister())
                .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
            else
                try self.copyToRegisterWithInstTracking(inst, operand_tys[sources[0].?], lhs_mcv);
            const dst_reg = dst_mcv.getReg().?;
            const dst_alias = registerAlias(dst_reg, max_abi_size);

            const mir_tag: Mir.Inst.FixedTag = if ((elem_abi_size >= 4 and elem_ty.isRuntimeFloat()) or
                (dst_abi_size > 16 and !self.hasFeature(.avx2))) .{ switch (elem_abi_size) {
                4 => if (has_avx) .v_ps else ._ps,
                8 => if (has_avx) .v_pd else ._pd,
                else => unreachable,
            }, variant } else .{ if (has_avx) .vp_ else .p_, switch (variant) {
                .unpckl => switch (elem_abi_size) {
                    1 => .unpcklbw,
                    2 => .unpcklwd,
                    4 => .unpckldq,
                    8 => .unpcklqdq,
                    else => unreachable,
                },
                .unpckh => switch (elem_abi_size) {
                    1 => .unpckhbw,
                    2 => .unpckhwd,
                    4 => .unpckhdq,
                    8 => .unpckhqdq,
                    else => unreachable,
                },
                else => unreachable,
            } };
            if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemory(
                mir_tag,
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
            ) else try self.asmRegisterRegisterRegister(
                mir_tag,
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
            ) else if (rhs_mcv.isBase()) try self.asmRegisterMemory(
                mir_tag,
                dst_alias,
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
            ) else try self.asmRegisterRegister(
                mir_tag,
                dst_alias,
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
            );
            break :result dst_mcv;
        }

        pshufd: {
            if (elem_abi_size != 4) break :pshufd;
            if (max_abi_size > self.vectorSize(.float)) break :pshufd;

            var control: u8 = 0b00_00_00_00;
            var sources: [1]?u1 = @splat(null);
            for (mask_elems, 0..) |maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index: u3 = @intCast(if (mask_elem < 0) ~mask_elem else mask_elem);
                if (mask_elem_index & 0b100 != elem_index & 0b100) break :pshufd;

                const source = @intFromBool(mask_elem < 0);
                if (sources[0]) |prev_source| {
                    if (source != prev_source) break :pshufd;
                } else sources[(elem_index & 0b010) >> 1] = source;

                const select_bit: u3 = @intCast((elem_index & 0b011) << 1);
                const select_mask = @as(u8, @intCast(mask_elem_index & 0b011)) << select_bit;
                if (elem_index & 0b100 == 0)
                    control |= select_mask
                else if (control & @as(u8, 0b11) << select_bit != select_mask) break :pshufd;
            }

            const operands = [2]Air.Inst.Ref{ extra.a, extra.b };
            const operand_tys = [2]Type{ lhs_ty, rhs_ty };
            const src_mcv = try self.resolveInst(operands[sources[0] orelse break :pshufd]);

            const dst_reg = if (src_mcv.isRegister() and
                self.reuseOperand(inst, operands[sources[0].?], sources[0].?, src_mcv))
                src_mcv.getReg().?
            else
                try self.register_manager.allocReg(inst, abi.RegisterClass.sse);
            const dst_alias = registerAlias(dst_reg, max_abi_size);

            if (src_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                .{ if (has_avx) .vp_d else .p_d, .shuf },
                dst_alias,
                try src_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
                .u(control),
            ) else try self.asmRegisterRegisterImmediate(
                .{ if (has_avx) .vp_d else .p_d, .shuf },
                dst_alias,
                registerAlias(if (src_mcv.isRegister())
                    src_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[0].?], src_mcv), max_abi_size),
                .u(control),
            );
            break :result .{ .register = dst_reg };
        }

        shufps: {
            if (elem_abi_size != 4) break :shufps;
            if (max_abi_size > self.vectorSize(.float)) break :shufps;

            var control: u8 = 0b00_00_00_00;
            var sources: [2]?u1 = @splat(null);
            for (mask_elems, 0..) |maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index: u3 = @intCast(if (mask_elem < 0) ~mask_elem else mask_elem);
                if (mask_elem_index & 0b100 != elem_index & 0b100) break :shufps;

                const source = @intFromBool(mask_elem < 0);
                if (sources[(elem_index & 0b010) >> 1]) |prev_source| {
                    if (source != prev_source) break :shufps;
                } else sources[(elem_index & 0b010) >> 1] = source;

                const select_bit: u3 = @intCast((elem_index & 0b011) << 1);
                const select_mask = @as(u8, @intCast(mask_elem_index & 0b011)) << select_bit;
                if (elem_index & 0b100 == 0)
                    control |= select_mask
                else if (control & @as(u8, 0b11) << select_bit != select_mask) break :shufps;
            }
            if (sources[0] orelse break :shufps == sources[1] orelse break :shufps) break :shufps;

            const operands = [2]Air.Inst.Ref{ extra.a, extra.b };
            const operand_tys = [2]Type{ lhs_ty, rhs_ty };
            const lhs_mcv = try self.resolveInst(operands[sources[0].?]);
            const rhs_mcv = try self.resolveInst(operands[sources[1].?]);

            const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                self.reuseOperand(inst, operands[sources[0].?], sources[0].?, lhs_mcv))
                lhs_mcv
            else if (has_avx and lhs_mcv.isRegister())
                .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
            else
                try self.copyToRegisterWithInstTracking(inst, operand_tys[sources[0].?], lhs_mcv);
            const dst_reg = dst_mcv.getReg().?;
            const dst_alias = registerAlias(dst_reg, max_abi_size);

            if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                .{ .v_ps, .shuf },
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
                .u(control),
            ) else try self.asmRegisterRegisterRegisterImmediate(
                .{ .v_ps, .shuf },
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
                .u(control),
            ) else if (rhs_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                .{ ._ps, .shuf },
                dst_alias,
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
                .u(control),
            ) else try self.asmRegisterRegisterImmediate(
                .{ ._ps, .shuf },
                dst_alias,
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
                .u(control),
            );
            break :result dst_mcv;
        }

        shufpd: {
            if (elem_abi_size != 8) break :shufpd;
            if (max_abi_size > self.vectorSize(.float)) break :shufpd;

            var control: u4 = 0b0_0_0_0;
            var sources: [2]?u1 = @splat(null);
            for (mask_elems, 0..) |maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index: u2 = @intCast(if (mask_elem < 0) ~mask_elem else mask_elem);
                if (mask_elem_index & 0b10 != elem_index & 0b10) break :shufpd;

                const source = @intFromBool(mask_elem < 0);
                if (sources[elem_index & 0b01]) |prev_source| {
                    if (source != prev_source) break :shufpd;
                } else sources[elem_index & 0b01] = source;

                control |= @as(u4, @intCast(mask_elem_index & 0b01)) << @intCast(elem_index);
            }
            if (sources[0] orelse break :shufpd == sources[1] orelse break :shufpd) break :shufpd;

            const operands: [2]Air.Inst.Ref = .{ extra.a, extra.b };
            const operand_tys: [2]Type = .{ lhs_ty, rhs_ty };
            const lhs_mcv = try self.resolveInst(operands[sources[0].?]);
            const rhs_mcv = try self.resolveInst(operands[sources[1].?]);

            const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                self.reuseOperand(inst, operands[sources[0].?], sources[0].?, lhs_mcv))
                lhs_mcv
            else if (has_avx and lhs_mcv.isRegister())
                .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
            else
                try self.copyToRegisterWithInstTracking(inst, operand_tys[sources[0].?], lhs_mcv);
            const dst_reg = dst_mcv.getReg().?;
            const dst_alias = registerAlias(dst_reg, max_abi_size);

            if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                .{ .v_pd, .shuf },
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
                .u(control),
            ) else try self.asmRegisterRegisterRegisterImmediate(
                .{ .v_pd, .shuf },
                dst_alias,
                registerAlias(lhs_mcv.getReg() orelse dst_reg, max_abi_size),
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
                .u(control),
            ) else if (rhs_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                .{ ._pd, .shuf },
                dst_alias,
                try rhs_mcv.mem(self, .{ .size = .fromSize(max_abi_size) }),
                .u(control),
            ) else try self.asmRegisterRegisterImmediate(
                .{ ._pd, .shuf },
                dst_alias,
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(operand_tys[sources[1].?], rhs_mcv), max_abi_size),
                .u(control),
            );
            break :result dst_mcv;
        }

        blend: {
            if (elem_abi_size < 2) break :blend;
            if (dst_abi_size > self.vectorSize(.float)) break :blend;
            if (!self.hasFeature(.sse4_1)) break :blend;

            var control: u8 = 0b0_0_0_0_0_0_0_0;
            for (mask_elems, 0..) |maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index =
                    std.math.cast(u4, if (mask_elem < 0) ~mask_elem else mask_elem) orelse break :blend;
                if (mask_elem_index != elem_index) break :blend;

                const select_mask = @as(u8, @intFromBool(mask_elem < 0)) << @truncate(elem_index);
                if (elem_index & 0b1000 == 0)
                    control |= select_mask
                else if (control & @as(u8, 0b1) << @truncate(elem_index) != select_mask) break :blend;
            }

            if (!elem_ty.isRuntimeFloat() and self.hasFeature(.avx2)) vpblendd: {
                const expanded_control = switch (elem_abi_size) {
                    4 => control,
                    8 => @as(u8, if (control & 0b0001 != 0) 0b00_00_00_11 else 0b00_00_00_00) |
                        @as(u8, if (control & 0b0010 != 0) 0b00_00_11_00 else 0b00_00_00_00) |
                        @as(u8, if (control & 0b0100 != 0) 0b00_11_00_00 else 0b00_00_00_00) |
                        @as(u8, if (control & 0b1000 != 0) 0b11_00_00_00 else 0b00_00_00_00),
                    else => break :vpblendd,
                };

                const lhs_mcv = try self.resolveInst(extra.a);
                const lhs_reg = if (lhs_mcv.isRegister())
                    lhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(dst_ty, lhs_mcv);
                const lhs_lock = self.register_manager.lockReg(lhs_reg);
                defer if (lhs_lock) |lock| self.register_manager.unlockReg(lock);

                const rhs_mcv = try self.resolveInst(extra.b);
                const dst_reg = try self.register_manager.allocReg(inst, abi.RegisterClass.sse);
                if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                    .{ .vp_d, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    registerAlias(lhs_reg, dst_abi_size),
                    try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                    .u(expanded_control),
                ) else try self.asmRegisterRegisterRegisterImmediate(
                    .{ .vp_d, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    registerAlias(lhs_reg, dst_abi_size),
                    registerAlias(if (rhs_mcv.isRegister())
                        rhs_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                    .u(expanded_control),
                );
                break :result .{ .register = dst_reg };
            }

            if (!elem_ty.isRuntimeFloat() or elem_abi_size == 2) pblendw: {
                const expanded_control = switch (elem_abi_size) {
                    2 => control,
                    4 => if (dst_abi_size <= 16 or
                        @as(u4, @intCast(control >> 4)) == @as(u4, @truncate(control >> 0)))
                        @as(u8, if (control & 0b0001 != 0) 0b00_00_00_11 else 0b00_00_00_00) |
                            @as(u8, if (control & 0b0010 != 0) 0b00_00_11_00 else 0b00_00_00_00) |
                            @as(u8, if (control & 0b0100 != 0) 0b00_11_00_00 else 0b00_00_00_00) |
                            @as(u8, if (control & 0b1000 != 0) 0b11_00_00_00 else 0b00_00_00_00)
                    else
                        break :pblendw,
                    8 => if (dst_abi_size <= 16 or
                        @as(u2, @intCast(control >> 2)) == @as(u2, @truncate(control >> 0)))
                        @as(u8, if (control & 0b01 != 0) 0b0000_1111 else 0b0000_0000) |
                            @as(u8, if (control & 0b10 != 0) 0b1111_0000 else 0b0000_0000)
                    else
                        break :pblendw,
                    16 => break :pblendw,
                    else => unreachable,
                };

                const lhs_mcv = try self.resolveInst(extra.a);
                const rhs_mcv = try self.resolveInst(extra.b);

                const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                    self.reuseOperand(inst, extra.a, 0, lhs_mcv))
                    lhs_mcv
                else if (has_avx and lhs_mcv.isRegister())
                    .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
                else
                    try self.copyToRegisterWithInstTracking(inst, dst_ty, lhs_mcv);
                const dst_reg = dst_mcv.getReg().?;

                if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                    .{ .vp_w, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    registerAlias(if (lhs_mcv.isRegister())
                        lhs_mcv.getReg().?
                    else
                        dst_reg, dst_abi_size),
                    try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                    .u(expanded_control),
                ) else try self.asmRegisterRegisterRegisterImmediate(
                    .{ .vp_w, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    registerAlias(if (lhs_mcv.isRegister())
                        lhs_mcv.getReg().?
                    else
                        dst_reg, dst_abi_size),
                    registerAlias(if (rhs_mcv.isRegister())
                        rhs_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                    .u(expanded_control),
                ) else if (rhs_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                    .{ .p_w, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                    .u(expanded_control),
                ) else try self.asmRegisterRegisterImmediate(
                    .{ .p_w, .blend },
                    registerAlias(dst_reg, dst_abi_size),
                    registerAlias(if (rhs_mcv.isRegister())
                        rhs_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                    .u(expanded_control),
                );
                break :result .{ .register = dst_reg };
            }

            const expanded_control = switch (elem_abi_size) {
                4, 8 => control,
                16 => @as(u4, if (control & 0b01 != 0) 0b00_11 else 0b00_00) |
                    @as(u4, if (control & 0b10 != 0) 0b11_00 else 0b00_00),
                else => unreachable,
            };

            const lhs_mcv = try self.resolveInst(extra.a);
            const rhs_mcv = try self.resolveInst(extra.b);

            const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                self.reuseOperand(inst, extra.a, 0, lhs_mcv))
                lhs_mcv
            else if (has_avx and lhs_mcv.isRegister())
                .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
            else
                try self.copyToRegisterWithInstTracking(inst, dst_ty, lhs_mcv);
            const dst_reg = dst_mcv.getReg().?;

            if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryImmediate(
                switch (elem_abi_size) {
                    4 => .{ .v_ps, .blend },
                    8, 16 => .{ .v_pd, .blend },
                    else => unreachable,
                },
                registerAlias(dst_reg, dst_abi_size),
                registerAlias(if (lhs_mcv.isRegister())
                    lhs_mcv.getReg().?
                else
                    dst_reg, dst_abi_size),
                try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                .u(expanded_control),
            ) else try self.asmRegisterRegisterRegisterImmediate(
                switch (elem_abi_size) {
                    4 => .{ .v_ps, .blend },
                    8, 16 => .{ .v_pd, .blend },
                    else => unreachable,
                },
                registerAlias(dst_reg, dst_abi_size),
                registerAlias(if (lhs_mcv.isRegister())
                    lhs_mcv.getReg().?
                else
                    dst_reg, dst_abi_size),
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                .u(expanded_control),
            ) else if (rhs_mcv.isBase()) try self.asmRegisterMemoryImmediate(
                switch (elem_abi_size) {
                    4 => .{ ._ps, .blend },
                    8, 16 => .{ ._pd, .blend },
                    else => unreachable,
                },
                registerAlias(dst_reg, dst_abi_size),
                try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                .u(expanded_control),
            ) else try self.asmRegisterRegisterImmediate(
                switch (elem_abi_size) {
                    4 => .{ ._ps, .blend },
                    8, 16 => .{ ._pd, .blend },
                    else => unreachable,
                },
                registerAlias(dst_reg, dst_abi_size),
                registerAlias(if (rhs_mcv.isRegister())
                    rhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                .u(expanded_control),
            );
            break :result .{ .register = dst_reg };
        }

        blendv: {
            if (dst_abi_size > self.vectorSize(if (elem_abi_size >= 4) .float else .int)) break :blendv;

            const select_mask_elem_ty = try pt.intType(.unsigned, elem_abi_size * 8);
            const select_mask_ty = try pt.vectorType(.{
                .len = @intCast(mask_elems.len),
                .child = select_mask_elem_ty.toIntern(),
            });
            var select_mask_elems: [32]InternPool.Index = undefined;
            for (
                select_mask_elems[0..mask_elems.len],
                mask_elems,
                0..,
            ) |*select_mask_elem, maybe_mask_elem, elem_index| {
                const mask_elem = maybe_mask_elem orelse continue;
                const mask_elem_index =
                    std.math.cast(u5, if (mask_elem < 0) ~mask_elem else mask_elem) orelse break :blendv;
                if (mask_elem_index != elem_index) break :blendv;

                select_mask_elem.* = (if (mask_elem < 0)
                    try select_mask_elem_ty.maxIntScalar(pt, select_mask_elem_ty)
                else
                    try select_mask_elem_ty.minIntScalar(pt, select_mask_elem_ty)).toIntern();
            }
            const select_mask_mcv = try self.genTypedValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                .ty = select_mask_ty.toIntern(),
                .storage = .{ .elems = select_mask_elems[0..mask_elems.len] },
            } })));

            if (self.hasFeature(.sse4_1)) {
                const mir_tag: Mir.Inst.FixedTag = .{
                    if ((elem_abi_size >= 4 and elem_ty.isRuntimeFloat()) or
                        (dst_abi_size > 16 and !self.hasFeature(.avx2))) switch (elem_abi_size) {
                        4 => if (has_avx) .v_ps else ._ps,
                        8 => if (has_avx) .v_pd else ._pd,
                        else => unreachable,
                    } else if (has_avx) .vp_b else .p_b,
                    .blendv,
                };

                const select_mask_reg = if (!has_avx) reg: {
                    try self.register_manager.getKnownReg(.xmm0, null);
                    try self.genSetReg(.xmm0, select_mask_elem_ty, select_mask_mcv, .{});
                    break :reg .xmm0;
                } else try self.copyToTmpRegister(select_mask_ty, select_mask_mcv);
                const select_mask_alias = registerAlias(select_mask_reg, dst_abi_size);
                const select_mask_lock = self.register_manager.lockRegAssumeUnused(select_mask_reg);
                defer self.register_manager.unlockReg(select_mask_lock);

                const lhs_mcv = try self.resolveInst(extra.a);
                const rhs_mcv = try self.resolveInst(extra.b);

                const dst_mcv: MCValue = if (lhs_mcv.isRegister() and
                    self.reuseOperand(inst, extra.a, 0, lhs_mcv))
                    lhs_mcv
                else if (has_avx and lhs_mcv.isRegister())
                    .{ .register = try self.register_manager.allocReg(inst, abi.RegisterClass.sse) }
                else
                    try self.copyToRegisterWithInstTracking(inst, dst_ty, lhs_mcv);
                const dst_reg = dst_mcv.getReg().?;
                const dst_alias = registerAlias(dst_reg, dst_abi_size);

                if (has_avx) if (rhs_mcv.isBase()) try self.asmRegisterRegisterMemoryRegister(
                    mir_tag,
                    dst_alias,
                    if (lhs_mcv.isRegister())
                        registerAlias(lhs_mcv.getReg().?, dst_abi_size)
                    else
                        dst_alias,
                    try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                    select_mask_alias,
                ) else try self.asmRegisterRegisterRegisterRegister(
                    mir_tag,
                    dst_alias,
                    if (lhs_mcv.isRegister())
                        registerAlias(lhs_mcv.getReg().?, dst_abi_size)
                    else
                        dst_alias,
                    registerAlias(if (rhs_mcv.isRegister())
                        rhs_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                    select_mask_alias,
                ) else if (rhs_mcv.isBase()) try self.asmRegisterMemoryRegister(
                    mir_tag,
                    dst_alias,
                    try rhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
                    select_mask_alias,
                ) else try self.asmRegisterRegisterRegister(
                    mir_tag,
                    dst_alias,
                    registerAlias(if (rhs_mcv.isRegister())
                        rhs_mcv.getReg().?
                    else
                        try self.copyToTmpRegister(dst_ty, rhs_mcv), dst_abi_size),
                    select_mask_alias,
                );
                break :result dst_mcv;
            }

            const lhs_mcv = try self.resolveInst(extra.a);
            const rhs_mcv = try self.resolveInst(extra.b);

            const dst_mcv: MCValue = if (rhs_mcv.isRegister() and
                self.reuseOperand(inst, extra.b, 1, rhs_mcv))
                rhs_mcv
            else
                try self.copyToRegisterWithInstTracking(inst, dst_ty, rhs_mcv);
            const dst_reg = dst_mcv.getReg().?;
            const dst_alias = registerAlias(dst_reg, dst_abi_size);

            const mask_reg = try self.copyToTmpRegister(select_mask_ty, select_mask_mcv);
            const mask_alias = registerAlias(mask_reg, dst_abi_size);
            const mask_lock = self.register_manager.lockRegAssumeUnused(mask_reg);
            defer self.register_manager.unlockReg(mask_lock);

            const mir_fixes: Mir.Inst.Fixes = if (elem_ty.isRuntimeFloat())
                switch (elem_ty.floatBits(self.target.*)) {
                    16, 80, 128 => .p_,
                    32 => ._ps,
                    64 => ._pd,
                    else => unreachable,
                }
            else
                .p_;
            try self.asmRegisterRegister(.{ mir_fixes, .@"and" }, dst_alias, mask_alias);
            if (lhs_mcv.isBase()) try self.asmRegisterMemory(
                .{ mir_fixes, .andn },
                mask_alias,
                try lhs_mcv.mem(self, .{ .size = .fromSize(dst_abi_size) }),
            ) else try self.asmRegisterRegister(
                .{ mir_fixes, .andn },
                mask_alias,
                if (lhs_mcv.isRegister())
                    lhs_mcv.getReg().?
                else
                    try self.copyToTmpRegister(dst_ty, lhs_mcv),
            );
            try self.asmRegisterRegister(.{ mir_fixes, .@"or" }, dst_alias, mask_alias);
            break :result dst_mcv;
        }

        pshufb: {
            if (max_abi_size > 16) break :pshufb;
            if (!self.hasFeature(.ssse3)) break :pshufb;

            const temp_regs =
                try self.register_manager.allocRegs(2, .{ inst, null }, abi.RegisterClass.sse);
            const temp_locks = self.register_manager.lockRegsAssumeUnused(2, temp_regs);
            defer for (temp_locks) |lock| self.register_manager.unlockReg(lock);

            const lhs_temp_alias = registerAlias(temp_regs[0], max_abi_size);
            try self.genSetReg(temp_regs[0], lhs_ty, .{ .air_ref = extra.a }, .{});

            const rhs_temp_alias = registerAlias(temp_regs[1], max_abi_size);
            try self.genSetReg(temp_regs[1], rhs_ty, .{ .air_ref = extra.b }, .{});

            var lhs_mask_elems: [16]InternPool.Index = undefined;
            for (lhs_mask_elems[0..max_abi_size], 0..) |*lhs_mask_elem, byte_index| {
                const elem_index = byte_index / elem_abi_size;
                lhs_mask_elem.* = try pt.intern(.{ .int = .{
                    .ty = .u8_type,
                    .storage = .{ .u64 = if (elem_index >= mask_elems.len) 0b1_00_00000 else elem: {
                        const mask_elem = mask_elems[elem_index] orelse break :elem 0b1_00_00000;
                        if (mask_elem < 0) break :elem 0b1_00_00000;
                        const mask_elem_index: u31 = @intCast(mask_elem);
                        const byte_off: u32 = @intCast(byte_index % elem_abi_size);
                        break :elem @intCast(mask_elem_index * elem_abi_size + byte_off);
                    } },
                } });
            }
            const lhs_mask_ty = try pt.vectorType(.{ .len = max_abi_size, .child = .u8_type });
            const lhs_mask_mcv = try self.genTypedValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                .ty = lhs_mask_ty.toIntern(),
                .storage = .{ .elems = lhs_mask_elems[0..max_abi_size] },
            } })));
            const lhs_mask_mem: Memory = .{
                .base = .{ .reg = try self.copyToTmpRegister(.usize, lhs_mask_mcv.address()) },
                .mod = .{ .rm = .{ .size = .fromSize(@max(max_abi_size, 16)) } },
            };
            if (has_avx) try self.asmRegisterRegisterMemory(
                .{ .vp_b, .shuf },
                lhs_temp_alias,
                lhs_temp_alias,
                lhs_mask_mem,
            ) else try self.asmRegisterMemory(
                .{ .p_b, .shuf },
                lhs_temp_alias,
                lhs_mask_mem,
            );

            var rhs_mask_elems: [16]InternPool.Index = undefined;
            for (rhs_mask_elems[0..max_abi_size], 0..) |*rhs_mask_elem, byte_index| {
                const elem_index = byte_index / elem_abi_size;
                rhs_mask_elem.* = try pt.intern(.{ .int = .{
                    .ty = .u8_type,
                    .storage = .{ .u64 = if (elem_index >= mask_elems.len) 0b1_00_00000 else elem: {
                        const mask_elem = mask_elems[elem_index] orelse break :elem 0b1_00_00000;
                        if (mask_elem >= 0) break :elem 0b1_00_00000;
                        const mask_elem_index: u31 = @intCast(~mask_elem);
                        const byte_off: u32 = @intCast(byte_index % elem_abi_size);
                        break :elem @intCast(mask_elem_index * elem_abi_size + byte_off);
                    } },
                } });
            }
            const rhs_mask_ty = try pt.vectorType(.{ .len = max_abi_size, .child = .u8_type });
            const rhs_mask_mcv = try self.genTypedValue(.fromInterned(try pt.intern(.{ .aggregate = .{
                .ty = rhs_mask_ty.toIntern(),
                .storage = .{ .elems = rhs_mask_elems[0..max_abi_size] },
            } })));
            const rhs_mask_mem: Memory = .{
                .base = .{ .reg = try self.copyToTmpRegister(.usize, rhs_mask_mcv.address()) },
                .mod = .{ .rm = .{ .size = .fromSize(@max(max_abi_size, 16)) } },
            };
            if (has_avx) try self.asmRegisterRegisterMemory(
                .{ .vp_b, .shuf },
                rhs_temp_alias,
                rhs_temp_alias,
                rhs_mask_mem,
            ) else try self.asmRegisterMemory(
                .{ .p_b, .shuf },
                rhs_temp_alias,
                rhs_mask_mem,
            );

            if (has_avx) try self.asmRegisterRegisterRegister(
                .{ switch (elem_ty.zigTypeTag(zcu)) {
                    else => break :result null,
                    .int => .vp_,
                    .float => switch (elem_ty.floatBits(self.target.*)) {
                        32 => .v_ps,
                        64 => .v_pd,
                        16, 80, 128 => break :result null,
                        else => unreachable,
                    },
                }, .@"or" },
                lhs_temp_alias,
                lhs_temp_alias,
                rhs_temp_alias,
            ) else try self.asmRegisterRegister(
                .{ switch (elem_ty.zigTypeTag(zcu)) {
                    else => break :result null,
                    .int => .p_,
                    .float => switch (elem_ty.floatBits(self.target.*)) {
                        32 => ._ps,
                        64 => ._pd,
                        16, 80, 128 => break :result null,
                        else => unreachable,
                    },
                }, .@"or" },
                lhs_temp_alias,
                rhs_temp_alias,
            );
            break :result .{ .register = temp_regs[0] };
        }

        break :result null;
    }) orelse return self.fail("TODO implement airShuffle from {} and {} to {} with {}", .{
        lhs_ty.fmt(pt),
        rhs_ty.fmt(pt),
        dst_ty.fmt(pt),
        Value.fromInterned(extra.mask).fmtValue(pt),
    });
    return self.finishAir(inst, result, .{ extra.a, extra.b, .none });
}
```
嗯，这段代码看起来是Zig编程语言中的一个函数，名为`airShuffle`，可能属于某个编译器或代码生成器的部分。让我仔细读一下代码，尝试理解它的功能。

首先，函数接收一个`CodeGen`结构体的指针`self`和一个`Air.Inst.Index`类型的指令索引`inst`。返回类型是`!void`，表示可能返回错误。函数的目标可能是处理某种向量洗牌（shuffle）操作，生成对应的机器码。

接下来，代码中定义了一些变量，比如`pt`、`zcu`，然后通过`self.air`获取指令的数据。这里涉及到`Air.Shuffle`结构体，可能表示向量洗牌操作的额外数据。接着，代码获取目标类型`dst_ty`，并提取其子类型`elem_ty`，以及计算各种ABI大小，如`elem_abi_size`、`dst_abi_size`等。这些大小可能与寄存器的使用和内存对齐有关。

然后，代码创建了一个栈分配的缓冲区`stack`，用于临时存储掩码元素。通过`allocator`分配内存来存储掩码元素`mask_elems`，并在之后循环处理这些元素。这里，`mask_elems`可能表示洗牌操作的掩码，每个元素要么是未定义的（`null`），要么是一个有符号整数。

接下来，代码检查是否支持AVX指令集（`has_avx`），然后进入一个复杂的逻辑块，使用多个`for`循环和条件判断来处理不同的洗牌情况。例如，处理`unpckl`和`unpckh`指令变体，检查元素大小和对齐，生成相应的机器指令。

代码中还涉及到多种SSE和AVX指令的处理，如`pshufd`、`shufps`、`shufpd`、`blend`、`blendv`、`pshufb`等。每种情况都有特定的条件和生成相应机器码的逻辑。例如，在`pshufd`块中，处理4字节元素的情况，生成控制字`control`，并选择合适的源操作数。

最后，如果所有上述情况都无法处理，函数会返回一个错误，提示需要实现未处理的洗牌情况。

整个函数的逻辑看起来是根据不同的洗牌模式和目标架构特性，选择最优的机器指令来生成高效的代码。这需要对向量操作、指令集以及Zig编译器的内部结构有深入的理解。

在理解过程中，我可能会遇到一些难点，比如：

1. **寄存器分配和管理**：代码中多次使用`register_manager`来分配和释放寄存器，需要理解Zig的寄存器分配策略。
2. **指令选择逻辑**：不同的洗牌模式对应不同的机器指令，如何根据元素类型、大小和目标架构选择正确的指令。
3. **掩码处理**：如何将高级的向量掩码转换为底层机器指令所需的控制字或掩码寄存器。
4. **错误处理**：函数中多处使用`try`和错误返回，需要确保所有可能的错误情况都被妥善处理。

此外，代码中使用的`MCValue`类型可能表示机器码值，包括寄存器、内存地址等，需要明确其结构和用途。

总结来说，这个函数的核心思维模型是：根据输入的洗牌指令和架构特性，分析元素类型、大小和掩码，选择并生成最优的机器指令序列，同时处理寄存器分配和错误情况。
================================================
这段代码实现了一个向量洗牌操作的代码生成逻辑，其核心思维模型可总结如下：

---

### **核心目标**
根据输入的向量洗牌指令（`Air.Shuffle`），结合目标架构的指令集特性（如SSE、AVX等），生成高效的机器指令序列，完成向量元素的重新排列。

---

### **关键逻辑分层**
1. **元数据提取**  
   - 从中间表示（AIR）中提取洗牌指令的元数据，包括：
     - 目标向量类型（`dst_ty`）及其元素类型（`elem_ty`）
     - 左右操作数的类型（`lhs_ty`、`rhs_ty`）及ABI大小
     - 洗牌掩码（`mask_elems`）的解析（可能包含未定义值或具体索引）

2. **指令选择策略**  
   根据以下条件选择最优指令：
   - **元素大小**（`elem_abi_size`）：如4字节（`pshufd`）、8字节（`shufpd`）等。
   - **向量宽度**（`max_abi_size`）：是否超过SSE/AVX支持的向量长度（如128位、256位）。
   - **架构特性**（`has_avx`、`.sse4_1`等）：支持哪些扩展指令集。
   - **掩码模式**：是否满足特定指令的掩码约束（如连续索引、对称性等）。

3. **指令生成路径**  
   - **简单复制**：若掩码为恒等映射（如`0,1,2,3`）或反向映射（如`~0, ~1`），直接复用操作数或生成拷贝。
   - **Unpack指令**（`unpckl`/`unpckh`）：处理高低位交错的洗牌模式。
   - **Shuffle指令**（`pshufd`、`shufps`、`shufpd`）：针对特定元素大小和布局生成控制字。
   - **Blend指令**：根据掩码混合两个向量的元素。
   - **Blendv指令**：动态掩码混合（需生成掩码向量并配合逻辑运算）。
   - **PSHUFB指令**：通用字节级洗牌（需构造掩码向量）。

4. **寄存器管理**  
   - 动态分配/复用寄存器（`allocRegOrMem`、`reuseOperand`）。
   - 处理操作数来源（寄存器/内存），必要时生成临时副本。
   - 对AVX指令的3操作数形式（目标寄存器独立）与SSE的2操作数形式做适配。

5. **异常处理**  
   - 若所有路径均不匹配，抛出未实现错误，提示开发者扩展支持。

---

### **核心优化点**
- **指令级优化**：优先选择更高效的指令（如`blend`优于`blendv`）。
- **寄存器复用**：避免冗余拷贝（如`reuseOperand`直接复用输入寄存器）。
- **掩码分析**：通过静态分析掩码模式，匹配指令的隐式控制逻辑（如`pshufd`的控制字生成）。

---

### **关键数据结构**
- **MCValue**：表示机器码值（寄存器、内存地址等）。
- **MaskElems**：洗牌掩码的解析结果（可能为`null`表示未定义，或具体索引值）。
- **Mir.Inst**：底层机器指令的描述（操作码、操作数等）。

---

### **流程图概要**
```
开始
├─ 提取元数据（类型、大小、掩码）
├─ 检查恒等/反向掩码 → 生成拷贝
├─ 尝试Unpack指令 → 生成高低位交错
├─ 尝试Shuffle指令（按元素大小分支）→ 生成控制字
├─ 尝试Blend指令 → 生成掩码混合
├─ 尝试Blendv指令 → 动态掩码混合
├─ 尝试PSHUFB指令 → 构造字节掩码
└─ 若无匹配 → 报错（未实现）
结束
```

---

### **总结**
该函数通过分层条件判断，将高级向量洗牌操作逐级映射到底层机器指令，充分利用目标架构特性，同时兼顾寄存器分配和代码效率。其核心是对洗牌模式的静态分析与指令选择的精准匹配。