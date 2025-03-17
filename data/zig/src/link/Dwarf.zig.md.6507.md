```zig
pub const WipNav = struct {
    dwarf: *Dwarf,
    pt: Zcu.PerThread,
    unit: Unit.Index,
    entry: Entry.Index,
    any_children: bool,
    func: InternPool.Index,
    func_sym_index: u32,
    func_high_pc: u32,
    blocks: std.ArrayListUnmanaged(struct {
        abbrev_code: u32,
        low_pc_off: u64,
        high_pc: u32,
    }),
    cfi: struct {
        loc: u32,
        cfa: Cfa.RegOff,
    },
    debug_frame: std.ArrayListUnmanaged(u8),
    debug_info: std.ArrayListUnmanaged(u8),
    debug_line: std.ArrayListUnmanaged(u8),
    debug_loclists: std.ArrayListUnmanaged(u8),
    pending_lazy: std.ArrayListUnmanaged(InternPool.Index),

    pub fn deinit(wip_nav: *WipNav) void {
        const gpa = wip_nav.dwarf.gpa;
        if (wip_nav.func != .none) wip_nav.blocks.deinit(gpa);
        wip_nav.debug_frame.deinit(gpa);
        wip_nav.debug_info.deinit(gpa);
        wip_nav.debug_line.deinit(gpa);
        wip_nav.debug_loclists.deinit(gpa);
        wip_nav.pending_lazy.deinit(gpa);
    }

    pub fn genDebugFrame(wip_nav: *WipNav, loc: u32, cfa: Cfa) UpdateError!void {
        assert(wip_nav.func != .none);
        if (wip_nav.dwarf.debug_frame.header.format == .none) return;
        const loc_cfa: Cfa = .{ .advance_loc = loc };
        try loc_cfa.write(wip_nav);
        try cfa.write(wip_nav);
    }

    pub const LocalTag = enum { local_arg, local_var };
    pub fn genLocalDebugInfo(
        wip_nav: *WipNav,
        tag: LocalTag,
        name: []const u8,
        ty: Type,
        loc: Loc,
    ) UpdateError!void {
        assert(wip_nav.func != .none);
        try wip_nav.abbrevCode(switch (tag) {
            inline else => |ct_tag| @field(AbbrevCode, @tagName(ct_tag)),
        });
        try wip_nav.strp(name);
        try wip_nav.refType(ty);
        try wip_nav.infoExprloc(loc);
        wip_nav.any_children = true;
    }

    pub fn genVarArgsDebugInfo(wip_nav: *WipNav) UpdateError!void {
        assert(wip_nav.func != .none);
        try wip_nav.abbrevCode(.is_var_args);
        wip_nav.any_children = true;
    }

    pub fn advancePCAndLine(
        wip_nav: *WipNav,
        delta_line: i33,
        delta_pc: u64,
    ) error{OutOfMemory}!void {
        const dlw = wip_nav.debug_line.writer(wip_nav.dwarf.gpa);

        const header = wip_nav.dwarf.debug_line.header;
        assert(header.maximum_operations_per_instruction == 1);
        const delta_op: u64 = 0;

        const remaining_delta_line: i9 = @intCast(if (delta_line < header.line_base or
            delta_line - header.line_base >= header.line_range)
        remaining: {
            assert(delta_line != 0);
            try dlw.writeByte(DW.LNS.advance_line);
            try sleb128(dlw, delta_line);
            break :remaining 0;
        } else delta_line);

        const op_advance = @divExact(delta_pc, header.minimum_instruction_length) *
            header.maximum_operations_per_instruction + delta_op;
        const max_op_advance: u9 = (std.math.maxInt(u8) - header.opcode_base) / header.line_range;
        const remaining_op_advance: u8 = @intCast(if (op_advance >= 2 * max_op_advance) remaining: {
            try dlw.writeByte(DW.LNS.advance_pc);
            try uleb128(dlw, op_advance);
            break :remaining 0;
        } else if (op_advance >= max_op_advance) remaining: {
            try dlw.writeByte(DW.LNS.const_add_pc);
            break :remaining op_advance - max_op_advance;
        } else op_advance);

        if (remaining_delta_line == 0 and remaining_op_advance == 0)
            try dlw.writeByte(DW.LNS.copy)
        else
            try dlw.writeByte(@intCast((remaining_delta_line - header.line_base) +
                (header.line_range * remaining_op_advance) + header.opcode_base));
    }

    pub fn setColumn(wip_nav: *WipNav, column: u32) error{OutOfMemory}!void {
        const dlw = wip_nav.debug_line.writer(wip_nav.dwarf.gpa);
        try dlw.writeByte(DW.LNS.set_column);
        try uleb128(dlw, column + 1);
    }

    pub fn negateStmt(wip_nav: *WipNav) error{OutOfMemory}!void {
        const dlw = wip_nav.debug_line.writer(wip_nav.dwarf.gpa);
        try dlw.writeByte(DW.LNS.negate_stmt);
    }

    pub fn setPrologueEnd(wip_nav: *WipNav) error{OutOfMemory}!void {
        const dlw = wip_nav.debug_line.writer(wip_nav.dwarf.gpa);
        try dlw.writeByte(DW.LNS.set_prologue_end);
    }

    pub fn setEpilogueBegin(wip_nav: *WipNav) error{OutOfMemory}!void {
        const dlw = wip_nav.debug_line.writer(wip_nav.dwarf.gpa);
        try dlw.writeByte(DW.LNS.set_epilogue_begin);
    }

    pub fn enterBlock(wip_nav: *WipNav, code_off: u64) UpdateError!void {
        const dwarf = wip_nav.dwarf;
        const diw = wip_nav.debug_info.writer(dwarf.gpa);
        const block = try wip_nav.blocks.addOne(dwarf.gpa);

        block.abbrev_code = @intCast(wip_nav.debug_info.items.len);
        try wip_nav.abbrevCode(.block);
        block.low_pc_off = code_off;
        try wip_nav.infoAddrSym(wip_nav.func_sym_index, code_off);
        block.high_pc = @intCast(wip_nav.debug_info.items.len);
        try diw.writeInt(u32, 0, dwarf.endian);
        wip_nav.any_children = false;
    }

    pub fn leaveBlock(wip_nav: *WipNav, code_off: u64) UpdateError!void {
        const block_bytes = comptime uleb128Bytes(@intFromEnum(AbbrevCode.block));
        const block = wip_nav.blocks.pop().?;
        if (wip_nav.any_children)
            try uleb128(wip_nav.debug_info.writer(wip_nav.dwarf.gpa), @intFromEnum(AbbrevCode.null))
        else
            std.leb.writeUnsignedFixed(
                block_bytes,
                wip_nav.debug_info.items[block.abbrev_code..][0..block_bytes],
                try wip_nav.dwarf.refAbbrevCode(.empty_block),
            );
        std.mem.writeInt(u32, wip_nav.debug_info.items[block.high_pc..][0..4], @intCast(code_off - block.low_pc_off), wip_nav.dwarf.endian);
        wip_nav.any_children = true;
    }

    pub fn enterInlineFunc(
        wip_nav: *WipNav,
        func: InternPool.Index,
        code_off: u64,
        line: u32,
        column: u32,
    ) UpdateError!void {
        const dwarf = wip_nav.dwarf;
        const zcu = wip_nav.pt.zcu;
        const diw = wip_nav.debug_info.writer(dwarf.gpa);
        const block = try wip_nav.blocks.addOne(dwarf.gpa);

        block.abbrev_code = @intCast(wip_nav.debug_info.items.len);
        try wip_nav.abbrevCode(.inlined_func);
        try wip_nav.refNav(zcu.funcInfo(func).owner_nav);
        try uleb128(diw, zcu.navSrcLine(zcu.funcInfo(wip_nav.func).owner_nav) + line + 1);
        try uleb128(diw, column + 1);
        block.low_pc_off = code_off;
        try wip_nav.infoAddrSym(wip_nav.func_sym_index, code_off);
        block.high_pc = @intCast(wip_nav.debug_info.items.len);
        try diw.writeInt(u32, 0, dwarf.endian);
        try wip_nav.setInlineFunc(func);
        wip_nav.any_children = false;
    }

    pub fn leaveInlineFunc(wip_nav: *WipNav, func: InternPool.Index, code_off: u64) UpdateError!void {
        const inlined_func_bytes = comptime uleb128Bytes(@intFromEnum(AbbrevCode.inlined_func));
        const block = wip_nav.blocks.pop().?;
        if (wip_nav.any_children)
            try uleb128(wip_nav.debug_info.writer(wip_nav.dwarf.gpa), @intFromEnum(AbbrevCode.null))
        else
            std.leb.writeUnsignedFixed(
                inlined_func_bytes,
                wip_nav.debug_info.items[block.abbrev_code..][0..inlined_func_bytes],
                try wip_nav.dwarf.refAbbrevCode(.empty_inlined_func),
            );
        std.mem.writeInt(u32, wip_nav.debug_info.items[block.high_pc..][0..4], @intCast(code_off - block.low_pc_off), wip_nav.dwarf.endian);
        try wip_nav.setInlineFunc(func);
        wip_nav.any_children = true;
    }

    pub fn setInlineFunc(wip_nav: *WipNav, func: InternPool.Index) UpdateError!void {
        const zcu = wip_nav.pt.zcu;
        const dwarf = wip_nav.dwarf;
        if (wip_nav.func == func) return;

        const new_func_info = zcu.funcInfo(func);
        const new_file = zcu.navFileScopeIndex(new_func_info.owner_nav);
        const new_unit = try dwarf.getUnit(zcu.fileByIndex(new_file).mod);

        const dlw = wip_nav.debug_line.writer(dwarf.gpa);
        if (dwarf.incremental()) {
            const new_nav_gop = try dwarf.navs.getOrPut(dwarf.gpa, new_func_info.owner_nav);
            errdefer _ = if (!new_nav_gop.found_existing) dwarf.navs.pop();
            if (!new_nav_gop.found_existing) new_nav_gop.value_ptr.* = try dwarf.addCommonEntry(new_unit);

            try dlw.writeByte(DW.LNS.extended_op);
            try uleb128(dlw, 1 + dwarf.sectionOffsetBytes());
            try dlw.writeByte(DW.LNE.ZIG_set_decl);
            try dwarf.debug_line.section.getUnit(wip_nav.unit).getEntry(wip_nav.entry).cross_section_relocs.append(dwarf.gpa, .{
                .source_off = @intCast(wip_nav.debug_line.items.len),
                .target_sec = .debug_info,
                .target_unit = new_unit,
                .target_entry = new_nav_gop.value_ptr.toOptional(),
            });
            try dlw.writeByteNTimes(0, dwarf.sectionOffsetBytes());
            return;
        }

        const old_func_info = zcu.funcInfo(wip_nav.func);
        const old_file = zcu.navFileScopeIndex(old_func_info.owner_nav);
        if (old_file != new_file) {
            const mod_info = dwarf.getModInfo(wip_nav.unit);
            try mod_info.dirs.put(dwarf.gpa, new_unit, {});
            const file_gop = try mod_info.files.getOrPut(dwarf.gpa, new_file);

            try dlw.writeByte(DW.LNS.set_file);
            try uleb128(dlw, file_gop.index);
        }

        const old_src_line: i33 = zcu.navSrcLine(old_func_info.owner_nav);
        const new_src_line: i33 = zcu.navSrcLine(new_func_info.owner_nav);
        if (new_src_line != old_src_line) {
            try dlw.writeByte(DW.LNS.advance_line);
            try sleb128(dlw, new_src_line - old_src_line);
        }

        wip_nav.func = func;
    }

    fn externalReloc(wip_nav: *WipNav, sec: *Section, reloc: ExternalReloc) std.mem.Allocator.Error!void {
        try sec.getUnit(wip_nav.unit).getEntry(wip_nav.entry).external_relocs.append(wip_nav.dwarf.gpa, reloc);
    }

    pub fn infoExternalReloc(wip_nav: *WipNav, reloc: ExternalReloc) std.mem.Allocator.Error!void {
        try wip_nav.externalReloc(&wip_nav.dwarf.debug_info.section, reloc);
    }

    fn frameExternalReloc(wip_nav: *WipNav, reloc: ExternalReloc) std.mem.Allocator.Error!void {
        try wip_nav.externalReloc(&wip_nav.dwarf.debug_frame.section, reloc);
    }

    fn abbrevCode(wip_nav: *WipNav, abbrev_code: AbbrevCode) UpdateError!void {
        try uleb128(wip_nav.debug_info.writer(wip_nav.dwarf.gpa), try wip_nav.dwarf.refAbbrevCode(abbrev_code));
    }

    fn sectionOffset(wip_nav: *WipNav, comptime sec: Section.Index, target_sec: Section.Index, target_unit: Unit.Index, target_entry: Entry.Index, target_off: u32) UpdateError!void {
        const dwarf = wip_nav.dwarf;
        const gpa = dwarf.gpa;
        const entry_ptr = @field(dwarf, @tagName(sec)).section.getUnit(wip_nav.unit).getEntry(wip_nav.entry);
        const bytes = &@field(wip_nav, @tagName(sec));
        const source_off: u32 = @intCast(bytes.items.len);
        if (target_sec != sec) {
            try entry_ptr.cross_section_relocs.append(gpa, .{
                .source_off = source_off,
                .target_sec = target_sec,
                .target_unit = target_unit,
                .target_entry = target_entry.toOptional(),
                .target_off = target_off,
            });
        } else if (target_unit != wip_nav.unit) {
            try entry_ptr.cross_unit_relocs.append(gpa, .{
                .source_off = source_off,
                .target_unit = target_unit,
                .target_entry = target_entry.toOptional(),
                .target_off = target_off,
            });
        } else {
            try entry_ptr.cross_entry_relocs.append(gpa, .{
                .source_off = source_off,
                .target_entry = target_entry.toOptional(),
                .target_off = target_off,
            });
        }
        try bytes.appendNTimes(gpa, 0, dwarf.sectionOffsetBytes());
    }

    fn infoSectionOffset(wip_nav: *WipNav, target_sec: Section.Index, target_unit: Unit.Index, target_entry: Entry.Index, target_off: u32) UpdateError!void {
        try wip_nav.sectionOffset(.debug_info, target_sec, target_unit, target_entry, target_off);
    }

    fn strp(wip_nav: *WipNav, str: []const u8) UpdateError!void {
        try wip_nav.infoSectionOffset(.debug_str, StringSection.unit, try wip_nav.dwarf.debug_str.addString(wip_nav.dwarf, str), 0);
    }

    const ExprLocCounter = struct {
        const Stream = std.io.CountingWriter(std.io.NullWriter);
        stream: Stream,
        section_offset_bytes: u32,
        address_size: AddressSize,
        fn init(dwarf: *Dwarf) ExprLocCounter {
            return .{
                .stream = std.io.countingWriter(std.io.null_writer),
                .section_offset_bytes = dwarf.sectionOffsetBytes(),
                .address_size = dwarf.address_size,
            };
        }
        fn writer(counter: *ExprLocCounter) Stream.Writer {
            return counter.stream.writer();
        }
        fn endian(_: ExprLocCounter) std.builtin.Endian {
            return @import("builtin").cpu.arch.endian();
        }
        fn addrSym(counter: *ExprLocCounter, _: u32) error{}!void {
            counter.stream.bytes_written += @intFromEnum(counter.address_size);
        }
        fn infoEntry(counter: *ExprLocCounter, _: Unit.Index, _: Entry.Index) error{}!void {
            counter.stream.bytes_written += counter.section_offset_bytes;
        }
    };

    fn infoExprloc(wip_nav: *WipNav, loc: Loc) UpdateError!void {
        var counter: ExprLocCounter = .init(wip_nav.dwarf);
        try loc.write(&counter);

        const adapter: struct {
            wip_nav: *WipNav,
            fn writer(ctx: @This()) std.ArrayListUnmanaged(u8).Writer {
                return ctx.wip_nav.debug_info.writer(ctx.wip_nav.dwarf.gpa);
            }
            fn endian(ctx: @This()) std.builtin.Endian {
                return ctx.wip_nav.dwarf.endian;
            }
            fn addrSym(ctx: @This(), sym_index: u32) UpdateError!void {
                try ctx.wip_nav.infoAddrSym(sym_index, 0);
            }
            fn infoEntry(ctx: @This(), unit: Unit.Index, entry: Entry.Index) UpdateError!void {
                try ctx.wip_nav.infoSectionOffset(.debug_info, unit, entry, 0);
            }
        } = .{ .wip_nav = wip_nav };
        try uleb128(adapter.writer(), counter.stream.bytes_written);
        try loc.write(adapter);
    }

    fn infoAddrSym(wip_nav: *WipNav, sym_index: u32, sym_off: u64) UpdateError!void {
        try wip_nav.infoExternalReloc(.{
            .source_off = @intCast(wip_nav.debug_info.items.len),
            .target_sym = sym_index,
            .target_off = sym_off,
        });
        try wip_nav.debug_info.appendNTimes(wip_nav.dwarf.gpa, 0, @intFromEnum(wip_nav.dwarf.address_size));
    }

    fn frameExprloc(wip_nav: *WipNav, loc: Loc) UpdateError!void {
        var counter: ExprLocCounter = .init(wip_nav.dwarf);
        try loc.write(&counter);

        const adapter: struct {
            wip_nav: *WipNav,
            fn writer(ctx: @This()) std.ArrayListUnmanaged(u8).Writer {
                return ctx.wip_nav.debug_frame.writer(ctx.wip_nav.dwarf.gpa);
            }
            fn endian(ctx: @This()) std.builtin.Endian {
                return ctx.wip_nav.dwarf.endian;
            }
            fn addrSym(ctx: @This(), sym_index: u32) UpdateError!void {
                try ctx.wip_nav.frameAddrSym(sym_index, 0);
            }
            fn infoEntry(ctx: @This(), unit: Unit.Index, entry: Entry.Index) UpdateError!void {
                try ctx.wip_nav.sectionOffset(.debug_frame, .debug_info, unit, entry, 0);
            }
        } = .{ .wip_nav = wip_nav };
        try uleb128(adapter.writer(), counter.stream.bytes_written);
        try loc.write(adapter);
    }

    fn frameAddrSym(wip_nav: *WipNav, sym_index: u32, sym_off: u64) UpdateError!void {
        try wip_nav.frameExternalReloc(.{
            .source_off = @intCast(wip_nav.debug_frame.items.len),
            .target_sym = sym_index,
            .target_off = sym_off,
        });
        try wip_nav.debug_frame.appendNTimes(wip_nav.dwarf.gpa, 0, @intFromEnum(wip_nav.dwarf.address_size));
    }

    fn getNavEntry(wip_nav: *WipNav, nav_index: InternPool.Nav.Index) UpdateError!struct { Unit.Index, Entry.Index } {
        const zcu = wip_nav.pt.zcu;
        const ip = &zcu.intern_pool;
        const unit = try wip_nav.dwarf.getUnit(zcu.fileByIndex(ip.getNav(nav_index).srcInst(ip).resolveFile(ip)).mod);
        const gop = try wip_nav.dwarf.navs.getOrPut(wip_nav.dwarf.gpa, nav_index);
        if (gop.found_existing) return .{ unit, gop.value_ptr.* };
        const entry = try wip_nav.dwarf.addCommonEntry(unit);
        gop.value_ptr.* = entry;
        return .{ unit, entry };
    }

    fn refNav(wip_nav: *WipNav, nav_index: InternPool.Nav.Index) UpdateError!void {
        const unit, const entry = try wip_nav.getNavEntry(nav_index);
        try wip_nav.infoSectionOffset(.debug_info, unit, entry, 0);
    }

    fn getTypeEntry(wip_nav: *WipNav, ty: Type) UpdateError!struct { Unit.Index, Entry.Index } {
        const zcu = wip_nav.pt.zcu;
        const ip = &zcu.intern_pool;
        const maybe_inst_index = ty.typeDeclInst(zcu);
        const unit = if (maybe_inst_index) |inst_index|
            try wip_nav.dwarf.getUnit(zcu.fileByIndex(inst_index.resolveFile(ip)).mod)
        else
            .main;
        const gop = try wip_nav.dwarf.types.getOrPut(wip_nav.dwarf.gpa, ty.toIntern());
        if (gop.found_existing) return .{ unit, gop.value_ptr.* };
        const entry = try wip_nav.dwarf.addCommonEntry(unit);
        gop.value_ptr.* = entry;
        if (maybe_inst_index == null) try wip_nav.pending_lazy.append(wip_nav.dwarf.gpa, ty.toIntern());
        return .{ unit, entry };
    }

    fn refType(wip_nav: *WipNav, ty: Type) UpdateError!void {
        const unit, const entry = try wip_nav.getTypeEntry(ty);
        try wip_nav.infoSectionOffset(.debug_info, unit, entry, 0);
    }

    fn getValueEntry(wip_nav: *WipNav, value: Value) UpdateError!struct { Unit.Index, Entry.Index } {
        const zcu = wip_nav.pt.zcu;
        const ip = &zcu.intern_pool;
        const ty = value.typeOf(zcu);
        if (std.debug.runtime_safety) assert(ty.comptimeOnly(zcu) and try ty.onePossibleValue(wip_nav.pt) == null);
        if (ty.toIntern() == .type_type) return wip_nav.getTypeEntry(value.toType());
        if (ip.isFunctionType(ty.toIntern()) and !value.isUndef(zcu)) return wip_nav.getNavEntry(zcu.funcInfo(value.toIntern()).owner_nav);
        const gop = try wip_nav.dwarf.values.getOrPut(wip_nav.dwarf.gpa, value.toIntern());
        const unit: Unit.Index = .main;
        if (gop.found_existing) return .{ unit, gop.value_ptr.* };
        const entry = try wip_nav.dwarf.addCommonEntry(unit);
        gop.value_ptr.* = entry;
        try wip_nav.pending_lazy.append(wip_nav.dwarf.gpa, value.toIntern());
        return .{ unit, entry };
    }

    fn refValue(wip_nav: *WipNav, value: Value) UpdateError!void {
        const unit, const entry = try wip_nav.getValueEntry(value);
        try wip_nav.infoSectionOffset(.debug_info, unit, entry, 0);
    }

    fn refForward(wip_nav: *WipNav) std.mem.Allocator.Error!u32 {
        const dwarf = wip_nav.dwarf;
        const cross_entry_relocs = &dwarf.debug_info.section.getUnit(wip_nav.unit).getEntry(wip_nav.entry).cross_entry_relocs;
        const reloc_index: u32 = @intCast(cross_entry_relocs.items.len);
        try cross_entry_relocs.append(dwarf.gpa, .{
            .source_off = @intCast(wip_nav.debug_info.items.len),
            .target_entry = undefined,
            .target_off = undefined,
        });
        try wip_nav.debug_info.appendNTimes(dwarf.gpa, 0, dwarf.sectionOffsetBytes());
        return reloc_index;
    }

    fn finishForward(wip_nav: *WipNav, reloc_index: u32) void {
        const reloc = &wip_nav.dwarf.debug_info.section.getUnit(wip_nav.unit).getEntry(wip_nav.entry).cross_entry_relocs.items[reloc_index];
        reloc.target_entry = wip_nav.entry.toOptional();
        reloc.target_off = @intCast(wip_nav.debug_info.items.len);
    }

    fn blockValue(wip_nav: *WipNav, src_loc: Zcu.LazySrcLoc, val: Value) UpdateError!void {
        const ty = val.typeOf(wip_nav.pt.zcu);
        const diw = wip_nav.debug_info.writer(wip_nav.dwarf.gpa);
        const bytes = if (ty.hasRuntimeBits(wip_nav.pt.zcu)) ty.abiSize(wip_nav.pt.zcu) else 0;
        try uleb128(diw, bytes);
        if (bytes == 0) return;
        const old_len = wip_nav.debug_info.items.len;
        try codegen.generateSymbol(
            wip_nav.dwarf.bin_file,
            wip_nav.pt,
            src_loc,
            val,
            &wip_nav.debug_info,
            .{ .debug_output = .{ .dwarf = wip_nav } },
        );
        assert(old_len + bytes == wip_nav.debug_info.items.len);
    }

    const AbbrevCodeForForm = struct {
        sdata: AbbrevCode,
        udata: AbbrevCode,
        block: AbbrevCode,
    };

    fn bigIntConstValue(
        wip_nav: *WipNav,
        abbrev_code: AbbrevCodeForForm,
        ty: Type,
        big_int: std.math.big.int.Const,
    ) UpdateError!void {
        const zcu = wip_nav.pt.zcu;
        const diw = wip_nav.debug_info.writer(wip_nav.dwarf.gpa);
        const signedness = switch (ty.toIntern()) {
            .comptime_int_type, .comptime_float_type => .signed,
            else => ty.intInfo(zcu).signedness,
        };
        const bits = @max(1, big_int.bitCountTwosCompForSignedness(signedness));
        if (bits <= 64) {
            try wip_nav.abbrevCode(switch (signedness) {
                .signed => abbrev_code.sdata,
                .unsigned => abbrev_code.udata,
            });
            try wip_nav.debug_info.ensureUnusedCapacity(wip_nav.dwarf.gpa, std.math.divCeil(usize, bits, 7) catch unreachable);
            var bit: usize = 0;
            var carry: u1 = 1;
            while (bit < bits) {
                const limb_bits = @typeInfo(std.math.big.Limb).int.bits;
                const limb_index = bit / limb_bits;
                const limb_shift: std.math.Log2Int(std.math.big.Limb) = @intCast(bit % limb_bits);
                const low_abs_part: u7 = @truncate(big_int.limbs[limb_index] >> limb_shift);
                const abs_part = if (limb_shift > limb_bits - 7 and limb_index + 1 < big_int.limbs.len) abs_part: {
                    const high_abs_part: u7 = @truncate(big_int.limbs[limb_index + 1] << -%limb_shift);
                    break :abs_part high_abs_part | low_abs_part;
                } else low_abs_part;
                const twos_comp_part = if (big_int.positive) abs_part else twos_comp_part: {
                    const twos_comp_part, carry = @addWithOverflow(~abs_part, carry);
                    break :twos_comp_part twos_comp_part;
                };
                bit += 7;
                wip_nav.debug_info.appendAssumeCapacity(@as(u8, if (bit < bits) 0x80 else 0x00) | twos_comp_part);
            }
        } else {
            try wip_nav.abbrevCode(abbrev_code.block);
            const bytes = @max(ty.abiSize(zcu), std.math.divCeil(usize, bits, 8) catch unreachable);
            try uleb128(diw, bytes);
            big_int.writeTwosComplement(
                try wip_nav.debug_info.addManyAsSlice(wip_nav.dwarf.gpa, @intCast(bytes)),
                wip_nav.dwarf.endian,
            );
        }
    }

    fn enumConstValue(
        wip_nav: *WipNav,
        loaded_enum: InternPool.LoadedEnumType,
        abbrev_code: AbbrevCodeForForm,
        field_index: usize,
    ) UpdateError!void {
        const zcu = wip_nav.pt.zcu;
        const ip = &zcu.intern_pool;
        var big_int_space: Value.BigIntSpace = undefined;
        try wip_nav.bigIntConstValue(abbrev_code, .fromInterned(loaded_enum.tag_ty), if (loaded_enum.values.len > 0)
            Value.fromInterned(loaded_enum.values.get(ip)[field_index]).toBigInt(&big_int_space, zcu)
        else
            std.math.big.int.Mutable.init(&big_int_space.limbs, field_index).toConst());
    }

    fn declCommon(
        wip_nav: *WipNav,
        abbrev_code: struct {
            decl: AbbrevCode,
            generic_decl: AbbrevCode,
            decl_instance: AbbrevCode,
        },
        nav: *const InternPool.Nav,
        file: Zcu.File.Index,
        decl: *const std.zig.Zir.Inst.Declaration.Unwrapped,
    ) UpdateError!void {
        const zcu = wip_nav.pt.zcu;
        const ip = &zcu.intern_pool;
        const dwarf = wip_nav.dwarf;
        const diw = wip_nav.debug_info.writer(dwarf.gpa);

        const orig_entry = wip_nav.entry;
        defer wip_nav.entry = orig_entry;
        const parent_type, const is_generic_decl = if (nav.analysis) |analysis| parent_info: {
            const parent_type: Type = .fromInterned(zcu.namespacePtr(analysis.namespace).owner_type);
            const decl_gop = try dwarf.decls.getOrPut(dwarf.gpa, analysis.zir_index);
            errdefer _ = if (!decl_gop.found_existing) dwarf.decls.pop();
            const was_generic_decl = decl_gop.found_existing and
                switch (try dwarf.debug_info.declAbbrevCode(wip_nav.unit, decl_gop.value_ptr.*)) {
                    .null,
                    .decl_alias,
                    .decl_empty_enum,
                    .decl_enum,
                    .decl_namespace_struct,
                    .decl_struct,
                    .decl_packed_struct,
                    .decl_union,
                    .decl_var,
                    .decl_const,
                    .decl_const_runtime_bits,
                    .decl_const_comptime_state,
                    .decl_const_runtime_bits_comptime_state,
                    .decl_nullary_func,
                    .decl_func,
                    .decl_nullary_func_generic,
                    .decl_func_generic,
                    => false,
                    .generic_decl_var,
                    .generic_decl_const,
                    .generic_decl_func,
                    => true,
                    else => unreachable,
                };
            if (parent_type.getCaptures(zcu).len == 0) {
                if (was_generic_decl) try dwarf.freeCommonEntry(wip_nav.unit, decl_gop.value_ptr.*);
                decl_gop.value_ptr.* = orig_entry;
                break :parent_info .{ parent_type, false };
            } else {
                if (was_generic_decl)
                    dwarf.debug_info.section.getUnit(wip_nav.unit).getEntry(decl_gop.value_ptr.*).clear()
                else
                    decl_gop.value_ptr.* = try dwarf.addCommonEntry(wip_nav.unit);
                wip_nav.entry = decl_gop.value_ptr.*;
                break :parent_info .{ parent_type, true };
            }
        } else .{ null, false };

        try wip_nav.abbrevCode(if (is_generic_decl) abbrev_code.generic_decl else abbrev_code.decl);
        try wip_nav.refType((if (is_generic_decl) null else parent_type) orelse
            .fromInterned(zcu.fileRootType(file)));
        assert(wip_nav.debug_info.items.len == DebugInfo.declEntryLineOff(dwarf));
        try diw.writeInt(u32, decl.src_line + 1, dwarf.endian);
        try uleb128(diw, decl.src_column + 1);
        try diw.writeByte(if (decl.is_pub) DW.ACCESS.public else DW.ACCESS.private);
        try wip_nav.strp(nav.name.toSlice(ip));

        if (!is_generic_decl) return;
        const generic_decl_entry = wip_nav.entry;
        try dwarf.debug_info.section.replaceEntry(wip_nav.unit, generic_decl_entry, dwarf, wip_nav.debug_info.items);
        wip_nav.debug_info.clearRetainingCapacity();
        wip_nav.entry = orig_entry;
        try wip_nav.abbrevCode(abbrev_code.decl_instance);
        try wip_nav.refType(parent_type.?);
        try wip_nav.infoSectionOffset(.debug_info, wip_nav.unit, generic_decl_entry, 0);
    }

    fn updateLazy(wip_nav: *WipNav, src_loc: Zcu.LazySrcLoc) UpdateError!void {
        const ip = &wip_nav.pt.zcu.intern_pool;
        while (wip_nav.pending_lazy.pop()) |val| switch (ip.typeOf(val)) {
            .type_type => try wip_nav.dwarf.updateLazyType(wip_nav.pt, src_loc, val, &wip_nav.pending_lazy),
            else => try wip_nav.dwarf.updateLazyValue(wip_nav.pt, src_loc, val, &wip_nav.pending_lazy),
        };
    }
}
```
嗯，我需要仔细看看这个Zig代码。首先，这个文件定义了一个名为WipNav的结构体，属于pub const，也就是说它是公开的。看起来这个结构体是用来处理DWARF调试信息的生成。DWARF是调试信息的一种格式，对吧？

结构体里面有很多字段，比如dwarf、pt、unit、entry等等。dwarf可能是指向DWARF相关数据的指针，pt可能是PerThread的缩写，unit可能表示编译单元，entry可能是某个索引。还有blocks数组，里面存储了一些结构，比如abbrev_code、low_pc_off、high_pc，这些可能和代码块的范围有关。

接着看deinit函数，它释放了多个ArrayListUnmanaged的资源，比如debug_frame、debug_info等，这些都是DWARF的不同节（section），比如.debug_frame、.debug_info等。这说明WipNav负责管理这些调试信息的生成和释放。

genDebugFrame函数似乎生成调试帧信息，处理CFA（Canonical Frame Address）相关的操作。这里调用了Cfa的write方法，可能将CFA信息写入到对应的节中。

LocalTag枚举有两个成员，local_arg和local_var，genLocalDebugInfo函数根据不同的标签生成局部变量或参数的调试信息。这里用到了abbrevCode方法，可能是写入DWARF的缩写代码，然后写入名称、类型和位置信息。

advancePCAndLine函数处理程序计数器和行号的推进。这里涉及到DW.LNS的各种操作码，比如advance_line、advance_pc、copy等，应该是调整调试行号信息。这部分代码有点复杂，需要处理不同的增量情况，比如行号的变化超过一定范围时需要拆分操作。

enterBlock和leaveBlock函数用于进入和离开代码块，更新blocks数组，并处理相关的调试信息。这里涉及到低地址和高地址的记录，可能用于确定代码块的范围。

enterInlineFunc和leaveInlineFunc处理内联函数的调试信息，记录函数的入口和出口，设置相关的行号和符号信息。setInlineFunc函数用于切换当前函数上下文，处理不同函数之间的调试信息关联。

还有一些辅助函数，比如refType、refValue、refNav等，用于生成类型、值、导航的引用。这些引用可能指向DWARF中的其他条目，确保调试信息的正确链接。

最后，updateLazy函数处理延迟生成的调试信息，可能在需要时动态更新某些条目。

整个结构体看起来是围绕生成和维护DWARF调试信息的各个部分展开的，每个方法负责不同的调试信息部分，比如行号、变量、函数、代码块等。通过维护不同的ArrayList来存储生成的调试数据，并在适当的时候进行资源的释放和管理。
================================================
该代码实现了一个用于生成DWARF调试信息的结构体`WipNav`，其核心功能是为Zig编译器提供调试信息生成的工具。以下是其思维模型的总结：

### 核心结构
1. **WipNav结构体**  
   包含多个字段，用于管理调试信息生成的上下文：
   - **调试节数据**：如`debug_frame`、`debug_info`、`debug_line`等，对应DWARF的不同节（section）。
   - **符号与代码块信息**：如`blocks`数组记录代码块的地址范围，`func`和`func_sym_index`关联当前函数的符号信息。
   - **资源管理**：通过`gpa`（通用分配器）管理内存，`deinit`方法负责释放所有分配的调试数据。

---

### 主要功能模块
1. **调试帧（Debug Frame）**  
   - `genDebugFrame`：生成`.debug_frame`节的内容，写入CFA（Canonical Frame Address）和位置信息。
   - 通过`Cfa`结构体描述栈帧布局，结合`advance_loc`和`cfa`操作更新调试帧。

2. **局部变量与参数**  
   - `genLocalDebugInfo`：根据`LocalTag`（`local_arg`或`local_var`）生成局部变量或参数的调试信息，包括名称、类型和位置。
   - `genVarArgsDebugInfo`：处理可变参数（如C语言的`va_list`）的调试信息。

3. **行号与程序计数器**  
   - `advancePCAndLine`：调整程序计数器（PC）和行号，处理不同增量的行号变化（如`DW.LNS.advance_line`和`DW.LNS.advance_pc`）。
   - `setColumn`、`negateStmt`等方法：设置列号、切换语句标记（如函数序言/尾声）。

4. **代码块与内联函数**  
   - `enterBlock`/`leaveBlock`：标记代码块的开始和结束，记录`low_pc`和`high_pc`地址。
   - `enterInlineFunc`/`leaveInlineFunc`：处理内联函数的调试信息，关联内联函数的符号、行号、列号。

5. **符号引用与类型系统**  
   - `refType`、`refValue`、`refNav`：生成对类型、值、符号的引用，通过`sectionOffset`写入跨节或跨单元的偏移量。
   - `getTypeEntry`、`getValueEntry`：管理类型和值的调试条目，支持延迟生成（Lazy Evaluation）。

---

### 辅助机制
1. **缩写代码（Abbrev Code）**  
   - `abbrevCode`方法写入DWARF的缩写代码，用于压缩调试信息格式（如`DW_TAG`和`DW_AT`的编码）。

2. **交叉引用与重定位**  
   - `externalReloc`和`sectionOffset`：处理不同调试节之间的符号重定位，确保地址和引用的正确链接。

3. **延迟更新（Lazy Update）**  
   - `pending_lazy`列表存储未解析的类型或值，`updateLazy`方法在需要时动态生成其调试信息。

---

### 设计目标
- **模块化**：每个方法专注于单一功能（如行号、符号、代码块），便于扩展和维护。
- **高效性**：通过`ArrayListUnmanaged`直接操作内存，减少动态分配开销。
- **兼容性**：支持增量编译（`dwarf.incremental()`），适配不同DWARF版本（如`.debug_frame`的格式）。

### 总结
`WipNav`是Zig编译器调试信息生成的核心工具，通过分层管理调试节的生成、符号引用、代码块范围和延迟解析，实现了高效且灵活的DWARF调试信息输出。其设计紧密结合Zig的类型系统和编译流程，确保调试信息与代码逻辑严格同步。