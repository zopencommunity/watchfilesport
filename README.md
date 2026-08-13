# watchfilesport

z/OS port of [watchfiles](https://github.com/samuelcolvin/watchfiles) v1.2.0 — fast file watching for Python, powered by Rust.

## Overview

watchfiles provides an efficient file-watching API for Python. The Rust core uses the
[notify](https://github.com/notify-rs/notify) crate. On z/OS, the `PollWatcher` backend
is used (notify's fallback for unsupported platforms), which polls file metadata via `stat()`.
This works correctly on HFS/zFS filesystems.

## Prebuilt wheels

| Wheel | Python |
|-------|--------|
| `watchfiles-1.2.0-cp312-cp312-os390_29_00_8561.whl` | Python 3.12 |
| `watchfiles-1.2.0-cp313-cp313-os390_29_00_8561.whl` | Python 3.13 |
| `watchfiles-1.2.0-cp314-cp314-zos.whl` | Python 3.14 |

## Installation

```sh
pip install watchfiles-1.2.0-cp312-cp312-os390_29_00_8561.whl  # Python 3.12
```

## Building from source

Cross-compiled on Linux-on-Power using IBM's Rust cross-compiler targeting `s390x-ibm-zos`.
See [rust-scripts](https://github.ibm.com/compiler/rust-scripts) branch `itodorov/zos-cross-compile-setup`.

Key notes:
- Uses `PollWatcher` backend (notify's platform fallback) — no inotify/kqueue needed
- `pyo3` 0.29.2 with z/OS patches from `github.ibm.com/itodorov/pyo3`
- `rustc-wrapper-wf.sh` injects `--extern pyo3_macros` for every s390x compilation
- `[lib] name = "_rust_notify"` in `Cargo.toml` ensures correct GOFF DLL name

## Cross-compilation

```sh
export PATH="/gsa/rtpgsa/projects/r/rustcross/v186/lop/rustcross/260610/usr/local/bin:$PATH"
export CROSS_SERVER_DOMAIN=zoscan2b.pok.stglabs.ibm.com
export CROSS_SERVER_PORT=5051

cd /tmp/watchfiles
cat > .cargo/config.toml << TOML
[build]
rustc-wrapper = "/home/itodorov/rust_bld/toolchain/rustc-wrapper-wf.sh"
[target.s390x-ibm-zos]
linker = "/home/itodorov/rust_bld/toolchain/s390x-ibm-zos-cc"
rustflags = [
  "-C", "link-arg=/usr/lpp/IBM/cyp/v3r12/pyz/lib/libpython3.12.x",
  "-C", "target-feature=-vector",
]
TOML

PYO3_CONFIG_FILE=.../pyo3-zos-config.txt PYO3_NO_PYTHON=1 \
  cargo build --release --target s390x-ibm-zos

python3 .../build_zos_wheel.py watchfiles --pytag cp312 \
  --platform os390_29_00_8561 --out-dir /tmp/wheels-release
```
