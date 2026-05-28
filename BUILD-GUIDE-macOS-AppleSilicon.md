# 在 macOS Apple Silicon 上构建 RISC-V 32-bit Newlib Multilib 工具链

> 完整经验沉淀。适用于在 macOS 14+ / Apple Silicon (M1/M2/M3/M4) 上从源码构建 `riscv32-unknown-elf-gcc` 多 multilib bare-metal 工具链。
> 本指南记录了 2026-05-27 在 macOS 26.4 + M4 上 7 次失败 1 次成功的全部踩坑过程。

---

## 0. TL;DR

如果你只想要**能跑的命令**：

```bash
# 1. 装依赖
brew install autoconf automake libtool texinfo flock gawk gnu-sed \
             gmp mpfr libmpc isl zlib expat libslirp python@3.12 make bash

# 2. clone 仓库
git clone https://github.com/riscv-collab/riscv-gnu-toolchain
cd riscv-gnu-toolchain
git submodule update --init --recursive   # 或让 make 自动按需 fetch

# 3. 拷贝本仓库根目录的 build-aarch64.sh
# 4. 一键构建
./build-aarch64.sh build
```

预计耗时：M4 10 核约 **11 分钟**（修复脚本就位后，纯净状态从零到完成）。

---

## 1. 背景

### 目标

- Host：macOS 14+ on Apple Silicon (arm64)
- Target：`riscv32-unknown-elf` bare-metal
- 工具链：GCC 13.2.0 + binutils 2.42 + newlib 4.4.0 + gdb 14.2
- 用途：HPMicro / SiFive / GD32V / CH32V 等 RISC-V 32-bit MCU 固件开发
- 关键要求：与 Intel 版（HPMicro 官方发布）行为一致

### 输出物结构

```
<install-prefix>/
├── bin/                              31 个 native arm64 工具
│   ├── riscv32-unknown-elf-gcc       
│   ├── riscv32-unknown-elf-g++
│   ├── riscv32-unknown-elf-gdb
│   ├── riscv32-unknown-elf-ld
│   ├── riscv32-unknown-elf-objcopy
│   └── ... (objdump, readelf, size, strip, as, ar, nm, ...)
├── lib/gcc/riscv32-unknown-elf/13.2.0/
├── libexec/gcc/riscv32-unknown-elf/13.2.0/
└── riscv32-unknown-elf/
    ├── include/                      newlib headers
    ├── bin/
    └── lib/                          12 个 multilib 变体
        ├── rv32imac_zicsr_zifencei/ilp32/
        ├── rv32imafc_zicsr_zifencei/ilp32/
        ├── rv32imafc_zicsr_zifencei/ilp32f/
        ├── rv32imafdc_zicsr/ilp32/
        ├── rv32imafdc_zicsr/ilp32f/
        ├── rv32imafdc_zicsr/ilp32d/
        ├── rv32imac_zicsr_zifencei_zba_zbb_zbc_zbs/ilp32/
        ├── rv32imafc_zicsr_zifencei_zba_zbb_zbc_zbs/ilp32/
        ├── rv32imafc_zicsr_zifencei_zba_zbb_zbc_zbs/ilp32f/
        ├── rv32imafdc_zicsr_zba_zbb_zbc_zbs/ilp32/
        ├── rv32imafdc_zicsr_zba_zbb_zbc_zbs/ilp32f/
        └── rv32imafdc_zicsr_zba_zbb_zbc_zbs/ilp32d/
```

---

## 2. 前置依赖

### macOS 系统

| 项 | 最低 | 推荐 | 备注 |
|---|---|---|---|
| 系统版本 | 14.0 | 14.5+ | 本指南验证于 26.4 |
| 架构 | arm64 | M1/2/3/4 都行 | |
| Xcode Command Line Tools | 已装 | 最新 | `xcode-select --install` |
| 磁盘空间 | 15 GiB 可用 | | 中间产物大 |
| 内存 | 8 GiB | 16+ GiB | -j10 时 |

### Homebrew 必需包

```bash
brew install autoconf automake libtool texinfo flock \
             gawk gnu-sed \
             gmp mpfr libmpc isl \
             zlib expat libslirp \
             python@3.12 make bash
```

**为什么各装这些**：

| 包 | 用途 | 不装会怎样 |
|---|---|---|
| `autoconf` | 已装好的 GCC submodule 不需要，但若手动 regen 时要 | 一般不会立刻 fail |
| `automake` | 同上，给 aclocal | 触发 regen 时报错 |
| `libtool` | binutils/gdb 链接 .la 文件用 | binutils 链接 fail |
| `texinfo` | makeinfo 生成 .info 文件 | `makeinfo command not found` |
| `flock` | macOS 没有，并行构建里序列化锁 | 个别 make 步骤 fail |
| `gawk` | 严格说本指南**不用 gawk**（坑 #6），但 brew 不装也能跑（系统有 /usr/bin/awk） | 装了反而要绕过 |
| `gnu-sed` | macOS 自带的 BSD sed 部分用法不兼容 | sed 调用挂在某些 in-place 写法上 |
| `gmp mpfr libmpc isl` | **只给 GDB**用（GCC 用 in-tree 自带源码，见坑 #5） | GDB configure fail |
| `zlib` | GCC `--with-system-zlib` | 缺则 GCC 编译失败或用 in-tree zlib |
| `expat` | gdb 读 XML target description | gdb 功能减 |
| `libslirp` | qemu user-mode 网络（本次没编 qemu，但 README 推荐） | 不影响 |
| `python@3.12` | gdb python scripting，autoconf 检查 | gdb 缺 python feature |
| `make` | macOS 自带 GNU Make 3.81 太老，brew make 是 4.4+ | 可能踩 GNU Make 4+ 特性 |
| `bash` | macOS 自带 bash 3.2 (2007 年版！GPL3 后未更新) | 多数 build script 能跑，安全起见装 5+ |

### 验证依赖就位

```bash
for pkg in autoconf automake libtool texinfo flock gawk gnu-sed \
           gmp mpfr libmpc isl zlib expat libslirp python@3.12 make bash; do
  brew list --formula | grep -qx "$pkg" && echo "✓ $pkg" || echo "✗ $pkg MISSING"
done
```

### SDK 选择

```bash
ls /Library/Developer/CommandLineTools/SDKs/
# MacOSX.sdk -> MacOSX15.2.sdk (default on macOS 15+)
# MacOSX14.5.sdk
# MacOSX13.3.sdk
# ...
```

**推荐：MacOSX14.5.sdk**

```bash
export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX14.5.sdk
```

为什么：
- macOS 15.2 SDK 的 libc++ 改动太激进，会触发 Apple Clang 16 在 `-std=gnu++11` 模式下的属性校验
- 13.3 SDK 也能用，但 14.5 更接近 GCC 13.2 发布期的环境
- **不要用更老的 13.x SDK 编 Apple Silicon native 工具**（一些 arm64 API 缺失）

---

## 3. 完整构建命令（修复后版本）

```bash
INSTALL_PREFIX="$HOME/path/to/riscv32-toolchain-aarch64"
REPO_ROOT=$(pwd)   # 在 riscv-gnu-toolchain 仓库里

# 受控环境（这些变量只在本 subshell 有效，不污染系统）
export PATH="/opt/homebrew/opt/gnu-sed/libexec/gnubin:\
/opt/homebrew/opt/make/libexec/gnubin:\
/opt/homebrew/opt/python@3.12/libexec/bin:\
/opt/homebrew/bin:\
/usr/bin:/bin:/usr/sbin:/sbin"
export SDKROOT=/Library/Developer/CommandLineTools/SDKs/MacOSX14.5.sdk
export MACOSX_DEPLOYMENT_TARGET=14.0
export CC=/usr/bin/clang
export CXX=/usr/bin/clang++

# Host 端 C/C++ 编译旗标（坑 #7、#8 的兼容补丁）
CXX_FLAGS_FIX="-O2 -isysroot $SDKROOT -stdlib=libc++ \
  -D_LIBCPP_NO_ABI_TAG \
  -Wno-error=register \
  -Wno-error=deprecated-declarations \
  -Wno-error=deprecated-non-prototype"
C_FLAGS_FIX="-O2 -isysroot $SDKROOT \
  -Wno-error=implicit-function-declaration \
  -Wno-error=incompatible-pointer-types \
  -Wno-error=deprecated-non-prototype \
  -Wno-error=nullability-completeness"
LD_FLAGS_FIX="-Wl,-syslibroot,$SDKROOT"
export CFLAGS_FOR_BUILD="$C_FLAGS_FIX"
export CXXFLAGS_FOR_BUILD="$CXX_FLAGS_FIX"
export LDFLAGS_FOR_BUILD="$LD_FLAGS_FIX"
export CFLAGS="$C_FLAGS_FIX"
export CXXFLAGS="$CXX_FLAGS_FIX"
export LDFLAGS="$LD_FLAGS_FIX"
export CFLAGS_FOR_TARGET="-Os -mcmodel=medlow"
export CXXFLAGS_FOR_TARGET="-Os -mcmodel=medlow"

# 关键：在 configure 之前 patch GCC system.h（坑 #7）
# 把 <map>/<list>/<algorithm> 等移到 safe-ctype.h 之前
python3 - "$REPO_ROOT/gcc/gcc/system.h" <<'PY'
import sys
path = sys.argv[1]
src = open(path).read()
old_block = '''#ifdef __cplusplus
#if defined (INCLUDE_ALGORITHM) || !defined (HAVE_SWAP_IN_UTILITY)
# include <algorithm>
#endif
#ifdef INCLUDE_LIST
# include <list>
#endif
#ifdef INCLUDE_MAP
# include <map>
#endif
#ifdef INCLUDE_SET
# include <set>
#endif
#ifdef INCLUDE_VECTOR
# include <vector>
#endif
#ifdef INCLUDE_ARRAY
# include <array>
#endif
#ifdef INCLUDE_FUNCTIONAL
# include <functional>
#endif
# include <cstring>
# include <initializer_list>
# include <new>
# include <utility>
# include <type_traits>
#endif

'''
marker = '/* INCLUDE_MAP_BEFORE_SAFE_CTYPE_PATCHED */\n'
if marker in src:
    sys.exit(0)
anchor = '#ifdef INCLUDE_STRING\n# include <string>\n#endif\n#endif\n'
src = src.replace(old_block, '', 1)
src = src.replace(anchor, anchor + marker + old_block, 1)
open(path, 'w').write(src)
PY

# 跑 wrapper configure
cd "$REPO_ROOT"
./configure \
  --prefix="$INSTALL_PREFIX" \
  --with-arch=rv32imac_zicsr_zifencei \
  --with-abi=ilp32 \
  --with-tune=rocket \
  --with-isa-spec=20191213 \
  --with-cmodel=medlow \
  --with-system-zlib \
  --enable-multilib \
  --with-multilib-generator='rv32imac_zicsr_zifencei-ilp32--;rv32imafc_zifencei-ilp32--;rv32imafc_zifencei-ilp32f--;rv32imafdc-ilp32--;rv32imafdc-ilp32f--;rv32imafdc-ilp32d--;rv32imac_zicsr_zifencei_zba_zbb_zbc_zbs-ilp32--;rv32imafc_zifencei_zba_zbb_zbc_zbs-ilp32--;rv32imafc_zifencei_zba_zbb_zbc_zbs-ilp32f--;rv32imafdc_zba_zbb_zbc_zbs-ilp32--;rv32imafdc_zba_zbb_zbc_zbs-ilp32f--;rv32imafdc_zba_zbb_zbc_zbs-ilp32d--'

# 关键：configure 后立刻 patch awk wrapper（坑 #6）
# 让 scripts/wrapper/awk/awk 实际执行 BSD awk
cat > scripts/wrapper/awk/awk <<'AWK_WRAPPER'
#!/bin/sh
exec /usr/bin/awk "$@"
AWK_WRAPPER
chmod +x scripts/wrapper/awk/awk

# 跑 make，命令行强制 AWK 覆盖 Makefile.in 里的硬编码
BREW="--with-gmp=/opt/homebrew/opt/gmp \
      --with-mpfr=/opt/homebrew/opt/mpfr \
      --with-mpc=/opt/homebrew/opt/libmpc \
      --with-isl=/opt/homebrew/opt/isl"

make -j$(sysctl -n hw.ncpu) \
  AWK=/usr/bin/awk \
  GCC_EXTRA_CONFIGURE_FLAGS="--disable-libcc1" \
  GDB_TARGET_FLAGS_EXTRA="$BREW" \
  GDB_NATIVE_FLAGS_EXTRA="$BREW"
```

---

## 4. 踩过的 8 个坑及解决方案

### 坑 #1：默认 multilib 出 `riscv64-unknown-elf-` 前缀，不是 `riscv32`

**症状**：跑 `./configure --enable-multilib` 默认 target 是 `riscv64-unknown-elf`，bin 目录里全是 `riscv64-unknown-elf-*`，与 HPMicro / 多数 32-bit MCU 期望的 `riscv32-unknown-elf-*` 不符。

**根因**：riscv-gnu-toolchain wrapper 的默认 target 由 `--with-arch` 决定。如果没指定，默认是 `rv64gc`，于是 wrapper 推断 target 为 `riscv64-unknown-elf`。

**修复**：显式传 `--with-arch=rv32...` 和 `--with-abi=ilp32`

```bash
./configure \
  --with-arch=rv32imac_zicsr_zifencei \
  --with-abi=ilp32 \
  --with-tune=rocket \
  ...
```

**怎么检测**：
```bash
grep "^target" Makefile | head -3   # 应该看到 riscv32-unknown-elf
```

---

### 坑 #2：wrapper configure 不认 `--with-gmp` 等参数

**症状**：
```
configure: WARNING: unrecognized options: --with-gmp, --with-mpfr, --with-mpc, --with-isl
```

**根因**：riscv-gnu-toolchain 根目录的 `./configure` 是个 wrapper（autoconf 生成），只接受 wrapper 自己的参数。`--with-gmp` 这些是给 GCC submodule 的 configure 用的。

**修复**：把 `--with-gmp` 等放到 `make` 命令行通过 `GCC_EXTRA_CONFIGURE_FLAGS` 传给 GCC、`GDB_TARGET_FLAGS_EXTRA` 传给 GDB：

```bash
make -j10 \
  GCC_EXTRA_CONFIGURE_FLAGS="--disable-libcc1" \
  GDB_TARGET_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp ..." \
  GDB_NATIVE_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp ..."
```

注意：**不要给 GCC 传 `--with-gmp=brew`**（见坑 #5）。

---

### 坑 #3：`set -o pipefail` + `grep -qx` 触发 SIGPIPE 误判

**症状**：在 bash 脚本里 `if ! brew list --formula | grep -qx "autoconf"; then echo missing; fi` 总是报 autoconf 缺失，但实际装了。

**根因**：`grep -q` 一旦匹配立即退出 0。上游 `brew list` 还在写管道，被 SIGPIPE 中断，退出非 0。`set -o pipefail` 让管道返回非 0，`if ! ...` 取反后判断"未匹配"，误加到缺失列表。

**复现**：
```bash
set -euo pipefail
brew list --formula | grep -qx "autoconf" && echo found || echo missing
# 输出: missing （即使 autoconf 装了）
```

**修复**：先一次性 capture brew list 输出，再多次匹配：

```bash
brew_list="$(brew list --formula 2>/dev/null)"
for pkg in autoconf automake ...; do
  if ! grep -qx "$pkg" <<< "$brew_list"; then
    missing+=("$pkg")
  fi
done
```

或者关闭 pipefail（不推荐，会掩盖其他错误）。

---

### 坑 #4：默认 `make` target 把 GDB 拉进依赖，GDB 缺 GMP

**症状**：
```
configure: error: Building GDB requires GMP 4.2+, and MPFR 3.1.0+.
```

**根因**：riscv-gnu-toolchain 的 `Makefile:140` 让 `newlib` target 依赖 `stamps/build-gdb-newlib`。默认 `make` (`all: newlib`) 会编 GDB。GDB submodule 的 configure 用 GMP/MPFR 做计算，找不到就 fail。

**修复**：给 make 传 `GDB_TARGET_FLAGS_EXTRA` 把 brew GMP 路径传给 GDB：

```bash
make -j10 \
  GDB_TARGET_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp --with-mpfr=/opt/homebrew/opt/mpfr ..." \
  GDB_NATIVE_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp --with-mpfr=/opt/homebrew/opt/mpfr ..."
```

`GDB_TARGET_FLAGS_EXTRA` 用于 cross-targeted GDB，`GDB_NATIVE_FLAGS_EXTRA` 用于 host GDB。两个都加最稳。

如果**只想要工具链不要 GDB**，可以 `make stamps/build-newlib stamps/build-newlib-nano`，但默认 build 流程是带 GDB 的。

---

### 坑 #5：给 GCC 传 `--with-gmp=brew` 反而让 in-tree ISL 找不到 gmp.h

**症状**：
```
checking which gmp to use... system
checking for gmp.h... no
configure: error: gmp.h header not found
make[1]: *** [Makefile:6505: configure-isl] Error 1
make: *** [Makefile:627: stamps/build-gcc-newlib-stage1] Error 2
```

**根因**：现代 GCC 源码树自带 `gcc/gmp/`、`gcc/mpfr/`、`gcc/mpc/`、`gcc/isl/` 子目录（一般通过 `contrib/download_prerequisites` 拉下来）。GCC 优先使用 in-tree 版本。

当你传 `--with-gmp=/opt/homebrew/opt/gmp` 时：
- 上层 GCC 用 brew 的 GMP
- 但 in-tree ISL 的 configure 看到 `--with-gmp-headers` 没指定，去标准路径找 `gmp.h`
- 标准路径找不到（brew 在 `/opt/homebrew/include`，不在默认 cpp include 路径）

**修复**：**不要给 GCC 传 `--with-gmp/--with-mpfr/--with-mpc/--with-isl`**，让它用 in-tree 源码自建一遍 gmp/mpfr/mpc/isl。

```bash
# 错误的做法
make GCC_EXTRA_CONFIGURE_FLAGS="--with-gmp=/opt/homebrew/opt/gmp ..."

# 正确的做法
make GCC_EXTRA_CONFIGURE_FLAGS="--disable-libcc1"   # 只传 GCC 真正需要的
```

GDB 没有 in-tree gmp/mpfr，必须传 brew 路径。

**检测**：
```bash
ls gcc/{gmp,mpfr,mpc,isl} 2>/dev/null   # 都存在则不能给 GCC 传 --with-gmp=
```

---

### 坑 #6：gawk 5.4 与 GCC 13.2 `opt-read.awk` SUBSEP 不兼容

**症状**：GCC stage1 编译时 `options.h` 全是 `#error` 行：
```
options.h:1:2: error: Empty option argument 'Type' during parsing of: Name(abi_type) Type(enum riscv_abi_type)
options.h:2:2: error: Empty option argument 'Type' during parsing of: ...
```

随后 `gencheck.cc` 编译 fail。

**根因**：GCC 用 awk 脚本 `opt-functions.awk` + `opt-read.awk` + `optc-gen.awk` 处理 `*.opt` 文件生成 `options.h` 和 `options.cc`。`opt-read.awk` 里 `FS=SUBSEP`（用 `\034` 作字段分隔）。

gawk 5.4.0（2024 年发布的最新版）改变了 SUBSEP 的边界行为，导致 `opt_args(name, flags)` 函数在抽取 `Type(...)` 内容时返回空字符串。

BSD awk（macOS 自带的 One True Awk）行为正常。

**复现**：
```bash
echo "Name(abi_type) Type(enum riscv_abi_type)" | gawk '
function opt_args(name, flags,  i, ch) {
  flags = " " flags
  sub(".* " name "\\(", "", flags)
  paren_count = 1
  for (i = 1; i <= length(flags); i++) {
    ch = substr(flags, i, 1)
    if (ch == "(") paren_count++
    else if (ch == ")") paren_count--
    if (paren_count == 0) break
  }
  return substr(flags, 1, i - 1)
}
{ print "Type arg:", opt_args("Type", $0) }'
# gawk 5.4: 输出 "Type arg: "  ← 空！
# bsd awk: 输出 "Type arg: enum riscv_abi_type"
```

**修复**：用 BSD awk 替代 gawk。但有两层覆盖要做：

1. **wrapper Makefile** (`Makefile.in:2`) 写死了 `AWK := @GAWK@`，并 `export AWK`。命令行 `make AWK=...` 可以覆盖：

```bash
make AWK=/usr/bin/awk ...
```

2. **wrapper 脚本** `scripts/wrapper/awk/awk` 由 configure 生成，硬编码 `${AWK:-/opt/homebrew/opt/gawk/libexec/gnubin/awk}`。这个 wrapper 被 `Makefile.in:55` 前置到 PATH，让所有 submodule configure 看到的 `awk` 都是这个 wrapper。修复：configure 后立刻覆盖：

```bash
cat > scripts/wrapper/awk/awk <<'EOF'
#!/bin/sh
exec /usr/bin/awk "$@"
EOF
chmod +x scripts/wrapper/awk/awk
```

**注意**：只调整 PATH 是没用的，因为 wrapper 脚本是按完整路径硬编码 gawk 的，PATH 里有没有 gawk 不影响。

**为什么 `export AWK=/usr/bin/awk` 不行**：
autoconf 的 `AC_PROG_AWK` 不读 env 变量（autoconf 2.69 的行为）。但 `AWK=/usr/bin/awk make` 这种命令行变量 make 会把它当作 override，并在 sub-make 时通过 MAKEFLAGS 传下去。

---

### 坑 #7：GCC `system.h` 让 `safe-ctype.h` 在 `<map>` 之前包含，macros 污染 libc++

**症状**：编 `riscv-selftests.cc` 时：
```
SDKROOT/usr/include/c++/v1/__locale:557:48: error: too many arguments provided to function-like macro invocation
  557 |     const char_type* toupper(char_type* __low, const char_type* __high) const
```

**根因**：

GCC `gcc/include/safe-ctype.h` 第 145-146 行：
```c
#undef toupper
#define toupper(c) do_not_use_toupper_with_safe_ctype
```

这把 `toupper` 定义成 1 参数宏。

GCC `gcc/gcc/system.h` 的 include 顺序：
```c
// Line 209
#include "safe-ctype.h"          // 定义 toupper/tolower 等为 1 参数宏

// Line 227 - 在 safe-ctype.h 之后
#ifdef INCLUDE_MAP
# include <map>                  // 引入 libc++ <__locale>
#endif
```

macOS libc++ `<__locale>` 里有成员函数：
```cpp
char_type* toupper(char_type* __low, const char_type* __high) const;
```

预处理器一看 `toupper(...)` 是个宏调用，把它和后面的参数 expand，但参数数量不对（1 个 vs 2 个），fail。

注意 system.h 第 197-204 行已经把 `<string>` 放在 `safe-ctype.h` **之前**，注释明确说"avoid GCC poisoning the ctype macros through safe-ctype.h"。可见这是已知问题，但 `<map>` 没被一起处理。

**为什么 Intel 版没踩**：旧版 macOS 的 libc++ 里 `<map>` 不会 transitively include `<locale>`。macOS 14+ libc++ 才引入这个传递依赖（因为加了 `<format>` / `boyer_moore_searcher` 等新 header）。

**修复**：把 `<map>`、`<list>`、`<algorithm>` 等 C++ 标准库 include 移到 `safe-ctype.h` 之前。

```python
# patch_gcc_system_h.py
import sys
path = "gcc/gcc/system.h"
src = open(path).read()

old_block = '''#ifdef __cplusplus
#if defined (INCLUDE_ALGORITHM) || !defined (HAVE_SWAP_IN_UTILITY)
# include <algorithm>
#endif
#ifdef INCLUDE_LIST
# include <list>
#endif
#ifdef INCLUDE_MAP
# include <map>
#endif
#ifdef INCLUDE_SET
# include <set>
#endif
#ifdef INCLUDE_VECTOR
# include <vector>
#endif
#ifdef INCLUDE_ARRAY
# include <array>
#endif
#ifdef INCLUDE_FUNCTIONAL
# include <functional>
#endif
# include <cstring>
# include <initializer_list>
# include <new>
# include <utility>
# include <type_traits>
#endif

'''

marker = '/* INCLUDE_MAP_BEFORE_SAFE_CTYPE_PATCHED */\n'
if marker in src:
    print("Already patched")
    sys.exit(0)

anchor = '#ifdef INCLUDE_STRING\n# include <string>\n#endif\n#endif\n'

src = src.replace(old_block, '', 1)
src = src.replace(anchor, anchor + marker + old_block, 1)

open(path, 'w').write(src)
print("Patched")
```

**为什么 `-D_LIBCPP_NO_ABI_TAG` 不够**：
有人会想用 `-D_LIBCPP_NO_ABI_TAG` 让 libc++ 不在 `_LIBCPP_INLINE_VISIBILITY` 宏里加 `__attribute__((__abi_tag__(...)))`。这能解决第一波 `'__abi_tag__' attribute only applies to...` 错误，但不能解决根本问题——后面还会有 `too many arguments provided` 这类预处理 error。必须做 header 顺序的修复。

---

### 坑 #8：libcc1 包含 `<locale>` 触发 GCC poison list 与 libc++ 冲突

**症状**：stage2 编 libcc1：
```
SDKROOT/usr/include/c++/v1/locale:2822:27: error: attempt to use a poisoned identifier
make[4]: *** [Makefile:608: libcp1plugin.lo] Error 1
make: *** [Makefile:731: stamps/build-gcc-newlib-stage2] Error 2
```

**根因**：GCC 在 host 端用 `#pragma GCC poison` 禁掉了一批不安全的 libc 函数（gets, mktemp, sprintf 等）。libcc1 是 GCC 的 C 编译插件，它的 .cc 文件 include 了 `<locale>`，而 macOS 14+ libc++ 的 `<locale>` 内部用了 GCC poison 列表里的 identifier。

`libcc1` 是 GDB 的 `compile code` 功能用的，跟交叉编译完全无关。

**修复**：`--disable-libcc1`

```bash
make GCC_EXTRA_CONFIGURE_FLAGS="--disable-libcc1" ...
```

**对功能的影响**：
- 失去：GDB 的 `compile code "..."` 和 `compile file foo.c` 两条命令（运行时编译注入）
- 这两条命令**只在 Linux 用户态调试**有用，对裸机 MCU 调试无意义
- 所有标准 GDB 调试功能（断点、单步、watch、reg/mem 查看、远程 OpenOCD）都不受影响

详见本指南末尾的"GDB 功能不缩水"小节。

---

## 5. 验证产物

### 必须通过的检查

```bash
PREFIX=/path/to/install-dir

# 5.1 关键二进制存在且 native arm64
file $PREFIX/bin/riscv32-unknown-elf-gcc
# 期望: Mach-O 64-bit executable arm64

# 5.2 版本号
$PREFIX/bin/riscv32-unknown-elf-gcc -dumpversion
# 期望: 13.2.0

$PREFIX/bin/riscv32-unknown-elf-gcc -dumpmachine
# 期望: riscv32-unknown-elf

# 5.3 multilib 集合（应该 12 个）
$PREFIX/bin/riscv32-unknown-elf-gcc --print-multi-lib | wc -l
# 期望: 12

# 5.4 简单编译测试（rv32imac/ilp32）
cat > hello.c <<'EOF'
int main(void) { volatile int x = 42; return x; }
EOF
$PREFIX/bin/riscv32-unknown-elf-gcc -march=rv32imac_zicsr_zifencei -mabi=ilp32 \
  -nostartfiles -nostdlib hello.c -o hello.elf
file hello.elf
# 期望: ELF 32-bit LSB executable, UCB RISC-V, RVC, soft-float ABI

# 5.5 newlib 链接（用 nano specs）
cat > full.c <<'EOF'
#include <stdio.h>
int main(void) { printf("Hello, RISC-V\n"); return 0; }
EOF
$PREFIX/bin/riscv32-unknown-elf-gcc -march=rv32imac_zicsr_zifencei -mabi=ilp32 \
  --specs=nano.specs --specs=nosys.specs full.c -o full.elf
# 应该链接成功（会有 _close/_write not implemented warnings，正常）

# 5.6 GDB 启动
$PREFIX/bin/riscv32-unknown-elf-gdb --version | head -1
# 期望: GNU gdb (GDB) 14.2

# 5.7 二进制无异常依赖
otool -L $PREFIX/bin/riscv32-unknown-elf-gcc
# 期望: 只看到 /usr/lib/lib{iconv,c++,System}.dylib
otool -L $PREFIX/bin/riscv32-unknown-elf-gdb
# 期望: 上面那些 + /opt/homebrew/opt/{python,gmp,mpfr,zstd}/lib/...
```

### 隔离验证（确认没污染系统）

```bash
# 5.8 Shell rc 未改
stat -f "%Sm %N" ~/.zshrc ~/.bashrc ~/.profile 2>/dev/null
# 所有 mtime 应早于会话开始时间

# 5.9 PATH 未持久污染
exec $SHELL -l   # 起新 login shell
echo "$PATH" | tr ':' '\n' | grep -i riscv && echo "WARNING: PATH polluted" || echo "OK"

# 5.10 系统目录未写
ls /usr/local/bin/ 2>/dev/null | grep riscv || echo "OK"
ls /usr/bin/ 2>/dev/null | grep riscv || echo "OK"
```

---

## 6. 使用工具链

### 不修改 PATH 的用法（推荐）

```bash
PREFIX=/path/to/install-dir

# 直接全路径调用
$PREFIX/bin/riscv32-unknown-elf-gcc -march=rv32imac_zicsr_zifencei -mabi=ilp32 \
  --specs=nano.specs --specs=nosys.specs \
  -Os -g -ffunction-sections -fdata-sections \
  -Wl,--gc-sections \
  main.c startup.s -T linker.ld -o firmware.elf
```

### 临时 PATH 用法

```bash
# 仅当前 shell 会话生效，关掉就没了
export PATH="$PREFIX/bin:$PATH"

riscv32-unknown-elf-gcc -march=... -mabi=...
```

### 在 CMake / Makefile 里指定

```cmake
# CMakeLists.txt
set(TOOLCHAIN_PREFIX /path/to/install-dir/bin/riscv32-unknown-elf-)
set(CMAKE_C_COMPILER   ${TOOLCHAIN_PREFIX}gcc)
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_PREFIX}g++)
set(CMAKE_OBJCOPY      ${TOOLCHAIN_PREFIX}objcopy)
set(CMAKE_ASM_COMPILER ${TOOLCHAIN_PREFIX}gcc)
```

```makefile
# Makefile
TOOLCHAIN := /path/to/install-dir/bin/riscv32-unknown-elf-
CC        := $(TOOLCHAIN)gcc
AR        := $(TOOLCHAIN)ar
OBJCOPY   := $(TOOLCHAIN)objcopy
```

### HPM SDK 集成

HPM SDK 通过环境变量 `$RISCV_GCC` 或 `$GNURISCV_TOOLCHAIN_PATH` 找工具链：

```bash
export GNURISCV_TOOLCHAIN_PATH=/path/to/install-dir
export RISCV_GCC=$GNURISCV_TOOLCHAIN_PATH/bin/riscv32-unknown-elf-gcc
```

或者在 `hpm_sdk/cmake/toolchains/riscv32-unknown-elf.cmake` 里直接改路径。

---

## 7. 卸载

工具链是一个独立目录，卸载就是删目录。**不会留任何系统痕迹**。

```bash
# 删除最终工具链
rm -rf /path/to/install-dir

# 清理仓库构建中间产物（可选，省 ~10G 磁盘）
cd /path/to/riscv-gnu-toolchain
rm -rf build-binutils-newlib \
       build-gcc-newlib-stage1 \
       build-gcc-newlib-stage2 \
       build-gdb-newlib \
       build-newlib \
       build-newlib-nano \
       build-tmp \
       build-logs \
       stamps
rm -f Makefile config.log config.status
```

Shell rc、PATH、系统目录都没被改过，无需"反向操作"。

如果做了 `gcc/gcc/system.h` 的 patch，恢复方法：
```bash
cd /path/to/riscv-gnu-toolchain/gcc
git checkout gcc/system.h
```

---

## 8. GDB 功能不缩水

`--disable-libcc1` 只关掉 GDB 的两条**运行时编译注入**命令，与嵌入式调试无关。

| GDB 功能 | 是否可用 |
|---|---|
| `break main` / `break file:42` / `break *0x80001000` | ✅ |
| `step` / `next` / `finish` / `until` / `stepi` | ✅ |
| `watch *(int*)0x40000000` / `rwatch` / `awatch` | ✅ |
| `backtrace` / `frame` / `up` / `down` | ✅ |
| `print x` / `print *ptr@10` | ✅ |
| `print func(a, b)`（调用目标函数） | ✅ |
| `x/16xw 0x40000000` | ✅ |
| `info registers` / `info reg mstatus mepc mhartid` | ✅ |
| `disassemble` / `x/20i $pc` | ✅ |
| `set var x=42` / `set *(int*)0x40000000 = 0xdeadbeef` | ✅ |
| `break foo if x>10` (条件断点) | ✅ |
| `hbreak`（硬件断点） | ✅ |
| `target remote :3333` (OpenOCD) | ✅ |
| Python scripting / TUI / `commands` 块 | ✅ |
| `compile code "..."` | ❌（用不到） |
| `compile file foo.c` | ❌（用不到） |

---

## 9. 常见后续问题

### Q1: hpm_sdk 编译时报 multilib 不匹配

```
cannot find suitable multilib: -march=rv32imac_zicsr_zifencei_xandes -mabi=ilp32
```

**原因**：HPM SDK 用了非标 ISA 扩展（如 Andes 的 `_xandes`）。本工具链不带这些。

**解决**：在 SDK 里改 `-march` 去掉私有扩展，或重新构建工具链加 `--with-multilib-generator=...rv32imac_xandes-ilp32...`。

### Q2: `_write/_read not implemented and will always fail`

这是 newlib nano 的预期行为。`--specs=nano.specs --specs=nosys.specs` 表示"用最小 stub"。你的 BSP 代码需要实现 `_write/_read/_close/_sbrk` 等，把 stdio 重定向到 UART/USB CDC：

```c
int _write(int fd, char *ptr, int len) {
    for (int i = 0; i < len; i++) {
        hpm_uart_send_byte(BOARD_DEBUG_UART, ptr[i]);
    }
    return len;
}
```

### Q3: 我以后想加新的 multilib 怎么办？

重新跑 configure + make 即可，时间约 11 分钟（M4）：

```bash
./build-aarch64.sh clean
# 编辑 build-aarch64.sh，改 MULTILIB_GEN 变量
./build-aarch64.sh build
```

### Q4: macOS 升级到 27 / Apple Clang 17 之后还能用吗？

工具链本身是静态产物，不会变。但**重建**时可能遇到新的 host 编译器兼容问题，类似本指南坑 #7 #8。届时按相同思路定位（错误堆栈、patch system.h、加 `-D_LIBCPP_NO_ABI_TAG` 等）。

### Q5: 能不能装到 `/usr/local` 或 `/opt`？

**强烈不推荐**：
- 工具链不该污染系统目录
- 升级/卸载麻烦
- macOS SIP 可能拒绝写入

建议放在用户家目录或项目目录下，纯净独立。

### Q6: 想要 LLVM/clang 后端？

加 `--enable-llvm` 重新 configure 即可（会多花 30-60 分钟编 LLVM）。但 HPM SDK 默认不用 clang，没必要。

---

## 10. 速查表

### 这次踩坑总账

| # | 问题描述 | 修复 |
|---|---|---|
| 1 | 默认 multilib 出 riscv64 前缀 | 显式 `--with-arch=rv32imac_zicsr_zifencei --with-abi=ilp32` |
| 2 | configure 不认 --with-gmp 等 | 这些参数挪到 `make GCC_EXTRA_...= / GDB_..._FLAGS_EXTRA=` |
| 3 | `pipefail + grep -qx` SIGPIPE 误判 | 先 `var=$(brew list)` 再多次 `grep <<< "$var"` |
| 4 | 默认 make 拉 GDB 进来，GDB 缺 GMP | `make GDB_TARGET_FLAGS_EXTRA="--with-gmp=..." GDB_NATIVE_FLAGS_EXTRA="..."` |
| 5 | 给 GCC 传 brew GMP，in-tree ISL 找不到 gmp.h | GCC 不传 `--with-gmp/mpfr/mpc/isl`，让它用 in-tree |
| 6 | gawk 5.4 SUBSEP 行为变化把 GCC opt-read 弄坏 | `make AWK=/usr/bin/awk` + 覆盖 `scripts/wrapper/awk/awk` 为 BSD awk |
| 7 | GCC `system.h` 让 `safe-ctype.h` 污染 libc++ 成员函数名 | python patch 把 `<map>` 等移到 `safe-ctype.h` 之前 |
| 8 | libcc1 `<locale>` 触发 GCC poison list | `--disable-libcc1` |

### Configure 关键参数（HPMicro 风味）

```
--target=riscv32-unknown-elf
--with-arch=rv32imac_zicsr_zifencei
--with-abi=ilp32
--with-tune=rocket
--with-isa-spec=20191213
--with-cmodel=medlow
--with-system-zlib
--enable-multilib
--with-multilib-generator=<12 项 generator 列表>
--with-newlib            (自动加，不显式)
--enable-tls             (自动加，不显式)
--enable-languages=c,c++ (自动加，不显式)
--disable-shared --disable-threads --disable-nls (自动加)
```

### Make 关键参数

```bash
make -j$(sysctl -n hw.ncpu) \
  AWK=/usr/bin/awk \
  GCC_EXTRA_CONFIGURE_FLAGS="--disable-libcc1" \
  GDB_TARGET_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp --with-mpfr=/opt/homebrew/opt/mpfr --with-mpc=/opt/homebrew/opt/libmpc --with-isl=/opt/homebrew/opt/isl" \
  GDB_NATIVE_FLAGS_EXTRA="--with-gmp=/opt/homebrew/opt/gmp --with-mpfr=/opt/homebrew/opt/mpfr --with-mpc=/opt/homebrew/opt/libmpc --with-isl=/opt/homebrew/opt/isl"
```

### Host 端 CXXFLAGS 兼容补丁

```
CXXFLAGS_FOR_BUILD="-O2 -isysroot $SDKROOT -stdlib=libc++ \
  -D_LIBCPP_NO_ABI_TAG \
  -Wno-error=register \
  -Wno-error=deprecated-declarations \
  -Wno-error=deprecated-non-prototype"
```

### 验证 4 件套

```bash
PREFIX/bin/riscv32-unknown-elf-gcc -dumpversion              # 13.2.0
PREFIX/bin/riscv32-unknown-elf-gcc -dumpmachine              # riscv32-unknown-elf
PREFIX/bin/riscv32-unknown-elf-gcc --print-multi-lib | wc -l # 12
file PREFIX/bin/riscv32-unknown-elf-gcc                       # Mach-O ... arm64
```

---

## 11. 参考资料

- riscv-gnu-toolchain 上游：https://github.com/riscv-collab/riscv-gnu-toolchain
- GCC build prerequisites：https://gcc.gnu.org/install/prerequisites.html
- macOS libc++ 头文件源码：`/Library/Developer/CommandLineTools/SDKs/MacOSX*.sdk/usr/include/c++/v1/`
- HPMicro 官方 SDK：https://github.com/hpmicro/hpm_sdk
- 已知 GCC + macOS 14 SDK 问题（社区）：搜 "gcc 13 macos sonoma `__locale` toupper"
- One True Awk（macOS 自带 awk）：https://github.com/onetrueawk/awk

---

## 12. 致谢

本指南由实际构建过程的 7 次失败迭代沉淀而来，每一条坑都有对应的 build-log 行号佐证。每个 fix 都不依赖任何"魔法"，可以照搬执行。

更新日期：2026-05-27
验证环境：macOS 26.4 (Build 25E246) / Apple M4 / Apple Clang 16.0.0 / Homebrew 5.1.14
