```zig
fn airAsm(self: *CodeGen, inst: Air.Inst.Index) !void {
    const pt = self.pt;
    const zcu = pt.zcu;
    const ty_pl = self.air.instructions.items(.data)[@intFromEnum(inst)].ty_pl;
    const extra = self.air.extraData(Air.Asm, ty_pl.payload);
    const clobbers_len: u31 = @truncate(extra.data.flags);
    var extra_i: usize = extra.end;
    const outputs: []const Air.Inst.Ref =
        @ptrCast(self.air.extra[extra_i..][0..extra.data.outputs_len]);
    extra_i += outputs.len;
    const inputs: []const Air.Inst.Ref = @ptrCast(self.air.extra[extra_i..][0..extra.data.inputs_len]);
    extra_i += inputs.len;

    var result: MCValue = .none;
    var args: std.ArrayList(MCValue) = .init(self.gpa);
    try args.ensureTotalCapacity(outputs.len + inputs.len);
    defer {
        for (args.items) |arg| if (arg.getReg()) |reg| self.register_manager.unlockReg(.{
            .tracked_index = RegisterManager.indexOfRegIntoTracked(reg) orelse continue,
        });
        args.deinit();
    }
    var arg_map: std.StringHashMap(u8) = .init(self.gpa);
    try arg_map.ensureTotalCapacity(@intCast(outputs.len + inputs.len));
    defer arg_map.deinit();

    var outputs_extra_i = extra_i;
    for (outputs) |output| {
        const extra_bytes = std.mem.sliceAsBytes(self.air.extra[extra_i..]);
        const constraint = std.mem.sliceTo(std.mem.sliceAsBytes(self.air.extra[extra_i..]), 0);
        const name = std.mem.sliceTo(extra_bytes[constraint.len + 1 ..], 0);
        // This equation accounts for the fact that even if we have exactly 4 bytes
        // for the string, we still use the next u32 for the null terminator.
        extra_i += (constraint.len + name.len + (2 + 3)) / 4;

        const maybe_inst = switch (output) {
            .none => inst,
            else => null,
        };
        const ty = switch (output) {
            .none => self.typeOfIndex(inst),
            else => self.typeOf(output).childType(zcu),
        };
        const is_read = switch (constraint[0]) {
            '=' => false,
            '+' => read: {
                if (output == .none) return self.fail(
                    "read-write constraint unsupported for asm result: '{s}'",
                    .{constraint},
                );
                break :read true;
            },
            else => return self.fail("invalid constraint: '{s}'", .{constraint}),
        };
        const is_early_clobber = constraint[1] == '&';
        const rest = constraint[@as(usize, 1) + @intFromBool(is_early_clobber) ..];
        const arg_mcv: MCValue = arg_mcv: {
            const arg_maybe_reg: ?Register = if (std.mem.eql(u8, rest, "r") or
                std.mem.eql(u8, rest, "f") or std.mem.eql(u8, rest, "x"))
                registerAlias(
                    self.register_manager.tryAllocReg(maybe_inst, switch (rest[0]) {
                        'r' => abi.RegisterClass.gp,
                        'f' => abi.RegisterClass.x87,
                        'x' => abi.RegisterClass.sse,
                        else => unreachable,
                    }) orelse return self.fail("ran out of registers lowering inline asm", .{}),
                    @intCast(ty.abiSize(zcu)),
                )
            else if (std.mem.eql(u8, rest, "m"))
                if (output != .none) null else return self.fail(
                    "memory constraint unsupported for asm result: '{s}'",
                    .{constraint},
                )
            else if (std.mem.eql(u8, rest, "g") or
                std.mem.eql(u8, rest, "rm") or std.mem.eql(u8, rest, "mr") or
                std.mem.eql(u8, rest, "r,m") or std.mem.eql(u8, rest, "m,r"))
                self.register_manager.tryAllocReg(maybe_inst, abi.RegisterClass.gp) orelse
                    if (output != .none)
                        null
                    else
                        return self.fail("ran out of registers lowering inline asm", .{})
            else if (std.mem.startsWith(u8, rest, "{") and std.mem.endsWith(u8, rest, "}"))
                parseRegName(rest["{".len .. rest.len - "}".len]) orelse
                    return self.fail("invalid register constraint: '{s}'", .{constraint})
            else if (rest.len == 1 and std.ascii.isDigit(rest[0])) {
                const index = std.fmt.charToDigit(rest[0], 10) catch unreachable;
                if (index >= args.items.len) return self.fail("constraint out of bounds: '{s}'", .{
                    constraint,
                });
                break :arg_mcv args.items[index];
            } else return self.fail("invalid constraint: '{s}'", .{constraint});
            break :arg_mcv if (arg_maybe_reg) |reg| .{ .register = reg } else arg: {
                const ptr_mcv = try self.resolveInst(output);
                switch (ptr_mcv) {
                    .immediate => |addr| if (std.math.cast(i32, @as(i64, @bitCast(addr)))) |_|
                        break :arg ptr_mcv.deref(),
                    .register, .register_offset, .lea_frame => break :arg ptr_mcv.deref(),
                    else => {},
                }
                break :arg .{ .indirect = .{ .reg = try self.copyToTmpRegister(.usize, ptr_mcv) } };
            };
        };
        if (arg_mcv.getReg()) |reg| if (RegisterManager.indexOfRegIntoTracked(reg)) |tracked_index| {
            try self.register_manager.getRegIndex(tracked_index, if (output == .none) inst else null);
            _ = self.register_manager.lockRegIndexAssumeUnused(tracked_index);
        };
        if (!std.mem.eql(u8, name, "_"))
            arg_map.putAssumeCapacityNoClobber(name, @intCast(args.items.len));
        args.appendAssumeCapacity(arg_mcv);
        if (output == .none) result = arg_mcv;
        if (is_read) try self.load(arg_mcv, self.typeOf(output), .{ .air_ref = output });
    }

    for (inputs) |input| {
        const input_bytes = std.mem.sliceAsBytes(self.air.extra[extra_i..]);
        const constraint = std.mem.sliceTo(input_bytes, 0);
        const name = std.mem.sliceTo(input_bytes[constraint.len + 1 ..], 0);
        // This equation accounts for the fact that even if we have exactly 4 bytes
        // for the string, we still use the next u32 for the null terminator.
        extra_i += (constraint.len + name.len + (2 + 3)) / 4;

        const ty = self.typeOf(input);
        const input_mcv = try self.resolveInst(input);
        const arg_mcv: MCValue = if (std.mem.eql(u8, constraint, "r") or
            std.mem.eql(u8, constraint, "f") or std.mem.eql(u8, constraint, "x"))
        arg: {
            const rc = switch (constraint[0]) {
                'r' => abi.RegisterClass.gp,
                'f' => abi.RegisterClass.x87,
                'x' => abi.RegisterClass.sse,
                else => unreachable,
            };
            if (input_mcv.isRegister() and
                rc.isSet(RegisterManager.indexOfRegIntoTracked(input_mcv.getReg().?).?))
                break :arg input_mcv;
            const reg = try self.register_manager.allocReg(null, rc);
            try self.genSetReg(reg, ty, input_mcv, .{});
            break :arg .{ .register = registerAlias(reg, @intCast(ty.abiSize(zcu))) };
        } else if (std.mem.eql(u8, constraint, "i") or std.mem.eql(u8, constraint, "n"))
            switch (input_mcv) {
                .immediate => |imm| .{ .immediate = imm },
                else => return self.fail("immediate operand requires comptime value: '{s}'", .{
                    constraint,
                }),
            }
        else if (std.mem.eql(u8, constraint, "m")) arg: {
            switch (input_mcv) {
                .memory => |addr| if (std.math.cast(i32, @as(i64, @bitCast(addr)))) |_|
                    break :arg input_mcv,
                .indirect, .load_frame => break :arg input_mcv,
                .load_symbol, .load_direct, .load_got, .load_tlv => {},
                else => {
                    const temp_mcv = try self.allocTempRegOrMem(ty, false);
                    try self.genCopy(ty, temp_mcv, input_mcv, .{});
                    break :arg temp_mcv;
                },
            }
            const addr_reg = self.register_manager.tryAllocReg(null, abi.RegisterClass.gp) orelse {
                const temp_mcv = try self.allocTempRegOrMem(ty, false);
                try self.genCopy(ty, temp_mcv, input_mcv, .{});
                break :arg temp_mcv;
            };
            try self.genSetReg(addr_reg, .usize, input_mcv.address(), .{});
            break :arg .{ .indirect = .{ .reg = addr_reg } };
        } else if (std.mem.eql(u8, constraint, "g") or
            std.mem.eql(u8, constraint, "rm") or std.mem.eql(u8, constraint, "mr") or
            std.mem.eql(u8, constraint, "r,m") or std.mem.eql(u8, constraint, "m,r"))
        arg: {
            switch (input_mcv) {
                .register, .indirect, .load_frame => break :arg input_mcv,
                .memory => |addr| if (std.math.cast(i32, @as(i64, @bitCast(addr)))) |_|
                    break :arg input_mcv,
                else => {},
            }
            const temp_mcv = try self.allocTempRegOrMem(ty, true);
            try self.genCopy(ty, temp_mcv, input_mcv, .{});
            break :arg temp_mcv;
        } else if (std.mem.eql(u8, constraint, "X"))
            input_mcv
        else if (std.mem.startsWith(u8, constraint, "{") and std.mem.endsWith(u8, constraint, "}")) arg: {
            const reg = parseRegName(constraint["{".len .. constraint.len - "}".len]) orelse
                return self.fail("invalid register constraint: '{s}'", .{constraint});
            try self.register_manager.getReg(reg, null);
            try self.genSetReg(reg, ty, input_mcv, .{});
            break :arg .{ .register = reg };
        } else if (constraint.len == 1 and std.ascii.isDigit(constraint[0])) arg: {
            const index = std.fmt.charToDigit(constraint[0], 10) catch unreachable;
            if (index >= args.items.len) return self.fail("constraint out of bounds: '{s}'", .{constraint});
            try self.genCopy(ty, args.items[index], input_mcv, .{});
            break :arg args.items[index];
        } else return self.fail("invalid constraint: '{s}'", .{constraint});
        if (arg_mcv.getReg()) |reg| if (RegisterManager.indexOfRegIntoTracked(reg)) |_| {
            _ = self.register_manager.lockReg(reg);
        };
        if (!std.mem.eql(u8, name, "_"))
            arg_map.putAssumeCapacityNoClobber(name, @intCast(args.items.len));
        args.appendAssumeCapacity(arg_mcv);
    }

    {
        var clobber_i: u32 = 0;
        while (clobber_i < clobbers_len) : (clobber_i += 1) {
            const clobber = std.mem.sliceTo(std.mem.sliceAsBytes(self.air.extra[extra_i..]), 0);
            // This equation accounts for the fact that even if we have exactly 4 bytes
            // for the string, we still use the next u32 for the null terminator.
            extra_i += clobber.len / 4 + 1;

            if (std.mem.eql(u8, clobber, "") or std.mem.eql(u8, clobber, "memory")) {
                // ok, sure
            } else if (std.mem.eql(u8, clobber, "cc") or
                std.mem.eql(u8, clobber, "flags") or
                std.mem.eql(u8, clobber, "eflags") or
                std.mem.eql(u8, clobber, "rflags"))
            {
                try self.spillEflagsIfOccupied();
            } else {
                try self.register_manager.getReg(parseRegName(clobber) orelse
                    return self.fail("invalid clobber: '{s}'", .{clobber}), null);
            }
        }
    }

    const Label = struct {
        target: Mir.Inst.Index = undefined,
        pending_relocs: std.ArrayListUnmanaged(Mir.Inst.Index) = .empty,

        const Kind = enum { definition, reference };

        fn isValid(kind: Kind, name: []const u8) bool {
            for (name, 0..) |c, i| switch (c) {
                else => return false,
                '$' => if (i == 0) return false,
                '.' => {},
                '0'...'9' => if (i == 0) switch (kind) {
                    .definition => if (name.len != 1) return false,
                    .reference => {
                        if (name.len != 2) return false;
                        switch (name[1]) {
                            else => return false,
                            'B', 'F', 'b', 'f' => {},
                        }
                    },
                },
                '@', 'A'...'Z', '_', 'a'...'z' => {},
            };
            return name.len > 0;
        }
    };
    var labels: std.StringHashMapUnmanaged(Label) = .empty;
    defer {
        var label_it = labels.valueIterator();
        while (label_it.next()) |label| label.pending_relocs.deinit(self.gpa);
        labels.deinit(self.gpa);
    }

    const asm_source = std.mem.sliceAsBytes(self.air.extra[extra_i..])[0..extra.data.source_len];
    var line_it = std.mem.tokenizeAny(u8, asm_source, "\n\r;");
    next_line: while (line_it.next()) |line| {
        var mnem_it = std.mem.tokenizeAny(u8, line, " \t");
        var prefix: encoder.Instruction.Prefix = .none;
        const mnem_str = while (mnem_it.next()) |mnem_str| {
            if (mnem_str[0] == '#') continue :next_line;
            if (std.mem.startsWith(u8, mnem_str, "//")) continue :next_line;
            if (std.meta.stringToEnum(encoder.Instruction.Prefix, mnem_str)) |pre| {
                if (prefix != .none) return self.fail("extra prefix: '{s}'", .{mnem_str});
                prefix = pre;
                continue;
            }
            if (!std.mem.endsWith(u8, mnem_str, ":")) break mnem_str;
            const label_name = mnem_str[0 .. mnem_str.len - ":".len];
            if (!Label.isValid(.definition, label_name))
                return self.fail("invalid label: '{s}'", .{label_name});
            const label_gop = try labels.getOrPut(self.gpa, label_name);
            if (!label_gop.found_existing) label_gop.value_ptr.* = .{} else {
                const anon = std.ascii.isDigit(label_name[0]);
                if (!anon and label_gop.value_ptr.pending_relocs.items.len == 0)
                    return self.fail("redefined label: '{s}'", .{label_name});
                for (label_gop.value_ptr.pending_relocs.items) |pending_reloc|
                    self.performReloc(pending_reloc);
                if (anon)
                    label_gop.value_ptr.pending_relocs.clearRetainingCapacity()
                else
                    label_gop.value_ptr.pending_relocs.clearAndFree(self.gpa);
            }
            label_gop.value_ptr.target = @intCast(self.mir_instructions.len);
        } else continue;
        if (mnem_str[0] == '.') {
            if (prefix != .none) return self.fail("prefixed directive: '{s} {s}'", .{ @tagName(prefix), mnem_str });
            prefix = .directive;
        }

        var mnem_size: struct {
            used: bool,
            size: ?Memory.Size,
            fn use(size: *@This()) ?Memory.Size {
                size.used = true;
                return size.size;
            }
        } = .{
            .used = false,
            .size = if (prefix == .directive)
                null
            else if (std.mem.endsWith(u8, mnem_str, "b"))
                .byte
            else if (std.mem.endsWith(u8, mnem_str, "w"))
                .word
            else if (std.mem.endsWith(u8, mnem_str, "l"))
                .dword
            else if (std.mem.endsWith(u8, mnem_str, "q") and
                (std.mem.indexOfScalar(u8, "vp", mnem_str[0]) == null or !std.mem.endsWith(u8, mnem_str, "dq")))
                .qword
            else if (std.mem.endsWith(u8, mnem_str, "t"))
                .tbyte
            else
                null,
        };
        var mnem_tag = while (true) break std.meta.stringToEnum(
            encoder.Instruction.Mnemonic,
            mnem_str[0 .. mnem_str.len - @intFromBool(mnem_size.size != null)],
        ) orelse if (mnem_size.size) |_| {
            mnem_size.size = null;
            continue;
        } else return self.fail("invalid mnemonic: '{s}'", .{mnem_str});
        if (@as(?Memory.Size, switch (mnem_tag) {
            .clflush => .byte,
            .fldcw, .fnstcw, .fstcw, .fnstsw, .fstsw => .word,
            .fldenv, .fnstenv, .fstenv => .none,
            .frstor, .fsave, .fnsave, .fxrstor, .fxrstor64, .fxsave, .fxsave64 => .none,
            .invlpg => .none,
            .invpcid => .xword,
            .ldmxcsr, .stmxcsr, .vldmxcsr, .vstmxcsr => .dword,
            else => null,
        })) |fixed_mnem_size| {
            if (mnem_size.size) |size| if (size != fixed_mnem_size)
                return self.fail("invalid size: '{s}'", .{mnem_str});
            mnem_size.size = fixed_mnem_size;
        }

        var ops: [4]Operand = @splat(.none);
        var ops_len: usize = 0;

        var last_op = false;
        var op_it = std.mem.splitScalar(u8, mnem_it.rest(), ',');
        next_op: for (&ops) |*op| {
            const op_str = while (!last_op) {
                const full_str = op_it.next() orelse break :next_op;
                const code_str = if (std.mem.indexOfScalar(u8, full_str, '#') orelse
                    std.mem.indexOf(u8, full_str, "//")) |comment|
                code: {
                    last_op = true;
                    break :code full_str[0..comment];
                } else full_str;
                const trim_str = std.mem.trim(u8, code_str, " \t*");
                if (trim_str.len > 0) break trim_str;
            } else break;
            if (std.mem.startsWith(u8, op_str, "%%")) {
                const colon = std.mem.indexOfScalarPos(u8, op_str, "%%".len + 2, ':');
                const reg = parseRegName(op_str["%%".len .. colon orelse op_str.len]) orelse
                    return self.fail("invalid register: '{s}'", .{op_str});
                if (colon) |colon_pos| {
                    const disp = std.fmt.parseInt(i32, op_str[colon_pos + ":".len ..], 0) catch
                        return self.fail("invalid displacement: '{s}'", .{op_str});
                    op.* = .{ .mem = .{
                        .base = .{ .reg = reg },
                        .mod = .{ .rm = .{
                            .size = mnem_size.use() orelse
                                return self.fail("unknown size: '{s}'", .{op_str}),
                            .disp = disp,
                        } },
                    } };
                } else {
                    if (mnem_size.use()) |size| if (reg.bitSize() != size.bitSize(self.target))
                        return self.fail("invalid register size: '{s}'", .{op_str});
                    op.* = .{ .reg = reg };
                }
            } else if (std.mem.startsWith(u8, op_str, "%[") and std.mem.endsWith(u8, op_str, "]")) {
                const colon = std.mem.indexOfScalarPos(u8, op_str, "%[".len, ':');
                const modifier = if (colon) |colon_pos|
                    op_str[colon_pos + ":".len .. op_str.len - "]".len]
                else
                    "";
                op.* = switch (args.items[
                    arg_map.get(op_str["%[".len .. colon orelse op_str.len - "]".len]) orelse
                        return self.fail("no matching constraint: '{s}'", .{op_str})
                ]) {
                    .immediate => |imm| if (std.mem.eql(u8, modifier, "") or std.mem.eql(u8, modifier, "c"))
                        .{ .imm = .u(imm) }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .register => |reg| if (std.mem.eql(u8, modifier, ""))
                        .{ .reg = if (mnem_size.use()) |size|
                            registerAlias(reg, @intCast(@divExact(size.bitSize(self.target), 8)))
                        else
                            reg }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .memory => |addr| if (std.mem.eql(u8, modifier, "") or std.mem.eql(u8, modifier, "P"))
                        .{ .mem = .{
                            .base = .{ .reg = .ds },
                            .mod = .{ .rm = .{
                                .size = mnem_size.use() orelse
                                    return self.fail("unknown size: '{s}'", .{op_str}),
                                .disp = @intCast(@as(i64, @bitCast(addr))),
                            } },
                        } }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .indirect => |reg_off| if (std.mem.eql(u8, modifier, ""))
                        .{ .mem = .{
                            .base = .{ .reg = reg_off.reg },
                            .mod = .{ .rm = .{
                                .size = mnem_size.use() orelse
                                    return self.fail("unknown size: '{s}'", .{op_str}),
                                .disp = reg_off.off,
                            } },
                        } }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .load_frame => |frame_addr| if (std.mem.eql(u8, modifier, ""))
                        .{ .mem = .{
                            .base = .{ .frame = frame_addr.index },
                            .mod = .{ .rm = .{
                                .size = mnem_size.use() orelse
                                    return self.fail("unknown size: '{s}'", .{op_str}),
                                .disp = frame_addr.off,
                            } },
                        } }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .lea_got => |sym_index| if (std.mem.eql(u8, modifier, "P"))
                        .{ .reg = try self.copyToTmpRegister(.usize, .{ .lea_got = sym_index }) }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    .lea_symbol => |sym_off| if (std.mem.eql(u8, modifier, "P"))
                        .{ .reg = try self.copyToTmpRegister(.usize, .{ .lea_symbol = sym_off }) }
                    else
                        return self.fail("invalid modifier: '{s}'", .{modifier}),
                    else => return self.fail("invalid constraint: '{s}'", .{op_str}),
                };
            } else if (std.mem.startsWith(u8, op_str, "$")) {
                op.* = if (std.fmt.parseInt(u64, op_str["$".len..], 0)) |u|
                    .{ .imm = .u(u) }
                else |_| if (std.fmt.parseInt(i32, op_str["$".len..], 0)) |s|
                    .{ .imm = .s(s) }
                else |_|
                    return self.fail("invalid immediate: '{s}'", .{op_str});
            } else if (std.mem.endsWith(u8, op_str, ")")) {
                const open = std.mem.indexOfScalar(u8, op_str, '(') orelse
                    return self.fail("invalid operand: '{s}'", .{op_str});
                var sib_it = std.mem.splitScalar(u8, op_str[open + "(".len .. op_str.len - ")".len], ',');
                const base_str = sib_it.next() orelse
                    return self.fail("invalid memory operand: '{s}'", .{op_str});
                if (base_str.len > 0 and !std.mem.startsWith(u8, base_str, "%%"))
                    return self.fail("invalid memory operand: '{s}'", .{op_str});
                const index_str = sib_it.next() orelse "";
                if (index_str.len > 0 and !std.mem.startsWith(u8, base_str, "%%"))
                    return self.fail("invalid memory operand: '{s}'", .{op_str});
                const scale_str = sib_it.next() orelse "";
                if (index_str.len == 0 and scale_str.len > 0)
                    return self.fail("invalid memory operand: '{s}'", .{op_str});
                const scale: Memory.Scale = if (scale_str.len > 0)
                    switch (std.fmt.parseInt(u4, scale_str, 10) catch
                        return self.fail("invalid scale: '{s}'", .{op_str})) {
                        1 => .@"1",
                        2 => .@"2",
                        4 => .@"4",
                        8 => .@"8",
                        else => return self.fail("invalid scale: '{s}'", .{op_str}),
                    }
                else
                    .@"1";
                if (sib_it.next()) |_| return self.fail("invalid memory operand: '{s}'", .{op_str});
                op.* = if (std.mem.eql(u8, base_str, "%%dx") and index_str.len == 0) .{ .reg = .dx } else .{ .mem = .{
                    .base = if (base_str.len > 0)
                        .{ .reg = parseRegName(base_str["%%".len..]) orelse
                            return self.fail("invalid base register: '{s}'", .{base_str}) }
                    else
                        .none,
                    .mod = .{ .rm = .{
                        .size = mnem_size.use() orelse return self.fail("unknown size: '{s}'", .{op_str}),
                        .index = if (index_str.len > 0)
                            parseRegName(index_str["%%".len..]) orelse
                                return self.fail("invalid index register: '{s}'", .{op_str})
                        else
                            .none,
                        .scale = scale,
                        .disp = if (std.mem.startsWith(u8, op_str[0..open], "%[") and
                            std.mem.endsWith(u8, op_str[0..open], "]"))
                        disp: {
                            const colon = std.mem.indexOfScalarPos(u8, op_str[0..open], "%[".len, ':');
                            const modifier = if (colon) |colon_pos|
                                op_str[colon_pos + ":".len .. open - "]".len]
                            else
                                "";
                            break :disp switch (args.items[
                                arg_map.get(op_str["%[".len .. colon orelse open - "]".len]) orelse
                                    return self.fail("no matching constraint: '{s}'", .{op_str})
                            ]) {
                                .immediate => |imm| if (std.mem.eql(u8, modifier, "") or
                                    std.mem.eql(u8, modifier, "c"))
                                    std.math.cast(i32, @as(i64, @bitCast(imm))) orelse
                                        return self.fail("invalid displacement: '{s}'", .{op_str})
                                else
                                    return self.fail("invalid modifier: '{s}'", .{modifier}),
                                else => return self.fail("invalid constraint: '{s}'", .{op_str}),
                            };
                        } else if (open > 0)
                            std.fmt.parseInt(i32, op_str[0..open], 0) catch
                                return self.fail("invalid displacement: '{s}'", .{op_str})
                        else
                            0,
                    } },
                } };
            } else if (Label.isValid(.reference, op_str)) {
                const anon = std.ascii.isDigit(op_str[0]);
                const label_gop = try labels.getOrPut(self.gpa, op_str[0..if (anon) 1 else op_str.len]);
                if (!label_gop.found_existing) label_gop.value_ptr.* = .{};
                if (anon and (op_str[1] == 'b' or op_str[1] == 'B') and !label_gop.found_existing)
                    return self.fail("undefined label: '{s}'", .{op_str});
                const pending_relocs = &label_gop.value_ptr.pending_relocs;
                if (if (anon)
                    op_str[1] == 'f' or op_str[1] == 'F'
                else
                    !label_gop.found_existing or pending_relocs.items.len > 0)
                    try pending_relocs.append(self.gpa, @intCast(self.mir_instructions.len));
                op.* = .{ .inst = label_gop.value_ptr.target };
            } else return self.fail("invalid operand: '{s}'", .{op_str});
            ops_len += 1;
        } else if (op_it.next()) |op_str| return self.fail("extra operand: '{s}'", .{op_str});

        // convert from att syntax to intel syntax
        std.mem.reverse(Operand, ops[0..ops_len]);
        if (!mnem_size.used) if (mnem_size.size) |size| {
            comptime var max_mnem_len: usize = 0;
            inline for (@typeInfo(encoder.Instruction.Mnemonic).@"enum".fields) |mnem|
                max_mnem_len = @max(mnem.name.len, max_mnem_len);
            var intel_mnem_buf: [max_mnem_len + 1]u8 = undefined;
            const intel_mnem_str = std.fmt.bufPrint(&intel_mnem_buf, "{s}{c}", .{
                @tagName(mnem_tag),
                @as(u8, switch (size) {
                    .byte => 'b',
                    .word => 'w',
                    .dword => 'd',
                    .qword => 'q',
                    .tbyte => 't',
                    else => unreachable,
                }),
            }) catch unreachable;
            if (std.meta.stringToEnum(encoder.Instruction.Mnemonic, intel_mnem_str)) |intel_mnem_tag| mnem_tag = intel_mnem_tag;
        };
        const mnem_name = @tagName(mnem_tag);
        const mnem_fixed_tag: Mir.Inst.FixedTag = if (prefix == .directive)
            .{ ._, .pseudo }
        else for (std.enums.values(Mir.Inst.Fixes)) |fixes| {
            const fixes_name = @tagName(fixes);
            const space_i = std.mem.indexOfScalar(u8, fixes_name, ' ');
            const fixes_prefix = if (space_i) |i|
                std.meta.stringToEnum(encoder.Instruction.Prefix, fixes_name[0..i]).?
            else
                .none;
            if (fixes_prefix != prefix) continue;
            const pattern = fixes_name[if (space_i) |i| i + " ".len else 0..];
            const wildcard_i = std.mem.indexOfScalar(u8, pattern, '_').?;
            const mnem_prefix = pattern[0..wildcard_i];
            const mnem_suffix = pattern[wildcard_i + "_".len ..];
            if (!std.mem.startsWith(u8, mnem_name, mnem_prefix)) continue;
            if (!std.mem.endsWith(u8, mnem_name, mnem_suffix)) continue;
            break .{ fixes, std.meta.stringToEnum(
                Mir.Inst.Tag,
                mnem_name[mnem_prefix.len .. mnem_name.len - mnem_suffix.len],
            ) orelse continue };
        } else {
            assert(prefix != .none); // no combination of fixes produced a known mnemonic
            return self.fail("invalid prefix for mnemonic: '{s} {s}'", .{
                @tagName(prefix), mnem_name,
            });
        };

        (if (prefix == .directive) switch (mnem_tag) {
            .@".cfi_def_cfa" => if (ops[0] == .reg and ops[1] == .imm and ops[2] == .none)
                self.asmPseudoRegisterImmediate(.pseudo_cfi_def_cfa_ri_s, ops[0].reg, ops[1].imm)
            else
                error.InvalidInstruction,
            .@".cfi_def_cfa_register" => if (ops[0] == .reg and ops[1] == .none)
                self.asmPseudoRegister(.pseudo_cfi_def_cfa_register_r, ops[0].reg)
            else
                error.InvalidInstruction,
            .@".cfi_def_cfa_offset" => if (ops[0] == .imm and ops[1] == .none)
                self.asmPseudoImmediate(.pseudo_cfi_def_cfa_offset_i_s, ops[0].imm)
            else
                error.InvalidInstruction,
            .@".cfi_adjust_cfa_offset" => if (ops[0] == .imm and ops[1] == .none)
                self.asmPseudoImmediate(.pseudo_cfi_adjust_cfa_offset_i_s, ops[0].imm)
            else
                error.InvalidInstruction,
            .@".cfi_offset" => if (ops[0] == .reg and ops[1] == .imm and ops[2] == .none)
                self.asmPseudoRegisterImmediate(.pseudo_cfi_offset_ri_s, ops[0].reg, ops[1].imm)
            else
                error.InvalidInstruction,
            .@".cfi_val_offset" => if (ops[0] == .reg and ops[1] == .imm and ops[2] == .none)
                self.asmPseudoRegisterImmediate(.pseudo_cfi_val_offset_ri_s, ops[0].reg, ops[1].imm)
            else
                error.InvalidInstruction,
            .@".cfi_rel_offset" => if (ops[0] == .reg and ops[1] == .imm and ops[2] == .none)
                self.asmPseudoRegisterImmediate(.pseudo_cfi_rel_offset_ri_s, ops[0].reg, ops[1].imm)
            else
                error.InvalidInstruction,
            .@".cfi_register" => if (ops[0] == .reg and ops[1] == .reg and ops[2] == .none)
                self.asmPseudoRegisterRegister(.pseudo_cfi_register_rr, ops[0].reg, ops[1].reg)
            else
                error.InvalidInstruction,
            .@".cfi_restore" => if (ops[0] == .reg and ops[1] == .none)
                self.asmPseudoRegister(.pseudo_cfi_restore_r, ops[0].reg)
            else
                error.InvalidInstruction,
            .@".cfi_undefined" => if (ops[0] == .reg and ops[1] == .none)
                self.asmPseudoRegister(.pseudo_cfi_undefined_r, ops[0].reg)
            else
                error.InvalidInstruction,
            .@".cfi_same_value" => if (ops[0] == .reg and ops[1] == .none)
                self.asmPseudoRegister(.pseudo_cfi_same_value_r, ops[0].reg)
            else
                error.InvalidInstruction,
            .@".cfi_remember_state" => if (ops[0] == .none)
                self.asmPseudo(.pseudo_cfi_remember_state_none)
            else
                error.InvalidInstruction,
            .@".cfi_restore_state" => if (ops[0] == .none)
                self.asmPseudo(.pseudo_cfi_restore_state_none)
            else
                error.InvalidInstruction,
            .@".cfi_escape" => error.InvalidInstruction,
            else => unreachable,
        } else self.asmOps(mnem_fixed_tag, ops)) catch |err| switch (err) {
            error.InvalidInstruction => return self.fail(
                "invalid instruction: '{s} {s} {s} {s} {s}'",
                .{
                    mnem_str,
                    @tagName(ops[0]),
                    @tagName(ops[1]),
                    @tagName(ops[2]),
                    @tagName(ops[3]),
                },
            ),
            else => |e| return e,
        };
    }

    var label_it = labels.iterator();
    while (label_it.next()) |label| if (label.value_ptr.pending_relocs.items.len > 0)
        return self.fail("undefined label: '{s}'", .{label.key_ptr.*});

    for (outputs, args.items[0..outputs.len]) |output, arg_mcv| {
        const extra_bytes = std.mem.sliceAsBytes(self.air.extra[outputs_extra_i..]);
        const constraint =
            std.mem.sliceTo(std.mem.sliceAsBytes(self.air.extra[outputs_extra_i..]), 0);
        const name = std.mem.sliceTo(extra_bytes[constraint.len + 1 ..], 0);
        // This equation accounts for the fact that even if we have exactly 4 bytes
        // for the string, we still use the next u32 for the null terminator.
        outputs_extra_i += (constraint.len + name.len + (2 + 3)) / 4;

        if (output == .none) continue;
        if (arg_mcv != .register) continue;
        if (constraint.len == 2 and std.ascii.isDigit(constraint[1])) continue;
        try self.store(self.typeOf(output), .{ .air_ref = output }, arg_mcv, .{});
    }

    simple: {
        var buf: [Liveness.bpi - 1]Air.Inst.Ref = @splat(.none);
        var buf_index: usize = 0;
        for (outputs) |output| {
            if (output == .none) continue;

            if (buf_index >= buf.len) break :simple;
            buf[buf_index] = output;
            buf_index += 1;
        }
        if (buf_index + inputs.len > buf.len) break :simple;
        @memcpy(buf[buf_index..][0..inputs.len], inputs);
        return self.finishAir(inst, result, buf);
    }
    var bt = self.liveness.iterateBigTomb(inst);
    for (outputs) |output| if (output != .none) try self.feed(&bt, output);
    for (inputs) |input| try self.feed(&bt, input);
    return self.finishAirResult(inst, result);
}
```
嗯，让我仔细看看这段Zig代码。看起来这是一个处理内联汇编的函数，叫做`airAsm`。首先，我需要理解这段代码的大致结构和功能。

首先，函数接收一个`CodeGen`的指针和一个指令索引`inst`。然后，它从`self.air`中提取一些数据，比如`ty_pl`和`extra`。接着，它处理输出的约束、输入约束，以及可能的clobbers。

代码中有一个`args`数组，用来存储MCValue（可能是指机器码值），然后有一个`arg_map`来映射参数名到索引。接下来，处理输出部分，逐个解析每个输出的约束条件，比如寄存器分配、内存约束等。对于每个输出，会检查约束是否合法，并分配相应的寄存器或内存位置。

在处理输入部分时，同样会解析约束，并根据不同的约束类型（比如寄存器、立即数、内存等）生成相应的机器码值。这里会涉及到寄存器的分配和锁定，确保输入值正确传递到汇编指令中。

然后处理clobbers，也就是汇编指令可能破坏的寄存器或资源。比如`memory`表示内存可能被修改，`cc`表示条件码寄存器可能被修改，需要相应的处理。

接下来处理汇编源代码，逐行解析每一行的指令。这里涉及到标签的处理、指令的解析（比如前缀、助记符、操作数），以及生成相应的机器码指令。操作数可能有不同的形式，比如寄存器、立即数、内存地址等，需要正确解析并生成对应的操作数表示。

最后，处理输出参数的存储，确保结果正确写回到对应的变量或寄存器中，并处理可能的标签未定义错误。

总的来说，这段代码的主要功能是将Zig的内联汇编指令转换为目标机器的机器码，处理寄存器分配、约束条件、操作数解析，并生成相应的机器指令。同时，还要处理可能出现的错误情况，如无效的约束、寄存器不足等。

在思考过程中，我需要确认各个部分的逻辑是否正确，比如寄存器分配是否合理，约束条件的处理是否符合预期，标签解析是否正确，以及错误处理是否全面。此外，还需要考虑代码中的数据结构，如`args`、`arg_map`、`labels`等，如何在这些结构中正确存储和检索信息。

可能会有一些复杂的逻辑，比如处理不同的约束类型（如`r`、`m`、`i`等），以及不同操作数的解析（如内存地址的多种表示形式）。同时，标签的处理需要确保前向引用和后向引用的正确性，避免出现未定义的标签。

此外，代码中有许多错误处理的情况，需要确保在遇到无效输入时能够正确地返回错误信息，并释放已分配的资源，避免内存泄漏。

总之，这段代码是一个复杂的内联汇编处理函数，需要深入理解Zig编译器的内部结构、目标机器的指令集以及汇编语法的细节。
================================================
这段代码是Zig编译器中将内联汇编（`asm`）转换为目标机器码的关键部分，其思维模型可总结如下：

---

### **核心功能**
1. **内联汇编解析与代码生成**  
   将Zig的`asm`语句转换为底层机器指令，处理寄存器分配、操作数约束、标签跳转等，生成最终的机器码（通过`Mir`指令表示）。

---

### **核心流程**
1. **数据提取**  
   - 从中间表示（AIR）中提取汇编指令的元数据（类型、约束、输入/输出参数、clobbers等）。
   - 解析`outputs`和`inputs`的约束条件（如`"r"`、`"m"`），并分配寄存器或内存位置。

2. **参数处理**  
   - **输出参数**：  
     - 根据约束（如`=`、`+`）判断是否只写或读写，分配寄存器或内存地址。  
     - 处理特殊约束（如`{reg}`显式指定寄存器、`数字`引用其他参数）。  
   - **输入参数**：  
     - 将输入值加载到寄存器或内存（如`"i"`要求立即数，`"m"`要求内存地址）。  
   - **参数映射**：  
     - 通过`arg_map`将符号名（如`%[name]`）映射到参数索引，供后续操作数解析使用。

3. **Clobbers处理**  
   - 标记被汇编代码破坏的资源（如`"memory"`、`"cc"`），确保编译器正确处理寄存器状态和内存同步。

4. **汇编源码解析**  
   - 逐行解析汇编源码，处理：  
     - **标签定义与引用**（如`label:`、`jmp label`），解析前向/后向跳转。  
     - **指令解析**：提取助记符（如`mov`）、前缀（如`rep`）、操作数（寄存器、立即数、内存地址）。  
     - **AT&T到Intel语法转换**：调整操作数顺序和助记符后缀（如`movq`→`mov`+`.qword`）。

5. **机器码生成**  
   - 根据解析结果生成`Mir`指令（中间机器码表示），处理：  
     - 寄存器的显式分配与锁定。  
     - 内存操作数的寻址模式（如`[base + index*scale + disp]`）。  
     - 伪指令（如`.cfi_*`用于调试信息）。

6. **结果回写与清理**  
   - 将输出参数的值写回目标位置（如内存或寄存器）。  
   - 释放临时分配的寄存器/内存，处理未解析标签的错误。

---

### **关键数据结构**
- **`MCValue`**：表示机器码值（寄存器、立即数、内存地址等）。  
- **`arg_map`**：哈希表，映射符号名（如`%[var]`）到参数索引。  
- **`labels`**：管理汇编标签，处理跳转目标的前向/后向引用。  
- **`Mir`指令**：最终生成的机器码中间表示。

---

### **错误处理**
- **约束合法性**：检查无效约束（如`"=m"`用于结果）。  
- **寄存器不足**：抛出`ran out of registers`错误。  
- **标签未定义**：检测未解析的标签引用（如`jmp undefined_label`）。  
- **语法错误**：处理无效操作数格式或指令。

---

### **设计特点**
- **灵活性**：支持多种约束类型（寄存器、内存、立即数、复合约束如`"rm"`）。  
- **扩展性**：通过`Mir`指令抽象不同目标平台的机器码生成。  
- **安全性**：严格管理寄存器生命周期（锁定/解锁），避免资源泄漏。  

---

### **思维模型图示**
```
[Zig asm]  
   │  
   ▼  
解析约束、输入/输出、clobbers  
   │  
   ▼  
分配寄存器/内存 → 生成参数映射（arg_map）  
   │  
   ▼  
逐行解析汇编源码 → 处理标签、指令、操作数  
   │  
   ▼  
生成Mir指令（机器码中间表示）  
   │  
   ▼  
回写结果、清理资源 → 完成代码生成  
```

这段代码体现了编译器后端如何将高级内联汇编映射到具体机器指令，同时确保资源管理和语义正确性。