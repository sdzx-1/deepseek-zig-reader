```zig
pub fn create(gpa: Allocator, arena: Allocator, options: CreateOptions) !*Compilation {
    const output_mode = options.config.output_mode;
    const is_dyn_lib = switch (output_mode) {
        .Obj, .Exe => false,
        .Lib => options.config.link_mode == .dynamic,
    };
    const is_exe_or_dyn_lib = switch (output_mode) {
        .Obj => false,
        .Lib => is_dyn_lib,
        .Exe => true,
    };

    if (options.linker_export_table and options.linker_import_table) {
        return error.ExportTableAndImportTableConflict;
    }

    const have_zcu = options.config.have_zcu;

    const comp: *Compilation = comp: {
        // We put the `Compilation` itself in the arena. Freeing the arena will free the module.
        // It's initialized later after we prepare the initialization options.
        const root_name = try arena.dupeZ(u8, options.root_name);

        const use_llvm = options.config.use_llvm;

        // The "any" values provided by resolved config only account for
        // explicitly-provided settings. We now make them additionally account
        // for default setting resolution.
        const any_unwind_tables = options.config.any_unwind_tables or options.root_mod.unwind_tables != .none;
        const any_non_single_threaded = options.config.any_non_single_threaded or !options.root_mod.single_threaded;
        const any_sanitize_thread = options.config.any_sanitize_thread or options.root_mod.sanitize_thread;
        const any_sanitize_c = options.config.any_sanitize_c or options.root_mod.sanitize_c;
        const any_fuzz = options.config.any_fuzz or options.root_mod.fuzz;

        const link_eh_frame_hdr = options.link_eh_frame_hdr or any_unwind_tables;
        const build_id = options.build_id orelse .none;

        const link_libc = options.config.link_libc;

        const libc_dirs = try std.zig.LibCDirs.detect(
            arena,
            options.zig_lib_directory.path.?,
            options.root_mod.resolved_target.result,
            options.root_mod.resolved_target.is_native_abi,
            link_libc,
            options.libc_installation,
        );

        const sysroot = options.sysroot orelse libc_dirs.sysroot;

        const compiler_rt_strat: RtStrat = s: {
            if (options.skip_linker_dependencies) break :s .none;
            const want = options.want_compiler_rt orelse is_exe_or_dyn_lib;
            if (!want) break :s .none;
            if (have_zcu and output_mode == .Obj) break :s .zcu;
            if (is_exe_or_dyn_lib) break :s .lib;
            break :s .obj;
        };

        if (compiler_rt_strat == .zcu) {
            // For objects, this mechanism relies on essentially `_ = @import("compiler-rt");`
            // injected into the object.
            const compiler_rt_mod = try Package.Module.create(arena, .{
                .global_cache_directory = options.global_cache_directory,
                .paths = .{
                    .root = .{
                        .root_dir = options.zig_lib_directory,
                    },
                    .root_src_path = "compiler_rt.zig",
                },
                .fully_qualified_name = "compiler_rt",
                .cc_argv = &.{},
                .inherited = .{
                    .stack_check = false,
                    .stack_protector = 0,
                    .no_builtin = true,
                },
                .global = options.config,
                .parent = options.root_mod,
                .builtin_mod = options.root_mod.getBuiltinDependency(),
                .builtin_modules = null, // `builtin_mod` is set
            });
            try options.root_mod.deps.putNoClobber(arena, "compiler_rt", compiler_rt_mod);
        }

        // unlike compiler_rt, we always want to go through the `_ = @import("ubsan-rt")`
        // approach, since the ubsan runtime uses quite a lot of the standard library
        // and this reduces unnecessary bloat.
        const ubsan_rt_strat: RtStrat = s: {
            const is_spirv = options.root_mod.resolved_target.result.cpu.arch.isSpirV();
            const want_ubsan_rt = options.want_ubsan_rt orelse (!is_spirv and any_sanitize_c and is_exe_or_dyn_lib);
            if (!want_ubsan_rt) break :s .none;
            if (options.skip_linker_dependencies) break :s .none;
            if (have_zcu) break :s .zcu;
            if (is_exe_or_dyn_lib) break :s .lib;
            break :s .obj;
        };

        if (ubsan_rt_strat == .zcu) {
            const ubsan_rt_mod = try Package.Module.create(arena, .{
                .global_cache_directory = options.global_cache_directory,
                .paths = .{
                    .root = .{
                        .root_dir = options.zig_lib_directory,
                    },
                    .root_src_path = "ubsan_rt.zig",
                },
                .fully_qualified_name = "ubsan_rt",
                .cc_argv = &.{},
                .inherited = .{},
                .global = options.config,
                .parent = options.root_mod,
                .builtin_mod = options.root_mod.getBuiltinDependency(),
                .builtin_modules = null, // `builtin_mod` is set
            });
            try options.root_mod.deps.putNoClobber(arena, "ubsan_rt", ubsan_rt_mod);
        }

        if (options.verbose_llvm_cpu_features) {
            if (options.root_mod.resolved_target.llvm_cpu_features) |cf| print: {
                const target = options.root_mod.resolved_target.result;
                std.debug.lockStdErr();
                defer std.debug.unlockStdErr();
                const stderr = std.io.getStdErr().writer();
                nosuspend {
                    stderr.print("compilation: {s}\n", .{options.root_name}) catch break :print;
                    stderr.print("  target: {s}\n", .{try target.zigTriple(arena)}) catch break :print;
                    stderr.print("  cpu: {s}\n", .{target.cpu.model.name}) catch break :print;
                    stderr.print("  features: {s}\n", .{cf}) catch {};
                }
            }
        }

        const error_limit = options.error_limit orelse (std.math.maxInt(u16) - 1);

        // We put everything into the cache hash that *cannot be modified
        // during an incremental update*. For example, one cannot change the
        // target between updates, but one can change source files, so the
        // target goes into the cache hash, but source files do not. This is so
        // that we can find the same binary and incrementally update it even if
        // there are modified source files. We do this even if outputting to
        // the current directory because we need somewhere to store incremental
        // compilation metadata.
        const cache = try arena.create(Cache);
        cache.* = .{
            .gpa = gpa,
            .manifest_dir = try options.local_cache_directory.handle.makeOpenPath("h", .{}),
        };
        // These correspond to std.zig.Server.Message.PathPrefix.
        cache.addPrefix(.{ .path = null, .handle = std.fs.cwd() });
        cache.addPrefix(options.zig_lib_directory);
        cache.addPrefix(options.local_cache_directory);
        cache.addPrefix(options.global_cache_directory);
        errdefer cache.manifest_dir.close();

        // This is shared hasher state common to zig source and all C source files.
        cache.hash.addBytes(build_options.version);
        cache.hash.add(builtin.zig_backend);
        cache.hash.add(options.config.pie);
        cache.hash.add(options.config.lto);
        cache.hash.add(options.config.link_mode);
        cache.hash.add(options.config.any_unwind_tables);
        cache.hash.add(options.function_sections);
        cache.hash.add(options.data_sections);
        cache.hash.add(link_libc);
        cache.hash.add(options.config.link_libcpp);
        cache.hash.add(options.config.link_libunwind);
        cache.hash.add(output_mode);
        cache_helpers.addDebugFormat(&cache.hash, options.config.debug_format);
        cache_helpers.addOptionalEmitLoc(&cache.hash, options.emit_bin);
        cache_helpers.addOptionalEmitLoc(&cache.hash, options.emit_implib);
        cache_helpers.addOptionalEmitLoc(&cache.hash, options.emit_docs);
        cache.hash.addBytes(options.root_name);
        cache.hash.add(options.config.wasi_exec_model);
        cache.hash.add(options.config.san_cov_trace_pc_guard);
        cache.hash.add(options.debug_compiler_runtime_libs);
        // TODO audit this and make sure everything is in it

        const main_mod = options.main_mod orelse options.root_mod;
        const comp = try arena.create(Compilation);
        const opt_zcu: ?*Zcu = if (have_zcu) blk: {
            // Pre-open the directory handles for cached ZIR code so that it does not need
            // to redundantly happen for each AstGen operation.
            const zir_sub_dir = "z";

            var local_zir_dir = try options.local_cache_directory.handle.makeOpenPath(zir_sub_dir, .{});
            errdefer local_zir_dir.close();
            const local_zir_cache: Directory = .{
                .handle = local_zir_dir,
                .path = try options.local_cache_directory.join(arena, &[_][]const u8{zir_sub_dir}),
            };
            var global_zir_dir = try options.global_cache_directory.handle.makeOpenPath(zir_sub_dir, .{});
            errdefer global_zir_dir.close();
            const global_zir_cache: Directory = .{
                .handle = global_zir_dir,
                .path = try options.global_cache_directory.join(arena, &[_][]const u8{zir_sub_dir}),
            };

            const std_mod = options.std_mod orelse try Package.Module.create(arena, .{
                .global_cache_directory = options.global_cache_directory,
                .paths = .{
                    .root = .{
                        .root_dir = options.zig_lib_directory,
                        .sub_path = "std",
                    },
                    .root_src_path = "std.zig",
                },
                .fully_qualified_name = "std",
                .cc_argv = &.{},
                .inherited = .{},
                .global = options.config,
                .parent = options.root_mod,
                .builtin_mod = options.root_mod.getBuiltinDependency(),
                .builtin_modules = null, // `builtin_mod` is set
            });

            const zcu = try arena.create(Zcu);
            zcu.* = .{
                .gpa = gpa,
                .comp = comp,
                .main_mod = main_mod,
                .root_mod = options.root_mod,
                .std_mod = std_mod,
                .global_zir_cache = global_zir_cache,
                .local_zir_cache = local_zir_cache,
                .error_limit = error_limit,
                .llvm_object = null,
            };
            try zcu.init(options.thread_pool.getIdCount());
            break :blk zcu;
        } else blk: {
            if (options.emit_h != null) return error.NoZigModuleForCHeader;
            break :blk null;
        };
        errdefer if (opt_zcu) |zcu| zcu.deinit();

        var windows_libs = try std.StringArrayHashMapUnmanaged(void).init(gpa, options.windows_lib_names, &.{});
        errdefer windows_libs.deinit(gpa);

        comp.* = .{
            .gpa = gpa,
            .arena = arena,
            .zcu = opt_zcu,
            .cache_use = undefined, // populated below
            .bin_file = null, // populated below
            .implib_emit = null, // handled below
            .docs_emit = null, // handled below
            .root_mod = options.root_mod,
            .config = options.config,
            .zig_lib_directory = options.zig_lib_directory,
            .local_cache_directory = options.local_cache_directory,
            .global_cache_directory = options.global_cache_directory,
            .emit_asm = options.emit_asm,
            .emit_llvm_ir = options.emit_llvm_ir,
            .emit_llvm_bc = options.emit_llvm_bc,
            .work_queues = .{std.fifo.LinearFifo(Job, .Dynamic).init(gpa)} ** @typeInfo(std.meta.FieldType(Compilation, .work_queues)).array.len,
            .c_object_work_queue = std.fifo.LinearFifo(*CObject, .Dynamic).init(gpa),
            .win32_resource_work_queue = if (dev.env.supports(.win32_resource)) std.fifo.LinearFifo(*Win32Resource, .Dynamic).init(gpa) else .{},
            .astgen_work_queue = std.fifo.LinearFifo(Zcu.File.Index, .Dynamic).init(gpa),
            .c_source_files = options.c_source_files,
            .rc_source_files = options.rc_source_files,
            .cache_parent = cache,
            .self_exe_path = options.self_exe_path,
            .libc_include_dir_list = libc_dirs.libc_include_dir_list,
            .libc_framework_dir_list = libc_dirs.libc_framework_dir_list,
            .rc_includes = options.rc_includes,
            .mingw_unicode_entry_point = options.mingw_unicode_entry_point,
            .thread_pool = options.thread_pool,
            .clang_passthrough_mode = options.clang_passthrough_mode,
            .clang_preprocessor_mode = options.clang_preprocessor_mode,
            .verbose_cc = options.verbose_cc,
            .verbose_air = options.verbose_air,
            .verbose_intern_pool = options.verbose_intern_pool,
            .verbose_generic_instances = options.verbose_generic_instances,
            .verbose_llvm_ir = options.verbose_llvm_ir,
            .verbose_llvm_bc = options.verbose_llvm_bc,
            .verbose_cimport = options.verbose_cimport,
            .verbose_llvm_cpu_features = options.verbose_llvm_cpu_features,
            .verbose_link = options.verbose_link,
            .disable_c_depfile = options.disable_c_depfile,
            .reference_trace = options.reference_trace,
            .time_report = options.time_report,
            .stack_report = options.stack_report,
            .test_filters = options.test_filters,
            .test_name_prefix = options.test_name_prefix,
            .debug_compiler_runtime_libs = options.debug_compiler_runtime_libs,
            .debug_compile_errors = options.debug_compile_errors,
            .incremental = options.incremental,
            .libcxx_abi_version = options.libcxx_abi_version,
            .root_name = root_name,
            .sysroot = sysroot,
            .windows_libs = windows_libs,
            .version = options.version,
            .libc_installation = libc_dirs.libc_installation,
            .compiler_rt_strat = compiler_rt_strat,
            .ubsan_rt_strat = ubsan_rt_strat,
            .link_inputs = options.link_inputs,
            .framework_dirs = options.framework_dirs,
            .llvm_opt_bisect_limit = options.llvm_opt_bisect_limit,
            .skip_linker_dependencies = options.skip_linker_dependencies,
            .queued_jobs = .{
                .update_builtin_zig = have_zcu,
            },
            .function_sections = options.function_sections,
            .data_sections = options.data_sections,
            .native_system_include_paths = options.native_system_include_paths,
            .wasi_emulated_libs = options.wasi_emulated_libs,
            .force_undefined_symbols = options.force_undefined_symbols,
            .link_eh_frame_hdr = link_eh_frame_hdr,
            .global_cc_argv = options.global_cc_argv,
            .file_system_inputs = options.file_system_inputs,
            .parent_whole_cache = options.parent_whole_cache,
            .link_diags = .init(gpa),
            .remaining_prelink_tasks = 0,
        };

        // Prevent some footguns by making the "any" fields of config reflect
        // the default Module settings.
        comp.config.any_unwind_tables = any_unwind_tables;
        comp.config.any_non_single_threaded = any_non_single_threaded;
        comp.config.any_sanitize_thread = any_sanitize_thread;
        comp.config.any_sanitize_c = any_sanitize_c;
        comp.config.any_fuzz = any_fuzz;

        const lf_open_opts: link.File.OpenOptions = .{
            .linker_script = options.linker_script,
            .z_nodelete = options.linker_z_nodelete,
            .z_notext = options.linker_z_notext,
            .z_defs = options.linker_z_defs,
            .z_origin = options.linker_z_origin,
            .z_nocopyreloc = options.linker_z_nocopyreloc,
            .z_now = options.linker_z_now,
            .z_relro = options.linker_z_relro,
            .z_common_page_size = options.linker_z_common_page_size,
            .z_max_page_size = options.linker_z_max_page_size,
            .darwin_sdk_layout = libc_dirs.darwin_sdk_layout,
            .frameworks = options.frameworks,
            .lib_directories = options.lib_directories,
            .framework_dirs = options.framework_dirs,
            .rpath_list = options.rpath_list,
            .symbol_wrap_set = options.symbol_wrap_set,
            .repro = options.linker_repro orelse (options.root_mod.optimize_mode != .Debug),
            .allow_shlib_undefined = options.linker_allow_shlib_undefined,
            .bind_global_refs_locally = options.linker_bind_global_refs_locally orelse false,
            .compress_debug_sections = options.linker_compress_debug_sections orelse .none,
            .module_definition_file = options.linker_module_definition_file,
            .sort_section = options.linker_sort_section,
            .import_symbols = options.linker_import_symbols,
            .import_table = options.linker_import_table,
            .export_table = options.linker_export_table,
            .initial_memory = options.linker_initial_memory,
            .max_memory = options.linker_max_memory,
            .global_base = options.linker_global_base,
            .export_symbol_names = options.linker_export_symbol_names,
            .print_gc_sections = options.linker_print_gc_sections,
            .print_icf_sections = options.linker_print_icf_sections,
            .print_map = options.linker_print_map,
            .tsaware = options.linker_tsaware,
            .nxcompat = options.linker_nxcompat,
            .dynamicbase = options.linker_dynamicbase,
            .major_subsystem_version = options.major_subsystem_version,
            .minor_subsystem_version = options.minor_subsystem_version,
            .entry = options.entry,
            .stack_size = options.stack_size,
            .image_base = options.image_base,
            .version_script = options.version_script,
            .allow_undefined_version = options.linker_allow_undefined_version,
            .enable_new_dtags = options.linker_enable_new_dtags,
            .gc_sections = options.linker_gc_sections,
            .emit_relocs = options.link_emit_relocs,
            .soname = options.soname,
            .compatibility_version = options.compatibility_version,
            .build_id = build_id,
            .disable_lld_caching = options.disable_lld_caching or options.cache_mode == .whole,
            .subsystem = options.subsystem,
            .hash_style = options.hash_style,
            .enable_link_snapshots = options.enable_link_snapshots,
            .install_name = options.install_name,
            .entitlements = options.entitlements,
            .pagezero_size = options.pagezero_size,
            .headerpad_size = options.headerpad_size,
            .headerpad_max_install_names = options.headerpad_max_install_names,
            .dead_strip_dylibs = options.dead_strip_dylibs,
            .force_load_objc = options.force_load_objc,
            .discard_local_symbols = options.discard_local_symbols,
            .pdb_source_path = options.pdb_source_path,
            .pdb_out_path = options.pdb_out_path,
            .entry_addr = null, // CLI does not expose this option (yet?)
            .object_host_name = "env",
        };

        switch (options.cache_mode) {
            .incremental => {
                // Options that are specific to zig source files, that cannot be
                // modified between incremental updates.
                var hash = cache.hash;

                // Synchronize with other matching comments: ZigOnlyHashStuff
                hash.add(use_llvm);
                hash.add(options.config.use_lib_llvm);
                hash.add(options.config.dll_export_fns);
                hash.add(options.config.is_test);
                hash.addListOfBytes(options.test_filters);
                hash.addOptionalBytes(options.test_name_prefix);
                hash.add(options.skip_linker_dependencies);
                hash.add(options.emit_h != null);
                hash.add(error_limit);

                // Here we put the root source file path name, but *not* with addFile.
                // We want the hash to be the same regardless of the contents of the
                // source file, because incremental compilation will handle it, but we
                // do want to namespace different source file names because they are
                // likely different compilations and therefore this would be likely to
                // cause cache hits.
                try addModuleTableToCacheHash(gpa, arena, &hash, options.root_mod, main_mod, .path_bytes);

                // In the case of incremental cache mode, this `artifact_directory`
                // is computed based on a hash of non-linker inputs, and it is where all
                // build artifacts are stored (even while in-progress).
                comp.digest = hash.peekBin();
                const digest = hash.final();

                const artifact_sub_dir = "o" ++ std.fs.path.sep_str ++ digest;
                var artifact_dir = try options.local_cache_directory.handle.makeOpenPath(artifact_sub_dir, .{});
                errdefer artifact_dir.close();
                const artifact_directory: Directory = .{
                    .handle = artifact_dir,
                    .path = try options.local_cache_directory.join(arena, &[_][]const u8{artifact_sub_dir}),
                };

                const incremental = try arena.create(CacheUse.Incremental);
                incremental.* = .{
                    .artifact_directory = artifact_directory,
                };
                comp.cache_use = .{ .incremental = incremental };

                if (options.emit_bin) |emit_bin| {
                    const emit: Path = .{
                        .root_dir = emit_bin.directory orelse artifact_directory,
                        .sub_path = emit_bin.basename,
                    };
                    comp.bin_file = try link.File.open(arena, comp, emit, lf_open_opts);
                }

                if (options.emit_implib) |emit_implib| {
                    comp.implib_emit = .{
                        .root_dir = emit_implib.directory orelse artifact_directory,
                        .sub_path = emit_implib.basename,
                    };
                }

                if (options.emit_docs) |emit_docs| {
                    comp.docs_emit = .{
                        .root_dir = emit_docs.directory orelse artifact_directory,
                        .sub_path = emit_docs.basename,
                    };
                }
            },
            .whole => {
                // For whole cache mode, we don't know where to put outputs from
                // the linker until the final cache hash, which is available after
                // the compilation is complete.
                //
                // Therefore, bin_file is left null until the beginning of update(),
                // where it may find a cache hit, or use a temporary directory to
                // hold output artifacts.
                const whole = try arena.create(CacheUse.Whole);
                whole.* = .{
                    // This is kept here so that link.File.open can be called later.
                    .lf_open_opts = lf_open_opts,
                    // This is so that when doing `CacheMode.whole`, the mechanism in update()
                    // can use it for communicating the result directory via `bin_file.emit`.
                    // This is used to distinguish between -fno-emit-bin and -femit-bin
                    // for `CacheMode.whole`.
                    // This memory will be overwritten with the real digest in update() but
                    // the basename will be preserved.
                    .bin_sub_path = try prepareWholeEmitSubPath(arena, options.emit_bin),
                    .implib_sub_path = try prepareWholeEmitSubPath(arena, options.emit_implib),
                    .docs_sub_path = try prepareWholeEmitSubPath(arena, options.emit_docs),
                    .tmp_artifact_directory = null,
                    .lock = null,
                };
                comp.cache_use = .{ .whole = whole };
            },
        }

        // Handle the case of e.g. -fno-emit-bin -femit-llvm-ir.
        if (options.emit_bin == null and (comp.verbose_llvm_ir != null or
            comp.verbose_llvm_bc != null or
            (use_llvm and comp.emit_asm != null) or
            comp.emit_llvm_ir != null or
            comp.emit_llvm_bc != null))
        {
            if (opt_zcu) |zcu| zcu.llvm_object = try LlvmObject.create(arena, comp);
        }

        break :comp comp;
    };
    errdefer comp.destroy();

    const target = comp.root_mod.resolved_target.result;

    const capable_of_building_compiler_rt = canBuildLibCompilerRt(target, comp.config.use_llvm);
    const capable_of_building_zig_libc = canBuildZigLibC(target, comp.config.use_llvm);

    // Add a `CObject` for each `c_source_files`.
    try comp.c_object_table.ensureTotalCapacity(gpa, options.c_source_files.len);
    for (options.c_source_files) |c_source_file| {
        const c_object = try gpa.create(CObject);
        errdefer gpa.destroy(c_object);

        c_object.* = .{
            .status = .{ .new = {} },
            .src = c_source_file,
        };
        comp.c_object_table.putAssumeCapacityNoClobber(c_object, {});
    }
    comp.remaining_prelink_tasks += @intCast(comp.c_object_table.count());

    // Add a `Win32Resource` for each `rc_source_files` and one for `manifest_file`.
    const win32_resource_count =
        options.rc_source_files.len + @intFromBool(options.manifest_file != null);
    if (win32_resource_count > 0) {
        dev.check(.win32_resource);
        try comp.win32_resource_table.ensureTotalCapacity(gpa, win32_resource_count);
        // Add this after adding logic to updateWin32Resource to pass the
        // result into link.loadInput. loadInput integration is not implemented
        // for Windows linking logic yet.
        //comp.remaining_prelink_tasks += @intCast(win32_resource_count);
        for (options.rc_source_files) |rc_source_file| {
            const win32_resource = try gpa.create(Win32Resource);
            errdefer gpa.destroy(win32_resource);

            win32_resource.* = .{
                .status = .{ .new = {} },
                .src = .{ .rc = rc_source_file },
            };
            comp.win32_resource_table.putAssumeCapacityNoClobber(win32_resource, {});
        }

        if (options.manifest_file) |manifest_path| {
            const win32_resource = try gpa.create(Win32Resource);
            errdefer gpa.destroy(win32_resource);

            win32_resource.* = .{
                .status = .{ .new = {} },
                .src = .{ .manifest = manifest_path },
            };
            comp.win32_resource_table.putAssumeCapacityNoClobber(win32_resource, {});
        }
    }

    const have_bin_emit = switch (comp.cache_use) {
        .whole => |whole| whole.bin_sub_path != null,
        .incremental => comp.bin_file != null,
    };

    if (have_bin_emit and target.ofmt != .c) {
        if (!comp.skip_linker_dependencies) {
            // If we need to build libc for the target, add work items for it.
            // We go through the work queue so that building can be done in parallel.
            // If linking against host libc installation, instead queue up jobs
            // for loading those files in the linker.
            if (comp.config.link_libc and is_exe_or_dyn_lib) {
                // If the "is darwin" check is moved below the libc_installation check below,
                // error.LibCInstallationMissingCrtDir is returned from lci.resolveCrtPaths().
                if (target.isDarwinLibC()) {
                    // TODO delete logic from MachO flush() and queue up tasks here instead.
                } else if (comp.libc_installation) |lci| {
                    const basenames = LibCInstallation.CrtBasenames.get(.{
                        .target = target,
                        .link_libc = comp.config.link_libc,
                        .output_mode = comp.config.output_mode,
                        .link_mode = comp.config.link_mode,
                        .pie = comp.config.pie,
                    });
                    const paths = try lci.resolveCrtPaths(arena, basenames, target);

                    const fields = @typeInfo(@TypeOf(paths)).@"struct".fields;
                    try comp.link_task_queue.shared.ensureUnusedCapacity(gpa, fields.len + 1);
                    inline for (fields) |field| {
                        if (@field(paths, field.name)) |path| {
                            comp.link_task_queue.shared.appendAssumeCapacity(.{ .load_object = path });
                            comp.remaining_prelink_tasks += 1;
                        }
                    }
                    // Loads the libraries provided by `target_util.libcFullLinkFlags(target)`.
                    comp.link_task_queue.shared.appendAssumeCapacity(.load_host_libc);
                    comp.remaining_prelink_tasks += 1;
                } else if (target.isMuslLibC()) {
                    if (!std.zig.target.canBuildLibC(target)) return error.LibCUnavailable;

                    if (musl.needsCrt0(comp.config.output_mode, comp.config.link_mode, comp.config.pie)) |f| {
                        comp.queued_jobs.musl_crt_file[@intFromEnum(f)] = true;
                        comp.remaining_prelink_tasks += 1;
                    }
                    switch (comp.config.link_mode) {
                        .static => comp.queued_jobs.musl_crt_file[@intFromEnum(musl.CrtFile.libc_a)] = true,
                        .dynamic => comp.queued_jobs.musl_crt_file[@intFromEnum(musl.CrtFile.libc_so)] = true,
                    }
                    comp.remaining_prelink_tasks += 1;
                } else if (target.isGnuLibC()) {
                    if (!std.zig.target.canBuildLibC(target)) return error.LibCUnavailable;

                    if (glibc.needsCrt0(comp.config.output_mode)) |f| {
                        comp.queued_jobs.glibc_crt_file[@intFromEnum(f)] = true;
                        comp.remaining_prelink_tasks += 1;
                    }
                    comp.queued_jobs.glibc_shared_objects = true;
                    comp.remaining_prelink_tasks += glibc.sharedObjectsCount(&target);

                    comp.queued_jobs.glibc_crt_file[@intFromEnum(glibc.CrtFile.libc_nonshared_a)] = true;
                    comp.remaining_prelink_tasks += 1;
                } else if (target.isWasiLibC()) {
                    if (!std.zig.target.canBuildLibC(target)) return error.LibCUnavailable;

                    for (comp.wasi_emulated_libs) |crt_file| {
                        comp.queued_jobs.wasi_libc_crt_file[@intFromEnum(crt_file)] = true;
                    }
                    comp.remaining_prelink_tasks += @intCast(comp.wasi_emulated_libs.len);

                    comp.queued_jobs.wasi_libc_crt_file[@intFromEnum(wasi_libc.execModelCrtFile(comp.config.wasi_exec_model))] = true;
                    comp.queued_jobs.wasi_libc_crt_file[@intFromEnum(wasi_libc.CrtFile.libc_a)] = true;
                    comp.remaining_prelink_tasks += 2;
                } else if (target.isMinGW()) {
                    if (!std.zig.target.canBuildLibC(target)) return error.LibCUnavailable;

                    const main_crt_file: mingw.CrtFile = if (is_dyn_lib) .dllcrt2_o else .crt2_o;
                    comp.queued_jobs.mingw_crt_file[@intFromEnum(main_crt_file)] = true;
                    comp.queued_jobs.mingw_crt_file[@intFromEnum(mingw.CrtFile.mingw32_lib)] = true;
                    comp.remaining_prelink_tasks += 2;

                    // When linking mingw-w64 there are some import libs we always need.
                    try comp.windows_libs.ensureUnusedCapacity(gpa, mingw.always_link_libs.len);
                    for (mingw.always_link_libs) |name| comp.windows_libs.putAssumeCapacity(name, {});
                } else if (target.os.tag == .freestanding and capable_of_building_zig_libc) {
                    comp.queued_jobs.zig_libc = true;
                    comp.remaining_prelink_tasks += 1;
                } else {
                    return error.LibCUnavailable;
                }
            }

            // Generate Windows import libs.
            if (target.os.tag == .windows) {
                const count = comp.windows_libs.count();
                for (0..count) |i| {
                    try comp.queueJob(.{ .windows_import_lib = i });
                }
                // when integrating coff linker with prelink, the above
                // queueJob will need to change into something else since those
                // jobs are dispatched *after* the link_task_wait_group.wait()
                // that happens when separateCodegenThreadOk() is false.
            }
            if (comp.wantBuildLibUnwindFromSource()) {
                comp.queued_jobs.libunwind = true;
                comp.remaining_prelink_tasks += 1;
            }
            if (build_options.have_llvm and is_exe_or_dyn_lib and comp.config.link_libcpp) {
                comp.queued_jobs.libcxx = true;
                comp.queued_jobs.libcxxabi = true;
                comp.remaining_prelink_tasks += 2;
            }
            if (build_options.have_llvm and is_exe_or_dyn_lib and comp.config.any_sanitize_thread) {
                comp.queued_jobs.libtsan = true;
                comp.remaining_prelink_tasks += 1;
            }

            if (capable_of_building_compiler_rt) {
                if (comp.compiler_rt_strat == .lib) {
                    log.debug("queuing a job to build compiler_rt_lib", .{});
                    comp.queued_jobs.compiler_rt_lib = true;
                    comp.remaining_prelink_tasks += 1;
                } else if (comp.compiler_rt_strat == .obj) {
                    log.debug("queuing a job to build compiler_rt_obj", .{});
                    // In this case we are making a static library, so we ask
                    // for a compiler-rt object to put in it.
                    comp.queued_jobs.compiler_rt_obj = true;
                    comp.remaining_prelink_tasks += 1;
                }

                if (comp.ubsan_rt_strat == .lib) {
                    log.debug("queuing a job to build ubsan_rt_lib", .{});
                    comp.queued_jobs.ubsan_rt_lib = true;
                    comp.remaining_prelink_tasks += 1;
                } else if (comp.ubsan_rt_strat == .obj) {
                    log.debug("queuing a job to build ubsan_rt_obj", .{});
                    comp.queued_jobs.ubsan_rt_obj = true;
                    comp.remaining_prelink_tasks += 1;
                }

                if (is_exe_or_dyn_lib and comp.config.any_fuzz) {
                    log.debug("queuing a job to build libfuzzer", .{});
                    comp.queued_jobs.fuzzer_lib = true;
                    comp.remaining_prelink_tasks += 1;
                }
            }
        }

        try comp.link_task_queue.shared.append(gpa, .load_explicitly_provided);
        comp.remaining_prelink_tasks += 1;
    }
    log.debug("total prelink tasks: {d}", .{comp.remaining_prelink_tasks});

    return comp;
}```
嗯，我需要理解这段Zig代码的功能。看起来这是一个名为`create`的函数，属于`Compilation`模块的构造函数。首先，函数接收三个参数：两个分配器`gpa`和`arena`，以及一个`CreateOptions`结构体。返回类型是一个指向`Compilation`的错误联合指针。

函数开始时处理`output_mode`，根据不同的模式设置布尔值`is_dyn_lib`和`is_exe_or_dyn_lib`。接着检查`linker_export_table`和`linker_import_table`是否同时为真，如果是的话返回错误。然后定义了一些变量，比如`have_zcu`，可能表示是否有Zig编译单元。

接下来，初始化`comp`变量，这里用到了`comp: { ... }`的块结构，可能是为了在块内初始化复杂的结构。这里分配了`root_name`，处理了各种配置选项，如`use_llvm`，以及一些`any_*`标志，这些标志可能用于默认设置的处理。

然后处理了`libc_dirs`，检测系统库目录。根据不同的条件设置`compiler_rt_strat`和`ubsan_rt_strat`，这两个策略可能决定了如何包含编译器运行时库和未定义行为检测库。接着创建了`compiler_rt_mod`和`ubsan_rt_mod`模块，并将它们添加到根模块的依赖中。

之后，处理了缓存相关的逻辑，包括创建`Cache`实例，添加各种前缀到缓存中，并更新哈希值以包含配置选项和版本信息。然后初始化`Compilation`结构体的各个字段，包括工作队列、源文件、缓存目录等。

在`switch`语句中，根据`cache_mode`的不同（增量或整体），处理了不同的缓存策略。增量模式下，计算哈希值并创建工件目录；整体模式下，延迟处理直到`update`函数。

最后，处理了与目标平台相关的链接逻辑，包括libc、编译器运行时库、Windows导入库等，并队列化了多个作业任务，如构建`libunwind`、`libcxx`等。最终返回初始化好的`comp`实例。

整个过程中，错误处理非常严格，每一步都有可能返回错误，并且使用了`errdefer`来确保资源释放。代码结构复杂，涉及多个配置选项、缓存管理和多任务队列，需要深入理解Zig编译器的内部机制才能完全掌握。
================================================
这段代码是 Zig 编译器中构建 `Compilation` 实例的核心逻辑，负责初始化编译上下文、管理依赖项和缓存，并处理多任务队列。以下是其思维模型的总结：

---

### **核心功能**
1. **编译配置解析**  
   - 根据 `CreateOptions` 设置输出模式（`output_mode`），判断是否为动态库/可执行文件。
   - 处理冲突选项（如同时启用导出表和导入表时抛出错误）。
   - 解析默认配置（如 `any_unwind_tables` 等标志），合并用户显式设置与隐式默认值。

2. **运行时库策略**  
   - **Compiler-RT** 和 **UBSan-RT**：根据输出模式和目标平台，决定以何种形式（库、对象、Zig模块）嵌入运行时支持。
   - 通过动态创建 `compiler_rt.zig` 和 `ubsan_rt.zig` 模块，注入到依赖图中。

3. **缓存管理**  
   - **增量模式**：计算哈希（包含不可变配置），生成唯一的工件目录路径，存放编译结果。
   - **整体模式**：延迟哈希计算，最终通过 `update()` 确定输出路径。
   - 缓存内容包含编译器版本、目标平台、优化选项等不可变配置。

4. **多任务队列**  
   - **预链接任务**：包括 C 对象编译、Windows 资源处理、LLVM 中间文件生成等。
   - **平台特定任务**：如 Musl/Glibc/WASI 的 C 运行时库构建、Windows 导入库生成。
   - **并行化**：通过线程池分发任务（如编译器-rt、libcxx 的构建）。

5. **链接准备**  
   - 根据目标平台加载系统库（如 libc、libunwind）。
   - 处理动态库/可执行文件的特殊逻辑（如入口点、栈大小、子系统版本）。
   - 管理符号表（导出/导入表）、段布局（EH Frame）、优化选项（GC 段、LTO）。

6. **错误处理与资源管理**  
   - 严格使用 `errdefer` 确保分配器、目录句柄等资源的释放。
   - 处理平台兼容性问题（如不支持构建 libc 时抛出 `LibCUnavailable`）。

---

### **关键数据结构**
- **`Compilation`**：编译上下文的核心容器，包含：
  - 分配器（`gpa` 和 `arena`）。
  - 模块依赖图（`zcu` 表示 Zig 编译单元）。
  - 缓存目录、链接器选项、平台配置。
  - 多任务队列（C 对象、Win32 资源、AST 生成等）。

- **`Cache`**：管理编译缓存，通过哈希标识唯一性，支持增量与整体模式。

- **`RtStrat`**：运行时库的嵌入策略（如 `.none`、`.zcu`、`.lib`、`.obj`）。

---

### **流程概览**
1. **初始化配置**  
   - 解析输出模式、目标平台、链接选项。
   - 创建运行时模块（如 `compiler_rt`），注入依赖。

2. **缓存与目录准备**  
   - 根据模式初始化缓存目录，计算哈希标识。

3. **任务队列填充**  
   - 添加 C 源文件、RC 文件、平台库的构建任务。
   - 按需加载系统库（如 libc、libc++）。

4. **链接器设置**  
   - 打开链接器文件（`bin_file`），配置段、符号、优化选项。

5. **错误边界**  
   - 确保失败时释放资源（如关闭目录句柄、销毁临时对象）。

---

### **设计特点**
- **模块化**：通过策略模式（如 `RtStrat`）分离不同运行时库的嵌入逻辑。
- **增量友好**：通过哈希标识复用缓存，减少重复编译。
- **平台抽象**：统一处理不同操作系统（Windows、Darwin、WASI）的链接逻辑。
- **并行化**：利用线程池加速任务处理（如多 C 文件的并行编译）。

---

### **典型应用场景**
- 命令行编译（`zig build-exe`、`zig build-lib`）。
- 交叉编译（为目标平台生成运行时库）。
- 静态分析（如启用 Sanitizer 或 Fuzzer 支持）。

此模型是 Zig 编译器高效、跨平台的核心机制，结合了静态编译的严谨性与动态任务的灵活性。