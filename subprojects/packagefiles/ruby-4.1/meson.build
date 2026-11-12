project('ruby-4.1', 'c', 'cpp', meson_version: '>=1.4.0')

find_program('cmp', required: true)

common = subproject('common')
cflags = common.get_variable('base_cflags') + get_option('c_args')
cppflags = common.get_variable('base_cppflags') + get_option('cpp_args')
ldflags = common.get_variable('base_ldflags') + get_option('c_link_args')
quotetool = common.get_variable('quotetool')

srcdir = meson.project_source_root()
if build_machine.system() in ['windows', 'cygwin']
    srcdir = run_command(common.get_variable('cygpath'), '-u', srcdir, check: true).stdout().strip()
endif

destdir = meson.project_build_root() / 'dist'
if build_machine.system() in ['windows', 'cygwin']
    destdir = run_command(common.get_variable('cygpath'), '-m', destdir, check: true).stdout().strip()
endif

baseruby_program = find_program('ruby')
# Ruby 4.1's monitor.rb expects Monitor in the core; older baseruby needs its own extension first.
host_monitor = run_command(
    baseruby_program, '--disable=gems', '-rrbconfig', '-e',
    'print File.join(RbConfig::CONFIG["rubylibdir"], "monitor.rb")',
    check: true,
).stdout().strip()
baseruby_command = (meson.version().version_compare('>=1.10.0') ? baseruby_program.cmd_array() : [baseruby_program]) + ['-r' + host_monitor]
baseruby = run_command(quotetool, baseruby_command, check: true).stdout().strip()

# YJIT is Rust; on Windows it is built for the MinGW target (x86_64-pc-windows-gnu).
# ponytail: 'auto' only turns it on for Windows x86_64; elsewhere it stays opt-in.
yjit_windows = host_machine.system() == 'windows' and host_machine.cpu_family() == 'x86_64'
rustc = find_program('rustc', native: true, required: get_option('yjit'))
yjit_rustc_flags = yjit_windows ? '--target=x86_64-pc-windows-gnu' : ''
yjit = rustc.found() and (yjit_windows or get_option('yjit').enabled())
if yjit and yjit_windows
    rust_target_dir = run_command(rustc, yjit_rustc_flags, '--print', 'target-libdir', check: true).stdout().strip()
    yjit = import('fs').is_dir(rust_target_dir)
    if not yjit and get_option('yjit').enabled()
        error('YJIT needs the Rust target: rustup target add x86_64-pc-windows-gnu')
    endif
endif
yjit_options = ['--disable-yjit']
yjit_ld = 'false'
yjit_objcopy = 'false'
yjit_env = {}
yjit_libs = []
if yjit
    cc_cmd = meson.get_compiler('c').cmd_array()
    yjit_options = ['--enable-yjit', 'RUSTC=' + rustc.full_path()]
    # Partial link of libyjit.a into libruby, keeping only its rb_* symbols global.
    yjit_ld = run_command(cc_cmd, '-print-prog-name=ld', check: true).stdout().strip()
    yjit_objcopy = run_command(cc_cmd, '-print-prog-name=objcopy', check: true).stdout().strip()
    yjit_env = {'rustc_flags': yjit_rustc_flags}
    if host_machine.system() == 'windows'
        foreach lib : ['ntdll', 'userenv', 'dbghelp']
            yjit_libs += meson.get_compiler('c').find_library(lib, required: true)
        endforeach
        yjit_libs += meson.get_compiler('c').find_library('gcc_eh', static: true, required: true)
    endif
endif
summary('YJIT', yjit)

yaml_dep = dependency('yaml-0.1', required: true)
ruby_deps = [yaml_dep, dependency('zlib', required: true)]
yaml_options = []
if yaml_dep.type_name() == 'internal'
    yaml_options = [
        '--with-libyaml-include=' + meson.global_source_root() / 'subprojects' / 'libyaml-0.2.5' / 'include',
        '--with-libyaml-lib=' + meson.global_build_root() / 'subprojects' / 'libyaml-0.2.5',
    ]
endif

ruby = declare_dependency(
    dependencies: [
        ruby_deps,
        import('unstable-external_project').add_project(
            'configure',
            configure_options: [
                '--srcdir=' + srcdir,
                '--prefix=' + '@PREFIX@',
                '--prefix=/',
                meson.version().version_compare('>=1.7.0') ? '--bindir=' + '@PREFIX@' / '@BINDIR@' : [],
                '--bindir=/bin',
                '--libdir=' + '@PREFIX@' / '@LIBDIR@',
                '--libdir=/lib',
                '--includedir=' + '@PREFIX@' / '@INCLUDEDIR@',
                '--includedir=/include',
                '--with-rubyhdrdir=/include',
                '--with-rubyarchhdrdir=/include',
                '--with-destdir=' + destdir,
                '--with-baseruby=' + baseruby,
                '--with-soname=ruby',
                '--disable-shared',
                '--disable-install-doc',
                '--with-out-ext=openssl,readline,dbm,gdbm',
                '--with-static-linked-ext',
                '--disable-rubygems',
                '--without-gmp',
                '--enable-mkmf-verbose',
                yjit_options,
                '--disable-zjit',
                yaml_options,
                'CC=' + run_command(quotetool, meson.get_compiler('c').cmd_array(), check: true).stdout().strip(),
                'CXX=' + run_command(quotetool, meson.get_compiler('cpp').cmd_array(), check: true).stdout().strip(),
                'AR=' + run_command(quotetool, common.get_variable('ar'), check: true).stdout().strip(),
                'RANLIB=' + run_command(quotetool, common.get_variable('ranlib'), check: true).stdout().strip(),
                'AS=false',
                'LD=' + yjit_ld,
                'NM=false',
                'OBJCOPY=' + yjit_objcopy,
                'OBJDUMP=false',
                'STRIP=false',
                'CFLAGS=' + run_command(quotetool, cflags, check: true).stdout().strip() + ' ' + get_option('cflags'),
                'CXXFLAGS=' + run_command(quotetool, cppflags, check: true).stdout().strip() + ' ' + get_option('cppflags'),
                'LDFLAGS=' + run_command(quotetool, ldflags, check: true).stdout().strip() + ' ' + get_option('ldflags'),
            ],
            cross_configure_options: [
                '--host=@HOST@',
                '--host=' + common.get_variable('host_triplet'),
            ],
            env: yjit_env + {
                'CONFIG_SITE': ' ',
                'MKXP_LIBGCC': meson.get_compiler('c').has_link_argument('-lgcc') ? '-lgcc' : '',
                'MKXP_DISABLE_LTO_FOR_LINK_TESTS': (meson.get_compiler('c').get_linker_id().split('.')[-1] == 'lld' or meson.get_compiler('c').get_linker_id() == 'lld-link') and host_machine.cpu_family() == 'arm' ? 'yes' : 'no',
                'DLLWRAP': 'false',
                'WINDRES': run_command(quotetool, common.get_variable('windres'), check: true).stdout().strip(),
                'ac_cv_func___builtin_setjmp': 'no',
                'ac_cv_func_crypt': 'no',
                'ac_cv_func_crypt_r': 'no',
                'ac_cv_header_crypt_h': 'no',
                'ac_cv_lib_crypt_crypt': 'no',
                'ac_cv_lib_z_uncompress': 'yes',
                'rb_cv_broken_crypt': 'no',
                'rb_cv_gcc_atomic_builtins': 'yes',
                'rb_cv_gcc_sync_builtins': 'no',
                'rb_cv_prog_gnu_ld': common.get_variable('c_linker_is_gnu') ? 'yes' : 'no',
                'rb_cv_symbol_prefix': meson.get_compiler('c').symbols_have_underscore_prefix() ? '_' : 'NONE',
                # Preserve the Ruby 3.4 control's MinGW libssp workaround.
                'stack_protector': 'no',
            },
            depends: [
                executable('deps', sources: 'deps-stub.c',
                           dependencies: ruby_deps),
            ],
            verbose: true,
        ).dependency('ruby-static').partial_dependency(sources: true),
        host_machine.system() not in ['windows', 'cygwin', 'darwin']
            ? [
                meson.get_compiler('c').find_library('atomic', required: false),
                meson.get_compiler('c').find_library('dl', required: false),
                meson.get_compiler('c').find_library('rt', required: false),
            ]
            : [],
        host_machine.system() in ['windows', 'cygwin']
            ? [
                meson.get_compiler('c').find_library('shell32', required: true),
                meson.get_compiler('c').find_library('ws2_32', required: true),
                meson.get_compiler('c').find_library('iphlpapi', required: true),
                meson.get_compiler('c').find_library('imagehlp', required: true),
                meson.get_compiler('c').find_library('shlwapi', required: true),
                meson.get_compiler('c').find_library('bcrypt', required: true),
            ]
            : [],
        yjit_libs,
        host_machine.system() == 'freebsd'
            ? [meson.get_compiler('c').find_library('procstat', required: true)]
            : [],
        host_machine.system() in ['freebsd', 'netbsd']
            ? [meson.get_compiler('c').find_library('execinfo', required: true)]
            : [],
    ],
    compile_args: ['-I' + meson.project_build_root() / 'dist' / 'include'],
    link_args: [
        meson.get_compiler('c').first_supported_link_argument('-rdynamic', '-Wl,-export-dynamic'),
        (host_machine.system() in ['windows', 'cygwin']) and not meson.get_compiler('c').has_header_symbol('math.h', 'lgamma_r')
            ? [meson.project_build_root() / 'build' / 'lgamma_r.o']
            : [],
        meson.project_build_root() / 'build' / 'ext' / 'extinit.a',
        meson.project_build_root() / 'build' / 'ext' / 'cgi' / 'escape' / 'escape.a',
        meson.project_build_root() / 'build' / 'ext' / 'continuation' / 'continuation.a',
        meson.project_build_root() / 'build' / 'ext' / 'coverage' / 'coverage.a',
        meson.project_build_root() / 'build' / 'ext' / 'date' / 'date_core.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'digest.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'blake3' / 'blake3.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'bubblebabble' / 'bubblebabble.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'crc32' / 'crc32.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'md5' / 'md5.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'rmd160' / 'rmd160.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'sha1' / 'sha1.a',
        meson.project_build_root() / 'build' / 'ext' / 'digest' / 'sha2' / 'sha2.a',
        meson.project_build_root() / 'build' / 'ext' / 'erb' / 'escape' / 'escape.a',
        meson.project_build_root() / 'build' / 'ext' / 'etc' / 'etc.a',
        meson.project_build_root() / 'build' / 'ext' / 'fcntl' / 'fcntl.a',
        meson.project_build_root() / 'build' / 'ext' / 'io' / 'console' / 'console.a',
        meson.project_build_root() / 'build' / 'ext' / 'io' / 'nonblock' / 'nonblock.a',
        meson.project_build_root() / 'build' / 'ext' / 'json' / 'generator' / 'generator.a',
        meson.project_build_root() / 'build' / 'ext' / 'json' / 'parser' / 'parser.a',
        meson.project_build_root() / 'build' / 'ext' / 'objspace' / 'objspace.a',
        meson.project_build_root() / 'build' / 'ext' / 'psych' / 'psych.a',
        meson.project_build_root() / 'build' / 'ext' / 'rbconfig' / 'sizeof' / 'sizeof.a',
        meson.project_build_root() / 'build' / 'ext' / 'ripper' / 'ripper.a',
        meson.project_build_root() / 'build' / 'ext' / 'socket' / 'socket.a',
        meson.project_build_root() / 'build' / 'ext' / 'stringio' / 'stringio.a',
        meson.project_build_root() / 'build' / 'ext' / 'strscan' / 'strscan.a',
        meson.project_build_root() / 'build' / 'ext' / 'zlib' / 'zlib.a',
        host_machine.system() in ['windows', 'cygwin']
            ? [
                meson.project_build_root() / 'build' / 'ext' / 'win32' / 'resolv' / 'resolv.a',
            ]
            : [meson.project_build_root() / 'build' / 'ext' / 'pty' / 'pty.a'],
        meson.project_build_root() / 'build' / 'enc' / 'encinit.a',
        meson.project_build_root() / 'build' / 'enc' / 'libenc.a',
        meson.project_build_root() / 'build' / 'enc' / 'libtrans.a',
        meson.project_build_root() / 'build' / 'libruby-static.a',
        # rustc's raw-dylib import stubs, kept out of the partial link (defs/jit.mk)
        yjit and yjit_windows
            ? [meson.project_build_root() / 'build' / 'target' / 'release' / 'libyjit-imports.a']
            : [],
    ],
)

ruby_build_dir = meson.project_build_root()
# Ruby 4.1.0-preview1 uses development ABI 5 in its library directory.
ruby_stdlib_dir = ruby_build_dir / 'dist' / 'lib' / 'ruby' / '4.1.0+5'
miniruby = custom_target(
    'miniruby',
    output: 'miniruby',
    command: [
        common.get_variable('python'),
        files('build-miniruby.py'),
        baseruby_command,
        ruby_build_dir / 'build',
        '@OUTPUT@',
    ],
    depends: [
        executable('miniruby-deps', sources: 'deps-stub.c',
                   dependencies: ruby),
    ],
    build_by_default: true,
)

meson.override_dependency('ruby-4.1', ruby, static: true)
