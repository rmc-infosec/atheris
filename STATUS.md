# Atheris ARM/macOS Wheel Support - Status

## Objective

Add cross-platform binary wheel builds for ARM Linux and macOS (Intel + Apple Silicon) to the Atheris fuzzing library.

## Current State (Upstream)

**PyPI wheels available:**
- `manylinux2014_x86_64` only (Python 3.11, 3.12, 3.13)

**Missing:**
- `manylinux2014_aarch64` (ARM Linux)
- `macosx_x86_64` (Intel Mac)
- `macosx_arm64` (Apple Silicon)

## Related GitHub Issues

| Issue | Status | Summary |
|-------|--------|---------|
| [#52](https://github.com/google/atheris/issues/52) | Open (Dec 2022) | Request for cibuildwheel - assigned to 3.0.0 milestone, never completed |
| [#68](https://github.com/google/atheris/pull/68) | Merged (Aug 2023) | Added ARM64 Linux support in code |
| [#80](https://github.com/google/atheris/issues/80) | Open | macOS 14.2.1 build failures |
| [#74](https://github.com/google/atheris/issues/74) | Open | libFuzzer dependency issues across platforms |

## Testing Results

### M1 Mac (macOS 15.6, ARM64)

| Test | Python 3.12 | Python 3.13 |
|------|-------------|-------------|
| Build with Homebrew LLVM | ✓ | ✓ |
| Import atheris | ✓ | ✓ |
| FuzzedDataProvider | ✓ | ✓ |
| Fuzz test execution | ✓ | ✓ |

**Environment used:**
```bash
brew install llvm
export CLANG_BIN="$(brew --prefix llvm)/bin/clang"
export CC="$(brew --prefix llvm)/bin/clang"
export CXX="$(brew --prefix llvm)/bin/clang++"
export LDFLAGS="-L$(brew --prefix llvm)/lib/c++ -L$(brew --prefix llvm)/lib -Wl,-rpath,$(brew --prefix llvm)/lib/c++"
export CPPFLAGS="-I$(brew --prefix llvm)/include"
```

### Critical Finding: libc++ Linking

**Problem:** Homebrew LLVM uses newer libc++ with symbols not in system libc++.

**Symptom:**
```
ImportError: symbol not found in flat namespace '__ZNSt3__113__hash_memoryEPKvm'
```

**Solution:** Must link against Homebrew's libc++ explicitly:
```
LDFLAGS="-L$(brew --prefix llvm)/lib/c++ -Wl,-rpath,$(brew --prefix llvm)/lib/c++"
```

**Verification:**
```bash
# Correct (works):
otool -L atheris/native.*.so
  /opt/homebrew/opt/llvm/lib/c++/libc++.1.dylib

# Incorrect (fails):
  /usr/lib/libc++.1.dylib
```

## PR Implementation

### Files Changed

1. **`pyproject.toml`** - Added cibuildwheel configuration:
   - `[build-system]` - PEP 517 build requirements
   - `[tool.cibuildwheel]` - Skip PyPy/musllinux, Python 3.11-3.13
   - `[tool.cibuildwheel.linux]` - x86_64 + aarch64, manylinux2014
   - `[tool.cibuildwheel.macos]` - x86_64 + arm64, Homebrew LLVM

2. **`.github/workflows/wheels.yaml`** - New workflow:
   - Matrix: ubuntu-latest (x86_64, aarch64), macos-13 (x86_64), macos-14 (arm64)
   - QEMU for aarch64 emulation
   - Homebrew LLVM setup for macOS
   - PyPI trusted publishing on tagged releases

### Wheels Produced

| Platform | Arch | Python Versions |
|----------|------|-----------------|
| manylinux2014 | x86_64 | 3.11, 3.12, 3.13 |
| manylinux2014 | aarch64 | 3.11, 3.12, 3.13 |
| macosx | x86_64 | 3.11, 3.12, 3.13 |
| macosx | arm64 | 3.11, 3.12, 3.13 |

**Total: 12 wheels** (up from 3)

## Next Steps

1. [ ] Comment on Issue #52 to check maintainer interest
2. [ ] Fork google/atheris
3. [ ] Push changes to fork
4. [ ] Open PR with test results
5. [ ] Address any CI failures

## Known Limitations

- **macOS requires Homebrew LLVM** - Apple Clang doesn't include libFuzzer
- **No Windows support** - Shell-based build scripts incompatible
- **ASan/UBSan optional** - Sanitizer libraries may not be bundled on all platforms
- **Python 3.14+ unsupported** - Atheris only supports 3.6-3.13 currently

## Notes

- Wheel size ~7.5MB each (includes libFuzzer)
- aarch64 Linux builds use QEMU emulation (slower CI)
- macOS builds require `brew install llvm` in CI
