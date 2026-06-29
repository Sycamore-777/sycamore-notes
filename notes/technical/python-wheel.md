# Python Wheel 完全指南

> Wheel 是 Python 的**预编译分发包格式**。它的核心价值：把「在用户机器上编译 C 扩展、解析依赖、运行 setup.py」这个过程，提前到**打包阶段**完成——用户拿到的是一个可以直接安装的二进制包。

---

## 先看地图：安装一个 Python 包的三种路径

```text
                    ┌── Wheel (.whl)        ← 预编译：解压即用，无需编译
pip install pkg ────┤
                    ├── sdist (.tar.gz)      ← 源码分发：需要本地编译 C 扩展
                    │
                    └── VCS / 本地路径       ← 直接从 git/本地安装（本质仍走 sdist 流程）
```

Wheel 要解决的核心痛点：**让安装变快、变可靠、变统一**。当你 `pip install numpy` 等了五分钟看它编译 C/Fortran 代码时，那就是因为没有匹配的 wheel 可用，pip 被迫从源码编译。

---

## 1. 什么是 Wheel

### 1.1 核心定义

**Wheel** 是 Python 的**预编译二进制分发包格式**，文件扩展名 `.whl`。它本质是一个 **ZIP 压缩包**，内部包含：

- 已经编译好的 C/C++/Fortran 扩展（`.so` / `.pyd`）
- Python 源代码文件（`.py`）
- 元数据（包名、版本、依赖、作者等）

安装 wheel 时，pip 做的事非常简单：**解压 ZIP → 把文件复制到 site-packages → 记录元数据**。不需要运行 `setup.py`，不需要调编译器。

```text
Wheel 安装过程（快，确定性）：
  .whl 文件 → 解压 ZIP → 复制到 site-packages → 完成

sdist 安装过程（慢，依赖编译环境）：
  .tar.gz → 解压 → 运行 setup.py → 检测编译器 → 编译 C 扩展 → 复制文件 → 完成
```

> Wheel 之于 Python，类似于 `.deb`/`.rpm` 之于 Linux，`.msi` 之于 Windows——都是「拿了就能装」的预编译格式。

---

### 1.2 为什么叫 "Wheel"

名字来自「奶酪店」（Cheese Shop，PyPI 的旧昵称）的概念——「轮子」是一种便于运输奶酪的容器。PEP 427 正式定义了 Wheel 格式。

这个命名也呼应了 Python 社区的俗语「don't reinvent the wheel」——Wheel 就是那个你不需要重新发明的「轮子」。

### 1.3 Wheel 要解决什么问题

在 Wheel 之前，Python 包分发的标准方式是 **sdist**（Source Distribution，源码分发），后缀 `.tar.gz` 或 `.zip`。安装时需要：

1. 解压源码
2. 运行 `setup.py install` 或 `setup.py develop`
3. 如果有 C 扩展，调用编译器（gcc/clang/msvc）编译
4. 复制文件到 site-packages

**sdist 的三个核心问题：**

| 问题 | 严重性 | 表现 |
|------|--------|------|
| **安装慢** | 高 | 每次安装都编译，numpy/scipy/pandas 等科学计算包编译 C/Fortran 代码耗时几分钟到几十分钟 |
| **环境依赖** | 高 | 需要安装正确的编译器、Python 开发头文件（`python3-dev`）、系统库（`libblas`、`libffi` 等）。缺一个就报错 |
| **不确定性** | 中 | `setup.py` 是**任意可执行代码**，可以读文件、联网、执行系统命令——安装一个包意味着运行不受控代码 |

> `setup.py` 的任意代码执行能力是一个安全问题：`pip install` 一个包就等于 `exec()` 它的 `setup.py`。Wheel 消除了这个问题——安装 wheel 只是解压 ZIP，不运行任何代码。

---

## 2. Wheel 文件格式

### 2.1 文件名规范

一个合法的 wheel 文件名遵循严格的命名规则：

```
{distribution}-{version}(-{build tag})?-{python tag}-{abi tag}-{platform tag}.whl
```

**真实例子：**

```text
numpy-2.1.0-cp312-cp312-manylinux_2_17_x86_64.whl
│       │     │     │     └────────── 平台标签 ──────────┘
│       │     │     └── ABI 标签
│       │     └── Python 标签
│       └── 版本号 (2.1.0)
└── 分发包名 (numpy)
```

**逐字段解析：**

| 字段 | 含义 | 常见值 |
|------|------|--------|
| `distribution` | 分发包名 | `numpy`, `requests`, `cryptography` |
| `version` | 符合 PEP 440 的版本号 | `2.1.0`, `1.0.0a1`, `2024.1.0` |
| `build tag` | 构建编号（可选，极少用） | `1`, `2` |
| `python tag` | 支持的 Python 版本 | `cp312`（CPython 3.12）, `py3`（任意 Python 3）, `py2.py3`（2/3 兼容） |
| `abi tag` | ABI 兼容性 | `cp312`（需要 CPython 3.12 ABI）, `abi3`（稳定 ABI，兼容多个版本）, `none`（纯 Python，无 ABI 依赖） |
| `platform tag` | 目标平台 | `manylinux_2_17_x86_64`, `win_amd64`, `macosx_10_9_x86_64`, `any`（纯 Python，跨平台） |

> **ABI（Application Binary Interface）** 标签决定了一个 wheel 能否在特定 Python 版本上运行。`abi3` 是 Python 3.2+ 引入的「稳定 ABI」——标 `abi3` 的 C 扩展可以跨 Python 3.x 小版本运行。`none` 表示没有 C 扩展，纯 Python 代码天然跨版本。

### 2.2 标签组合的含义

Wheel 的三种标签组合决定了它的**可移植性**：

```text
┌─────────────────────────────────────────────────────────┐
│ py3-none-any      纯 Python，跨所有平台和 Python 3.x 版本   │  ← 最通用
│ cp312-cp312-any   纯 Python，但限定 CPython 3.12           │
│ cp312-abi3-*     有 C 扩展，但用了稳定 ABI，可跨版本        │
│ cp312-cp312-*     有 C 扩展，绑定特定 Python 版本和平台      │  ← 最受限
└─────────────────────────────────────────────────────────┘
```

- **`any`** platform tag = 纯 Python，不包含编译扩展，直接 `.py` 文件，放到哪都能跑
- **`none`** ABI tag = 同样表示没有 C 扩展依赖
- 任何非 `none` 的 ABI tag = 一定包含 C 扩展，只能在该 Python 版本/平台上运行

---

### 2.3 内部结构

`.whl` 文件就是 ZIP 压缩包，可以直接用 `unzip` 查看：

```text
numpy-2.1.0-cp312-cp312-manylinux_2_17_x86_64.whl
│
├── numpy/                          ← 包的实际代码
│   ├── __init__.py
│   ├── core/
│   │   ├── multiarray.cpython-312-x86_64-linux-gnu.so  ← 编译好的 C 扩展
│   │   └── ...
│   └── ...
│
├── numpy-2.1.0.dist-info/          ← 元数据目录（PEP 427 规定）
│   ├── METADATA                    ← 包名、版本、作者、依赖、分类器
│   ├── WHEEL                       ← wheel 自身的元信息（版本、生成工具）
│   ├── RECORD                      ← 包内所有文件的列表 + SHA256 哈希
│   ├── entry_points.txt            ← 命令行入口点（如 pip 命令指向 pip._internal）
│   ├── top_level.txt               ← 顶层模块名
│   └── LICENSE                     ← 许可证（可选）
│
└── {distribution}-{version}.data/  ← 数据目录（可选）
    ├── scripts/                    ← 可执行脚本
    ├── headers/                    ← C 头文件
    ├── data/                       ← 数据文件
    └── platlib/                    ← 平台相关库
```

**关键文件说明：**

| 文件 | 作用 | 谁生成 |
|------|------|--------|
| `METADATA` | 包的标识信息（类似 `setup.py` 中定义的字段） | 构建工具（setuptools/hatchling/pdm-backend） |
| `WHEEL` | Wheel 包自身的版本和生成工具信息 | 构建工具 |
| `RECORD` | 包含每个文件的路径 + SHA256 + 大小。用于**卸载**（pip 靠它知道该删哪些文件） | `pip wheel` 或构建工具 |
| `entry_points.txt` | 定义命令行入口点（`console_scripts`），`pip install` 时会自动生成可执行脚本 | 构建工具 |

> `RECORD` 是 wheel 的卸载机制核心——pip 卸载一个包时，不靠「记住当时复制了哪些文件」，而是读取 `RECORD` 文件，逐个删除记录中的路径。这也是为什么 wheel 的安装/卸载是**确定性的**。

---

## 3. Wheel 的两种类型

### 3.1 纯 Python Wheel（Pure Python Wheel）

- 平台标签 = `any`
- ABI 标签 = `none`
- 不含 C 扩展
- 一个 wheel 可以用在**所有平台**（Linux/macOS/Windows）

```text
requests-2.31.0-py3-none-any.whl
flask-3.0.0-py3-none-any.whl
```

**这些包本质上就是一堆 `.py` 文件打成 ZIP**，没有任何平台相关的编译产物。

### 3.2 平台 Wheel（Platform Wheel）

- 平台标签不是 `any`（如 `manylinux_2_17_x86_64`、`win_amd64`）
- 包含编译好的 C/C++/Fortran 扩展（`.so`、`.pyd`）
- 每个平台需要**独立的 wheel**

```text
numpy-2.1.0-cp312-cp312-manylinux_2_17_x86_64.whl    ← Linux x86_64
numpy-2.1.0-cp312-cp312-win_amd64.whl                ← Windows x64
numpy-2.1.0-cp312-cp312-macosx_11_0_arm64.whl        ← macOS Apple Silicon
```

> 一个包含 C 扩展的包（如 numpy），要在 PyPI 上覆盖主流平台 + Python 版本，需要构建数十个不同的 wheel。这正是 CI/CD 和 `cibuildwheel` 这类工具的用武之地。

---

## 4. manylinux：让 Linux Wheel 可移植

### 4.1 为什么需要 manylinux

Linux 发行版之间 ABI 不兼容——在 Ubuntu 22.04 上编译的 `.so` 可能无法在 CentOS 7 上运行（glibc 版本不同、系统库路径不同）。如果每个发行版都要单独的 wheel，生态会爆炸。

**manylinux** 标准解决了这个问题：它定义了一个「最小公分母」构建环境。只要按照 manylinux 规范构建的 wheel，就可以在所有兼容的 Linux 发行版上运行。

### 4.2 manylinux 版本

| 标签 | 基于 | glibc 版本 | 覆盖范围 |
|------|------|-----------|----------|
| `manylinux1` | CentOS 5 | glibc 2.5+ | PEP 513，已过时 |
| `manylinux2010` | CentOS 6 | glibc 2.12+ | PEP 571 |
| `manylinux2014` | CentOS 7 | glibc 2.17+ | PEP 599 |
| `manylinux_2_17` | 多种发行版 | glibc 2.17+ | PEP 600（当前推荐） |
| `manylinux_2_28` | AlmaLinux 8 | glibc 2.28+ | 需要较新的系统 |
| `musllinux_1_1` | Alpine Linux | musl libc 1.1+ | PEP 656 |

`manylinux_2_17_x86_64` 的含义：基于 `manylinux_2_17` 规范，架构为 `x86_64`。

**工作原理**：manylinux Docker 镜像内置了旧版 glibc，构建工具将所有依赖的库**静态链接或打包进 wheel**，使得编译产物可以在任何 glibc ≥ 指定版本的 Linux 上运行。

---

## 5. 构建与安装

### 5.1 构建 Wheel

三种主流方式：

```bash
# 方式 1：pip wheel（生成 .whl 到当前目录）
pip wheel .

# 方式 2：build（官方推荐，PEP 517 通用前端）
pip install build
python -m build --wheel    # 只生成 wheel，不生成 sdist

# 方式 3：setuptools 直接调用（老旧，不推荐）
python setup.py bdist_wheel
```

**构建流程**（以 `build` 工具为例）：

```text
pyproject.toml 中指定 build-backend
        │
        ▼
build 调用后端（setuptools / hatchling / flit-core / pdm-backend）
        │
        ├── 编译 C 扩展（如果有）→ .so / .pyd
        ├── 收集 Python 源码（.py）和包数据
        ├── 生成 .dist-info 元数据目录
        └── 打包为 ZIP → 命名 → 输出 .whl
```

> PEP 517 将「构建前端」（如 `pip`、`build`）和「构建后端」（如 `setuptools`、`hatchling`）分离。你可以换后端而不用改 `pip install` 命令——这就是 `pyproject.toml` 的价值。

### 5.2 安装 Wheel

```bash
pip install package.whl     # 从本地文件安装
pip install numpy           # pip 自动从 PyPI 找匹配的 wheel
```

**pip 的 wheel 选择算法：**

```text
pip install numpy
    │
    ├─ 查询 PyPI：numpy 有哪些版本？每个版本有哪些 wheel？
    ├─ 检查当前环境：Python 版本？OS 和架构？glibc 版本？
    ├─ 根据标签匹配合适的 wheel：
    │   cp312-cp312-manylinux_2_17_x86_64.whl ← 选中！
    └─ 如果没有匹配的 wheel → 回退到 sdist → 本地编译
```

**安装过程**（极简）：

```text
1. 下载 .whl 文件
2. 解压 ZIP 到临时目录
3. 检查 METADATA 中的依赖 → 安装缺失的依赖（递归）
4. 复制包文件到 site-packages/<package-name>/
5. 复制 .dist-info 目录到 site-packages/
6. 如果有 entry_points.txt → 在 bin/ 或 Scripts/ 生成命令行入口
7. 删除临时目录
```

---

## 6. Wheel vs Egg vs sdist vs Conda

这四个概念都在「Python 包分发」语境中使用，但机制和定位完全不同。

| 维度 | Wheel (.whl) | sdist (.tar.gz) | Egg (.egg) | Conda 包 (.conda/.tar.bz2) |
|------|-------------|-----------------|------------|---------------------------|
| 格式 | ZIP 压缩包 | tar.gz 源码包 | ZIP 压缩包（可带编译文件） | 压缩包（含二进制 + 非 Python 依赖） |
| 内容 | 预编译 + .py 源码 | 纯源码 + setup.py | 混合（可能有 .so） | 二进制 + 所有依赖（含 C 库） |
| 安装速度 | 最快（解压复制） | 慢（需编译） | 快 | 快 |
| 跨平台 | 平台 wheel 不跨平台 | 跨平台（需编译环境） | 基本不跨平台 | 跨平台（每个平台独立构建） |
| 标准化程度 | PEP 427 正式标准 | 事实标准（无 PEP） | **已废弃**（setuptools 45+ 不再支持） | Conda 社区自有标准 |
| 非 Python 依赖 | 不能带（需 manylinux 静态链接或外部依赖） | 不能带 | 不能带 | 可以带（libffi, openssl, cuda 等） |
| 典型用途 | `pip install` 默认格式 | pip 回退、开发者分发 | 历史遗留 | 科学计算、跨语言依赖 |

### 6.1 Wheel vs Egg

**Egg 已死**。它是 setuptools 早期的二进制包格式，存在严重缺陷：

- 安装时需要运行 `setup.py`（安全风险）
- 元数据不规范，卸载机制不可靠（直接覆盖，容易残留文件）
- 不支持 PEP 517 构建系统抽象
- 2019 年 setuptools v45 正式移除 egg 安装支持

> 如果你还在用 `python setup.py install` 看到 `.egg` 文件——是时候迁移到 wheel 了。

### 6.2 Wheel vs sdist

`sdist` 不是 wheel 的「对立面」，而是**互补**关系：

- **Wheel**：给最终用户用的。安装快、不需要编译环境。
- **sdist**：给打包生态用的。没有源分发，就无法在**新平台**上从源码构建 wheel。它是 wheel 的「原材料」。

```text
源码仓库 (git)
    │
    ├── 构建 → sdist (.tar.gz) ← 存档/再分发，永远可以从源码重建
    │
    └── 从 sdist 构建 → Wheel (.whl) ← 最终用户直接安装
```

### 6.3 Wheel vs Conda

Conda 解决的问题和 wheel **有重叠但不相同**：

- Wheel：只管 Python 包本身。非 Python 依赖（如 BLAS、CUDA、OpenSSL）需要**操作系统层面**单独安装。
- Conda：管理 Python 包 **+** 所有非 Python 依赖。一个 `conda install pytorch` 可以同时装 CUDA toolkit、cuDNN、MKL 等。

> Wheel 的目标是「让 Python 包安装更快更可靠」。Conda 的目标是「让整个科学计算环境可复现」（包括非 Python 依赖）。对于纯 Python 项目，wheel 完全足够。对于深度学习/科学计算项目（CUDA、BLAS、FFTW 等），Conda 简化了非 Python 依赖的管理。

---

## 7. 常见问题与实践

### 7.1 如何检查一个 .whl 文件的内容

```bash
# 列出文件
unzip -l package.whl

# 查看元数据
unzip -p package.whl package-version.dist-info/METADATA

# 检查平台兼容性
python -c "import packaging.tags; print(list(packaging.tags.sys_tags()))"
```

### 7.2 「No matching distribution found」是什么意思？

```
ERROR: Could not find a version that satisfies the requirement numpy==2.1.0
ERROR: No matching distribution found for numpy==2.1.0
```

**原因**：PyPI 上没有匹配你当前环境标签的 wheel，也没有对应的 sdist。

常见子原因：
- Python 版本过新/过旧（如 CPython 3.13 刚出，包还没构建兼容 wheel）
- 平台太罕见（如 32 位 ARM Linux，主流包只构建 `x86_64` 和 `aarch64`）
- macOS Apple Silicon 上用了 `x86_64` 的 Python（需要 `arm64` wheel）

**解决**：回退版本、从源码编译（`pip install --no-binary :all: numpy`），或换一个兼容环境。

### 7.3 如何强制从源码编译

```bash
pip install --no-binary :all: numpy     # 不使用任何 wheel，强制编译 sdist
pip install --no-binary numpy numpy     # 只对 numpy 禁用 wheel
```

### 7.4 macOS 上的架构问题

Apple Silicon（M1/M2/M3）Mac 上，如果 Python 本身是 `x86_64`（通过 Rosetta 运行），pip 会尝试下载 `x86_64` wheel。如果该包只提供 `arm64` wheel，会回退到编译。

```bash
# 检查当前 Python 架构
python -c "import platform; print(platform.machine())"  # arm64 或 x86_64

# 检查 wheel 需要的架构
unzip -p numpy.whl '*.dist-info/WHEEL' | grep Tag
```

### 7.5 安全注意事项

- **wheel 安装不执行任意代码**（只解压 ZIP）。比 `setup.py` 安全得多。
- 但仍然可以**包含恶意 `.so` / `.pyd` 扩展**——Python 导入时这些二进制代码会执行。wheel 安全 ≠ 完全免于供应链攻击。
- 审计建议：`unzip -l` 检查 wheel 内容，看是否有可疑的 `.so` 文件。

---

## 常见误区

1. **「Wheel 是编译后的，不能跨平台」**：只有*平台 wheel* 不能跨平台。纯 Python wheel（`py3-none-any`）和源码一样跨平台。
2. **「pip install 总是装 wheel」**：如果 PyPI 上没有兼容 wheel，pip 会自动回退到 sdist 编译——这就是你看到「Building wheel for ...」时的情形。
3. **「Wheel 取代了 sdist」**：不是取代，是互补。sdist 是「源码原材料」，wheel 是「预编译成品」。没有 sdist 就无法在新平台上造 wheel。
4. **「manylinux wheel 能跑在所有 Linux 上」**：只能跑在 glibc 版本 ≥ manylinux 版本的 Linux 上。`manylinux_2_28` 的 wheel 在 CentOS 7（glibc 2.17）上跑不了。
5. **「Wheel 安装是纯文件复制，完全安全」**：二进制扩展（`.so`、`.pyd`）在导入时仍然执行代码。wheel 的安装过程安全 ≠ 运行时安全。
6. **「Conda 和 wheel 做同一件事」**：Conda 管理 Python + 非 Python 依赖（CUDA、BLAS、OpenSSL、系统库），wheel 只管理 Python 包本身。科学计算/深度学习领域 Conda 有独特价值。

---

## 小结

1. Wheel = Python 的**预编译分发包**，本质是 ZIP 文件，安装 = 解压 + 复制。
2. 文件名编码了**兼容性信息**：`{包名}-{版本}-{Python标签}-{ABI标签}-{平台标签}.whl`。
3. 纯 Python wheel（`py3-none-any`）跨所有平台；平台 wheel 绑定特定 Python 版本 + OS + 架构。
4. `manylinux` 标准让 Linux wheel 在不同发行版间可移植——通过老版本 glibc 构建 + 静态链接实现。
5. Wheel 不是 sdist 的替代品——它是 sdist 的「编译成品」。sdist 是原材料，wheel 是商品。
6. Egg 已死，Conda 解决的是 wheel 管不到的非 Python 依赖问题。
7. 安装安全 ≠ 运行时安全：wheel 安装不执行代码，但导入 `.so`/`.pyd` 仍然执行二进制代码。

---

## 自测

1. Wheel 和 sdist 的区别是什么？为什么 PyPI 上同一个包会有几十个不同的 `.whl` 文件？
2. 解释标签 `cp312-abi3-manylinux_2_17_x86_64` 中每个部分的含义，并说明这个 wheel 能在哪些环境安装。
3. `py3-none-any.whl` 的 `none` 和 `any` 分别是什么意思？为什么这样的包只有一个 `.whl` 文件？
4. pip 安装 wheel 时做了什么？卸载时又是怎么知道该删哪些文件的？
5. manylinux 解决了什么问题？`manylinux_2_17` 和 `manylinux_2_28` 有什么区别？
6. 你在 macOS Apple Silicon 上用 pip 装了一个包，报错 `No matching distribution found`。可能是什么原因？有几种解决方法？
7. Egg 格式有什么问题，为什么被废弃了？
8. Conda 和 wheel 各自解决什么不同的问题？什么场景下 Conda 明显优于 pip+wheel？

---

## 参考资料

- [PEP 427 — The Wheel Binary Package Format 1.0](https://peps.python.org/pep-0427/)
- [PEP 517 — A build-system independent format for source trees](https://peps.python.org/pep-0517/)
- [PEP 600 — Future 'manylinux' Platform Tags for Portable Linux Built Distributions](https://peps.python.org/pep-0600/)
- [PEP 656 — Platform Tag for Linux Distributions Using Musl](https://peps.python.org/pep-0656/)
- [Python Packaging Authority: Packaging Python Projects](https://packaging.python.org/tutorials/packaging-projects/)
- [cibuildwheel: Build wheels for all platforms in CI](https://cibuildwheel.pypa.io/)
- [PyPA: Binary Distribution Format (Wheel)](https://packaging.python.org/specifications/binary-distribution-format/)
