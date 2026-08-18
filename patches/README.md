# watchfiles v1.2.0 z/OS patches

Patches to cross-compile [watchfiles v1.2.0](https://github.com/samuelcolvin/watchfiles)
for `s390x-ibm-zos`.

## Changes

### `.cargo/config.toml`
- `rustc-wrapper = "rustc-wrapper-wf.sh"` — injects `--extern pyo3_macros` for every
  s390x compilation (fixes proc-macro resolution across release profile recompile)
- `linker = "s390x-ibm-zos-cc"` for the z/OS target
- `link-arg = <libpython3.XY.x>` side-deck for the target Python version
- `-C target-feature=-vector` (required: LLVM auto-vectorises at opt-level≥2 without this)

### `Cargo.toml` `[patch.crates-io]`
- `pyo3` → IBM fork `github.ibm.com/itodorov/pyo3` branch `itodorov/zos-support` (0.29.2)
- `libc` → IBM fork `github.ibm.com/compiler/rust-libc` branch `zOS.0.2.169` (0.2.189)
- `target-lexicon` → git main (has `OperatingSystem::Zos`)

## No notify patch needed

`notify 8.0.0` automatically selects `PollWatcher` for unknown platforms (z/OS falls
through all `#[cfg(target_os = ...)]` guards to the final catch-all):

```rust
#[cfg(not(any(target_os = "linux", target_os = "android", target_os = "macos",
              target_os = "windows", target_os = "freebsd", target_os = "openbsd",
              target_os = "netbsd", target_os = "dragonfly", target_os = "ios")))]
pub type RecommendedWatcher = PollWatcher;
```

Poll-based watching works correctly on z/OS — it does periodic `stat()` comparisons.
Inotify could be added later for efficiency (z/OS 3.1 has `/usr/include/sys/inotify.h`),
but is not required for correct operation.

## rustc-wrapper approach

Unlike `--extern` in rustflags (which only applies to the final crate compilation),
`rustc-wrapper-wf.sh` intercepts every `rustc` invocation for the `s390x-ibm-zos` target
and dynamically injects `--extern pyo3_macros=<path>` when compiling pyo3 and other crates.
This is required because `cargo build --release` rebuilds pyo3 from scratch in release
profile, and Cargo 1.86 does not propagate proc-macro `--extern` flags automatically during
cross-compilation.

## Build steps

```bash
git clone --depth=1 https://github.com/samuelcolvin/watchfiles /tmp/watchfiles
cd /tmp/watchfiles
git apply /path/to/rust-scripts/cross/patches/watchfiles-zos/watchfiles-1.2.0-zos.patch

# Step 1: native cargo check to build pyo3_macros .so for the host
export PYO3_CONFIG_FILE=/path/to/pyo3-zos-config.txt
export PYO3_NO_PYTHON=1
cargo check   # errors expected (Python abi config) but pyo3_macros.so is built

# Step 2: cargo check for s390x target (seeds rlib cache)
export CC_s390x_ibm_zos=/home/itodorov/rust_bld/toolchain/s390x-ibm-zos-cc
export AR_s390x_ibm_zos=/home/itodorov/rust_bld/toolchain/s390x-ibm-zos-ar
export CROSS_SERVER_DOMAIN=your.zos.host
export CROSS_SERVER_PORT=5051
cargo check --target s390x-ibm-zos

# Step 3: release build (uses rustc-wrapper-wf.sh)
#   Update .cargo/config.toml with rustc-wrapper + correct libpython*.x path
cargo build --release --target s390x-ibm-zos

# Step 4: assemble wheel
python3 /path/to/build_zos_wheel.py watchfiles \
  --pytag cp312 --platform os390_29_00_8561 --out-dir /tmp/wheels-release
```

## Status

- [x] Builds successfully for Python 3.12, 3.13, 3.14
- [x] `RustNotify` extension loads on z/OS
- [x] `PollWatcher` detects file changes on HFS/zFS
- [x] Wheels released at https://github.com/zopencommunity/watchfilesport/releases/tag/v1.2.0
- [x] zopencommunity port created: https://github.com/zopencommunity/watchfilesport
