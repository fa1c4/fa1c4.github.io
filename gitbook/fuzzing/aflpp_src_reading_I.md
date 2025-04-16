# AFL++ Source Code Reading I - History

AFL++ 源码阅读记录, 和 AFL 原版代码进行对比分析. 分多篇 blogs 进行, 首先从版本迭代更新的内容开始, 到 AFL++ 整体实现的模式, 再到每个具体组件的实现细节



## History

AFL 从 2.52b 开始重构代码, fuzzing 逻辑的实现出现较大的改动. 从 2019 年的 2.52 版本到 2025 年的 4.31 版本, 按顺序进行代码变迁总结. 重要版本迭代用 `<!>` 标出.

### 2.52b

早期版本

+ QEMU 从 2.3.0 升级至 2.10.0

+ `afl-showmap` 添加 `setsid` 支持. 
+ `fuzzer_stats` 记录目标模式 (持久化/QEMU 等) 
+ `afl-tmin` 支持 Ctrl-C 时保存部分结果. 
+ `afl-analyze` 支持十六进制偏移量输出. 



### <!> 2.52c

这个版本开始整合已有代码和优化方法

+ **社区补丁**

  - 新增 `-e EXTENSION` 命令行选项 (`afl-fuzz`) 

  - LAF-Intel 性能改进 (需手动启用) 

+ 新增 **AFLfast 功率调度** (默认仍为 AFL 原版) 

- 新增 `afl-system-config` 脚本 (一键优化系统性能) 
- **兼容性升级**
  - LLVM 支持 3.9 到 8.0 版本. 
  - QEMU 从 2.1 升级至 3.1 (基于 `andreafioraldi/afl`) 



### 2.53c

+ README 转换为 Markdown 格式 (README.md) 

- 新增 `unicorn_mode` 支持 (基于 domenukk 的补丁) 
- 新增 `libcompcov` (QEMU 模式的 LAF-Intel 实现) 
- 新增 `instrim` (更快的 LLVM 模式插桩, 但牺牲路径发现) 
- 新增 **MOpt 模式** (基于 `puppet-meteor/MOpt-AFL`) 
- LLVM/QEMU 模式修复和优化 (如 `AFL_LLVM_NEVER_ZERO` 计数器) 
- 新增白名单支持 (`AFL_LLVM_WHITELIST`) 
- 新增 Python 模块变异器支持 (需 Python 2.7-dev) 
- `afl-tmin` 性能提升 (集成 TriforceAFL 的 forkserver 补丁) 
- **功能增强**
  - 显示 CPU 核心编号 (状态屏幕中的 `{#}`) 
  - 新增时间 / 执行次数限制选项 (`-V time` 和 `-E execs`) 
  - 支持固定随机种子 (`-s seed`) 
  - 队列文件命名包含发现时间. 
- **兼容性**
  - 提升跨平台支持 (非 Intel / Linux 环境) 
  - LLVM 环境变量统一前缀 (`AFL_LLVM_LAF_...`) 



### <!> 2.54c

- **代码重构**
  - 统一代码结构 (`include/` 和 `src/` 目录) 
  - 拆分 `afl-fuzz` 逻辑到模块化文件 (如 forkserver、内存映射等) 
- **新特性**
  - 自动生成工具手册页 (man pages) 
  - 新增 `AFL_FORCE_UI` 强制显示 UI. 
  - 支持 LLVM 9 和 Android (需修改 Makefile) 
  - 修复 QEMU 在 Ubuntu 的构建问题 (感谢 floyd) 
- **功能增强**
  - 支持自定义变异库 (感谢 kyakdan) 
  - `afl-showmap` 新增 `-r` 选项 (显示真实的桶值) 
  - 改进 *BSD 兼容性



### <!> 2.54d - 2.57c

- **核心改进**
  - **QEMU 持久化模式** (详见 `qemu_mode/README.md`) 
  - 自定义变异库改为附加选项. 
  - 新增 `qemu_mode/unsigaction` 库 (过滤 `sigaction` 事件) 
- **新功能**
  - `afl-fuzz`: 
    - 新增 `-I` 选项 (发现崩溃时执行命令) 
    - 支持输入文件为 FIFO 或磁盘分区 (取消链接输入)
    - 新增变异文档功能 (`make document` 生成 `afl-fuzz-document`) 
  - **Wine 模式** (`-W` 选项支持 Win32 二进制文件) 
  - **ARM 支持** (QEMU/Unicorn 的 CompareCoverage) 
- **其他优化**
  - 代码重构 (拆分 `afl-fuzz` 逻辑到独立文件) 
  - 新增 `make help` 和自动化测试框架 (`make tests`) 
  - 改进 *BSD 支持 (如 FreeBSD 的 CPU 绑定) 



### 2.58c

+ **性能修复**
  - 回退取消链接输入文件的补丁 (避免 10% 性能损失)


+ **新增功能**

  - 添加性能测试脚本 (`test/test-performance.sh`) 

  - 重新引入 **GCC 插件模式** (支持白名单和持久化) 

  - 为 GCC 插件添加测试用例. 




### 2.59c

- **新支持模式**
  - QBDI 模式: 支持 Android 原生库模糊测试. 
  - Unicorn 模式: 升级至 `unicornafl` (基于 [vanhauser-thc/unicorn](https://github.com/vanhauser-thc/unicorn)) 
- **功能增强**
  - `afl-fuzz`: 
    - 可选 Radamsa 变异阶段 (`-R[R]`) 
    - 新增 `-u` 选项 (禁止取消链接输入文件) 
    - **支持 Python3** (自动检测) 
    - 新增 `AFL_DISABLE_TRIM` 禁用修剪阶段. 
    - 支持 DragonFly BSD 的 CPU 亲和性. 
  - **LLVM 模式**: 
    - 浮点数拆分通过 `AFL_LLVM_LAF_SPLIT_FLOATS` 配置. 
    - 支持 LLVM 10. 
  - **libtokencap/compcov**: 
    - 支持 *BSD/OSX/DragonFly. 
    - 挂钩常见库的 `*cmp` 函数. 
- **其他改进**
  - QEMU 模式: 新增 `AFL_QEMU_DISABLE_CACHE` 禁用缓存. 
  - 新增正则表达式字典 (`regex.dictionary`) 
  - 优化 Android 支持 (集成 Google AFL 的部分功能) 



### 2.59d | 2.60c

- **关键修复**
  - 修复 `afl-tmin` 在 2.53d 版本中引入的严重 bug. 
- **新增功能**
  - 为 `afl-cmin` 和 `afl-tmin` 添加测试用例 (`test/test.sh`) 
  - 新增 `./experimental/argv_fuzzing` (`ld_preload` 库) 
  - 集成 `preeny` 的 `desock_dup` 库 (`./experimental/socket_fuzzing`), 支持网络模糊测试. 
- **环境变量**
  - 新增 `AFL_AS_FORCE_INSTRUMENT` (支持 `retrorewrite` 项目) 
  - QEMU 模式下自动从 `AFL_PRELOAD` 设置 `QEMU_SET_ENV`. 



### <!> 2.61c

- **性能与兼容性**
  - 支持 GCC 10, 默认启用 `-march=native` 优化. 
  - 禁用内存安全检查以提升速度 (可通过 `config.h` 配置) 
- **afl-fuzz 改进**
  - 修复 MOpt 越界写入崩溃. 
  - 支持 **Python 2/3 全版本**. 
  - 新增 **RedQueen 输入到状态变异器** (基于比较指令) 
  - Android 优先绑定大核 CPU. 
- **LLVM 模式修复**
  - 修复 InsTrim 插桩的 3 个关键问题 (包括 `prev_loc` 写入错误) 
- **QEMU 模式增强**
  - 持久化模式支持 ARM / AArch64. 
  - 新增 **CmpLog 插桩** (x86/ARM 架构) 
  - 支持指令级入口点 (`AFL_ENTRYPOINT`) 
- **工具链优化**
  - `afl-cmin` 改用 POSIX `sh` 脚本提升兼容性 (原版保留为 `afl-cmin.bash`) 
  - `afl-showmap` 支持多输入并行处理 (加速 `afl-cmin`) 



### 2.62c

- **关键修复**
  - 修复内存分配函数导致的崩溃检测失效问题. 
- **CmpLog 优化**
  - 不再依赖 `sancov`. 
- **小改进**
  - 修复 `-E/-V` 选项的 CPU 释放问题. 



### <!!> 2.63c

- **项目重组**: 代码库迁移至 **AFLplusplus** 组织, 开发集中在 `dev` 分支. 
- **多线程与稳定性**
  - 重构代码使 `afl-fuzz` 线程安全, 为多线程模糊测试铺路. 
  - 减少内存分配次数, 提升性能. 
- **afl-fuzz 新特性**
  - Python 和自定义变异器接口统一 (API 变更) 
  - `AFL_AUTORESUME` 支持自动恢复会话 (无需 `-i -`) 
  - 新增实验性调度策略: 
    - `mmopt`: 忽略运行时, 侧重最近 5 个队列条目. 
    - `rare`: 聚焦稀有分支路径. 
- **LLVM 模式重大更新**
  - **LTO 无碰撞插桩** (需手动编译 LLVM 11) 
  - **NGRAM 覆盖率** (`AFL_LLVM_NGRAM_SIZE`) 
  - **上下文敏感分支覆盖** (`AFL_LLVM_INSTRUMENT=CTX`) 
  - **控制流完整性检查** (`AFL_USE_CFISAN`) 
  - 移除 `USE_TRACE_PC` 编译选项. 
- **QEMU 模式**
  - 使用内置 Capstone 修复现代 Linux 构建问题. 
  - x86 目标支持 CmpLog 记录函数参数. 
- **其他工具改进**
  - `afl-tmin` 支持挂起 (hang)最小化模式 (`-H`) 
  - 自定义 API 重写 (Python 和共享库接口统一) 



### 2.64c

- **LLVM LTO 模式增强**
  - 要求 LLVM 11+, 支持编译所有目标. 
  - 新增 **自动字典生成** (`AFL_LLVM_LTO_AUTODICTIONARY`) 
  - 支持 **可变共享内存映射大小** (仅 LTO 模式可用) 
- **afl-fuzz 改进**
  - 快照 (Snapshot)功能在 UI 中可见. 
  - `-L -1` 允许 MOpt 与常规变异并行运行 (同时支持字典、Radamsa 和 CmpLog) 
  - 修复 CmpLog/RedQueen 模式在 stdin 输入时的兼容性问题. 
- **QEMU 模式**
  - 修复持久化模式可能卡死或无法退出的问题. 
- **比较转换优化**
  - `AFL_LLVM_LAF_TRANSFORM_COMPARES` 支持静态全局/局部变量的比较转换. 
- **其他关键变更**
  - 新增 `AFL_MAP_SIZE` 环境变量指定共享内存大小. 
  - 修复 `AFL_CC/AFL_CXX` 为空时的编译问题 (原版 AFL 也存在此问题) 
  - 新增 `NO_PYTHON` 编译选项禁用 Python 支持. 



### 2.65c

- **afl-fuzz 改进**: 
  - 修复 `AFL_MAP_SIZE` 问题. 
  - 弃用 `AFL_POST_LIBRARY`, 推荐使用 `AFL_CUSTOM_MUTATOR_LIBRARY`. 
- **LLVM 模式**: 
  - 默认不跳过单块函数 (可通过 `AFL_LLVM_SKIPSINGLEBLOCK` 启用) 
  - 支持 LLVM 11 的固定地址共享内存 (提升速度) 
  - 新增 LTO 版本的 InsTrim (最快模式) 
  - 修复 LTO 模式下的边缘计数问题. 
- **新增示例**: 
  - `afl_network_proxy` (网络模糊测试)、`afl_untracer` (二进制内存修改)、`afl_proxy` (非标准目标插桩) 
- **其他**: 
  - 改进 32 位构建选项和错误报告. 



### <!> 2.66c

- **命名规范**: 
  - 主分支更名为 `stable`, 弃用 `master/slave` 和 `blacklist/whitelist`. 
- **afl-fuzz 改进**: 
  - 次级节点 (`-S`)仅从主节点同步以提高性能. 
  - 改用 xxh3 和 xoshiro256** 哈希算法 (性能提升 5.5%) 
  - 新增 `SEEK` 调度策略 (忽略运行时, 专注测试用例长度) 
- **LLVM 模式**: 
  - 默认插桩改为 PCGUARD (LLVM ≥ 7), 支持避免碰撞. 
  - 支持 LLVM 3.4+ (最低版本要求降低) 
  - 改进 LTO 的 `instrument_files` 功能 (支持通配符) 
- **其他**: 
  - Unicornafl 新增 PowerPC 支持和 Rust 绑定. 
  - 持久模式支持共享内存传递测试用例 (性能提升 10-100%) 
  - 修复 afl-cmin.bash 和 64 位架构支持. 



### <!> 2.67c

- **快照模块**: 支持改进的 AFL++ Snapshot LKM. 
- **afl-fuzz 改进**: 
  - 新增 `-F` 选项支持同步外部模糊器 (如 honggfuzz) 
  - 新增 `-b` 选项绑定到特定 CPU. 
  - 修复 CPU 亲和性竞争条件. 
  - 扩展 havoc 模式 (增加拼接和 MOpt) 
  - 修复 Redqueen 的字符串处理问题. 
- **LLVM 模式**: 
  - 支持 LLVM 12. 
  - 新增 `AFL_LLVM_ALLOWLIST/DENYLIST` (替代旧的白名单/黑名单) 
  - 改进 LTO 的稳定性和功能 (如自动字典、边缘 ID 记录) 
  - 修复 laf-intel 浮点分割问题. 
- **其他**: 
  - 新增 honggfuzz 变异器和 Frida 集成示例. 
  - 修复 afl-plot 和 afl-whatsup 的小问题. 



### 2.68c

- **新增功能**: 集成语法变异器 (Grammar-Mutator) 
- **afl-fuzz 改进**: 
  - 修复自动字典与 `-x` 字典的冲突. 
  - 新增 `AFL_MAX_DET_EXTRAS` 和 `AFL_FORKSRV_INIT_TMOUT` 环境变量. 
  - 修复 cmplog 的堆溢出问题. 
  - 记录模糊测试配置到 `out/fuzzer_setup`. 
- **自定义变异器**: 
  - 新增 `afl_custom_fuzz_count` 函数控制变异次数. 
- **LLVM 模式**: 
  - 支持 LLVM 12. 
  - 弃用 `LLVM_SKIPSINGLEBLOCK` 环境变量. 
  - 改进 LTO 和 SanCov 的插桩位置. 



### <!> 3.00c

- **代码重组**: 
  - 合并 `llvm_mode/` 和 `gcc_plugin/` 到 `instrumentation/`. 
  - 重命名 `examples/` 为 `utils/`, 并移动部分工具到该目录. 
- **工具改进**: 
  - 合并所有编译器到 `afl-cc`, 统一模拟之前的功能. 
  - 合并运行时对象文件 (`afl-llvm/gcc-rt.o` 到 `afl-compiler-rt.o`) 
- **afl-fuzz 改进**: 
  - 默认启用 `-S default` 模式. 
  - 确定性模糊测试 (`-D`)默认关闭, 主节点 (`-M`)仍默认启用. 
  - 新的种子选择算法 (基于性能评分), 旧模式可通过 `-Z` 启用. 
  - 默认调度策略改为 `FAST`. 
  - 默认禁用内存限制 (需通过 `-m` 手动设置) 
  - 支持 `rpc.statsd` 统计和图表 (Edznux 贡献) 
  - 支持从 `-i` 目录递归读取测试用例. 
  - 允许多次使用 `-x` 选项 (最多 4 次) 
  - 改进崩溃种子处理 (跳过但用于拼接) 
  - 新增 `NO_SPLICING` 编译选项和 `INTROSPECTION` 目标. 
- **插桩改进**: 
  - 增强的 GCC 插件 (AdaCore 贡献) 
  - 新增 `AFL_LLVM_DICT2FILE` 选项, 生成字符串比较字典. 
  - LTO 自动字典支持更多比较操作 (如 `std::string` 比较) 
- **自定义变异器**: 
  - 新增 `symcc` 和 `libfuzzer` 变异器. 
  - 改进语法变异器集成. 
  - 支持 Python 变异器的性能优化. 
- **其他**: 
  - 新增 `AFL_NO_COLOR` 环境变量禁用彩色输出. 
  - 修复 `-n` 模式 (dumb fuzzing)的问题. 



### 3.10c

- **平台支持**: 新增 **Mac OS ARM64** 和 **Android** 支持. 
- **afl-fuzz**: 
  - 动态学习目标共享内存大小 (`AFL_MAP_SIZE` 废弃) 
  - 升级 cmplog / Redqueen: 支持浮点数、字符串转换 (如大小写、十六进制) 
  - 新增 `-l` 强度选项 (推荐 `2`) 
  - 优化超时处理 (`-t +` 改为自动计算最大值) 
- **afl-cc**: 
  - 支持选择性插桩 (`__AFL_COVERAGE_*` 指令) 
  - 修复 LLVM 位转换和 LAF 功能的潜在崩溃. 
- **QEMU 模式**: 
  - 引入 **QASan** (QEMU 版 Address Sanitizer) 
- **其他**: 
  - Unicornafl 提升 Python / Rust 绑定性能. 
  - 新增 Rust 自定义变异器绑定. 



### 3.11c

- **afl-fuzz**: 
  - 优化共享内存大小自动检测. 
  - 修复 cmplog 的 off-by-one 溢出问题. 
  - 新增非 Unicode 字典条目变体. 
- **afl-cc**: 
  - 新增 `AFL_NOOPT` 绕过插桩 (用于复杂配置脚本) 
  - 修复 ASAN + CMPLOG 的 Unicode 处理问题. 
  - 重命名 `CTX` 为 `CALLER`, 改进上下文插桩. 
- **QEMU 模式**: 
  - 新增 `AFL_QEMU_EXCLUDE_RANGES` 排除内存范围 (@realmadsci 贡献) 
  - 支持 `NO_CHECKOUT=1` 跳过 Git 更新. 
- **工具**: 
  - `afl-cmin` 支持含空格的文件名. 



### 3.12c

- **afl-fuzz**: 
  - 新增 `AFL_TARGET_ENV` 传递额外环境变量 (如 `LD_LIBRARY_PATH`) 
  - 优化共享内存大小检测 (`AFL_MAP_SIZE` 多数情况下不再需要) 
- **afl-cc**: 
  - 修复 cmplog 的指针数据收集问题. 
  - 确保共享库正确插桩. 
  - 支持 LLVM PCGUARD 原生模式 (`NATIVE` 缩写) 
- **QEMU 模式**: 
  - 将 `AFL_PRELOAD` 和 `AFL_USE_QASAN` 逻辑移至 `afl-qemu-trace`. 
  - 新增 `AFL_QEMU_CUSTOM_BIN` 支持. 
- **安全**: 
  - 默认文件权限设为 `0600` (通过 `DEFAULT_PERMISSION` 配置) 



### <!> 3.13c

- **新功能**: 
  - 新增 **Frida 模式** (二进制目标模糊测试, 支持持久模式和 cmplog) 
  - 新增 **CodeQL 字典生成工具** (`utils/autodict_ql`) 
- **afl-fuzz**: 
  - 支持 `@@` 作为命令行参数 (如 `--infile=@@`) 
  - 新增 `AFL_PERSISTENT_RECORD` 记录不可复现的崩溃. 
  - cmplog 默认强度改为 `-l 2`, `-l 3` 启用全局 Redqueen. 
  - 新增 `AFL_EXIT_ON_SEED_ISSUES` 和 `AFL_EXIT_ON_TIME` 环境变量. 
- **afl-cc**: 
  - 最低 LLVM 版本要求提升至 6.0. 
  - 新增线程安全计数器 (`AFL_LLVM_THREADSAFE_INST`) 
  - 移除 InsTrim 插桩 (性能不如 PCGUARD) 
- **其他**: 
  - Unicornafl 修复 MIPS 延迟槽和 aarch64 退出地址问题. 
  - 更新语法变异器至最新版本. 
  - 新增 `AFL_PRINT_FILENAMES` 支持 (显示当前文件名) 



### 3.14c

- **afl-fuzz**: 
  - 修复 `-F` 参数包含 `/` 时的错误. 
  - 修复 cmplog 对极慢输入的崩溃问题. 
  - 移除 `-M` 主节点默认的 `-D` 确定性模式. 
  - 新增错误日志检查 (`out/default/error.txt`) 
  - 修复自定义变异器的修剪问题. 
- **afl-cc**: 
  - 优化 COMPCOV/laf-intel 插桩速度. 
  - 修复 C++ 全局命名空间函数的插桩问题. 
  - 支持 LLVM 3.8~5.0 版本. 
  - 修复部分字符串插桩的失败问题. 
- **Frida 模式**: 
  - 修复 cmplog 问题, 移除 `AFL_FRIDA_PERSISTENT_RETADDR_OFFSET` 需求. 
  - 提升 aarch64 与 x86 的功能对等性 (持久模式、cmplog、ASAN) 
- **工具改进**: 
  - `afl-cmin` 和 `afl-showmap` 支持递归读取子目录. 
  - `afl_analyze` 新增 forkserver 支持以提升性能. 
  - 新增 `AFL_NO_FORKSRV` 环境变量支持. 



### <!!> 4.00c

- **重大更新**: 
  - 文档全面重构 (Google Season of Docs 支持) 
  - 新增 **Nyx 模式** (全系统仿真快照) 
  - 新增 **coresight_mode** (ARM aarch64 二进制模糊测试) 
- **Unicorn 模式**: 
  - 升级至 Unicorn2 (性能提升, 支持 RISC-V) 
- **安全改进**: 
  - 禁止 `dlopen()` 后插桩库的覆盖冲突 (强制修复错误配置) 
- **afl-fuzz**: 
  - 修复 3.10c 引入的覆盖率检测回归. 
  - 更高效的 CMPLOG 模式. 
- **afl-cc**: 
  - 新增 TSAN (Thread Sanitizer)支持. 
  - 修复 LAF 字符串处理和分割开关的潜在崩溃. 
- **工具与脚本**: 
  - 新增 `optimin` 语料库最小化工具. 
  - 新增 `afl-persistent-config` 系统配置脚本. 



### 4.01c

- **afl-fuzz**: 
  - 新增 `-c 0` 复用主二进制进行 CMPLOG. 
  - 新增 `-g/-G` 设置输入最小/最大长度. 
  - 重新引入 `AFL_PERSISTENT` 和 `AFL_DEFER_FORKSRV` 支持非二进制持久模式. 
  - 修复 Mopt 算法选择、效应图计算等关键问题. 
- **afl-cc**: 
  - 适配 LLVM 15 的新 Pass 管理器 (LTO 暂不可用) 
  - 修复 **LLVM 15 兼容性**问题. 
- **新功能**: 
  - 新增 LibAFL 自定义变异器 (支持 Token 模糊测试) 
  - Frida 模式支持 C++ 异常处理 (`throw/catch`) 



### 4.02c

- **关键修复**: 修复 LLVM IR 向量选择导致的 PCGUARD 崩溃. 
- **GCC 插件**: CMPLOG 支持. 
- **LLVM 模式**: 修复 LAF 比较分割的更多类型支持. 
- **Frida 模式**: 新增 Android 支持. 



### 4.03c

- **构建改进**: 构建完成后显示成功/失败的组件摘要. 
- **afl-fuzz**: 
  - 新增 `AFL_NO_STARTUP_CALIBRATION` 跳过初始校准 (适合大型队列或 CI) 
  - 默认校准周期从 8 次降至 7 次, 变量队列项校准周期从 12 次降至 5 次. 
- **afl-cc**: 
  - 修复 PCGUARD 实现的 off-by-one 错误. 
  - 支持 LLVM 15 的 LTO. 
  - 新增 `AFL_DUMP_MAP_SIZE=1` 获取目标共享内存大小. 
- **QEMU 模式**: 
  - 新增 `AFL_QEMU_TRACK_UNSTABLE` 记录不稳定边缘地址 (需 `AFL_DEBUG=1`) 
- **工具修复**: 
  - 修复 `afl-analyze`. 
  - `afl-cmin` 新增 `-A` 选项保留崩溃/超时输入. 



### 4.04c

- **构建与脚本**: 
  - 修复 `gramatron` 和 `grammar_mutator` 的构建脚本. 
  - 增强 `afl-persistent-config` 和 `afl-system-config` 脚本. 
- **afl-fuzz**: 
  - 强制退出时写入所有统计信息. 
  - 确保退出时终止所有目标进程. 
  - 新增 `AFL_FORK_SERVER_KILL_SIGNAL`. 
- **afl-cc**: 
  - 支持 GCC 3.6+ 版本的 `afl-gcc-fast`. 
- **QEMU 模式**: 
  - 修复 4.03c 版本中的 10 倍性能下降. 
  - 新增 `qemu_mode/fastexit` 辅助库. 
- **Unicorn 模式**: 
  - 支持 Tricore 架构. 
  - 更新 Rust 绑定的 Capstone 版本. 
- **LLVM 模式**: 
  - 优先通过共享内存传递输入 (忽略命令行) 



### 4.05c

- **平台兼容性**: 
  - 现代 macOS 下 `libdislocator`、`libtokencap` 等工具需手动修复. 
- **afl-fuzz**: 
  - 新增 `afl_custom_fuzz_send` 自定义变异器功能 (支持通过 IPC 发送数据) 
  - cmplog 模式新增 `-l R` 选项 (随机着色) 
  - 启用 `INTROSPECTION` 时, 每 30 分钟写入队列统计到 `out/NAME/queue_data`. 
  - 新增环境变量 `AFL_FORK_SERVER_KILL_SIGNAL`. 
- **工具改进**: 
  - `afl-showmap`/`afl-cmin` 的 `-t none` 默认超时改为 120 秒. 
- **其他**: 
  - 更新 Unicorn 模式、Rust 自定义变异器依赖. 
  - 优化 Sanitizer 默认配置处理. 



### 4.06c

- **afl-fuzz**: 
  - 修复 "pizza 模式" (愚人节彩蛋)的崩溃问题. 
  - 新增 `AFL_NO_WARN_INSTABILITY` 忽略稳定性警告. 
- **afl-cc**: 
  - 支持 LLVM 16/17. 
  - 新增 CFI Sanitizer 支持. 
- **QEMU 模式**: 
  - 基本 RISC-V 支持. 
- **新功能**: 
  - **autotoken** 文本输入的无语法变异器. 



### 4.07c

- **afl-fuzz**: 
  - 重启时反向读取种子 (提升性能) 
  - 新增 `AFL_POST_PROCESS_KEEP_ORIGINAL` 保留原始数据. 
- **afl-cc**: 
  - 支持 LLVM 15+ 的 PCGUARD (需 LLVM 13+) 
  - 新增 `AFL_LLVM_LTO_SKIPINIT` 支持 WASM 项目. 
- **QEMU 模式**: 支持 PPC32 的持久模式 + QASAN. 
- **新变异器**: **atnwalk** (语法变异器) 



### <!> 4.08c

- **突变引擎**: 
  - 新策略: 优先发现路径的突变 (10 分钟后切换至触发崩溃的突变, `-P` 可配置) 
  - 新增 **独立命令行工具** (`custom_mutators/aflpp/standalone/`) 
- **afl-fuzz**: 
  - UI 显示运行状态. 
  - 支持禁用 CMPLOG (`-c -`) 
- **afl-cc**: 
  - 修复 LAF 有符号整数比较分割问题. 
- **新工具**: 
  - **TritonDSE** 和 **SymQEMU** 自定义变异器. 



### 4.09c

- **afl-fuzz**: 
  - 新增 `AFL_FINAL_SYNC` 强制终止前同步. 
  - CMPLOG 支持缩放 (`-l S`)和跳过无用插入 (`u8`) 
  - 修复字典解析的无限循环问题. 
- **afl-whatsup**: 
  - 显示启动中实例和覆盖率. 
  - 新增 `-m` (精简统计)和 `-n` (无颜色输出) 
- **工具链**: 
  - 新增 `afl-addseeds` 向运行中任务添加种子. 
  - 新增性能对比脚本 `benchmark/benchmark.py`. 



### <!> 4.10c

- **核心改进**: 
  - 默认调度策略改为 `EXPLORE` (修复 Fast 策略问题后更优) 
  - 新增 **注入模糊测试** (SQL/LDAP/XSS, 见 `README.injections.md`) 
- **afl-cc**: 
  - 大型代码重构. 
  - LTO 模式要求 LLVM 12+. 
- **插桩优化**: 
  - 支持 LLVM 18. 
  - 修复 LAF 浮点/整数分割问题 (LLVM 17 禁用整数分割) 
- **QEMU 模式**: 
  - 默认启用插件, 新增 **drcov 追踪**. 



### <!> 4.20c

- **重大变更**: 
  - 新 Forkserver 通信模型 (需重新编译 CMPLOG 目标) 
  - 支持 **40 亿覆盖边** (原 600 万) 
- **性能增强**: 
  - `make PERFORMANCE=1` 启用 CPU 专属优化 (跨平台兼容性降低) 
- **afl-fuzz**: 
  - 默认启用新确定性模糊测试 (`-z` 禁用) 
  - 修复 MOpt 与 `-M` 的冲突问题 (未来可能移除 MOpt) 
  - `-e` 选项扩展名支持保存队列项. 
- **afl-cc**: 
  - LTO 模式新增无冲突调用者插桩 (`AFL_LLVM_LTO_CALLER=1`) 
  - 修复 GCC 插件 CMPLOG 的 `std::string` 问题. 
- **工具改进**: 
  - `afl-whatsup` 显示平均速度和覆盖率. 



### 4.21c

- **性能优化**: 
  - 修复 `gettimeofday()` 到 `clock_gettime()` 的回归问题 (提升 5-10% 性能) 
  - 基于 2 CPU 年队列数据分析的新队列选择算法 (覆盖提升明显) 
- **afl-fuzz**: 
  - 新增 `AFL_DISABLE_REDREDUNDANT` (针对大型队列优化) 
  - 新增 `AFL_NO_SYNC` 禁用同步. 
  - 修复 `AFL_PERSISTENT_RECORD` 和文件名空格问题. 
  - 校准/修剪/同步时间单独统计. 
- **afl-cc**: 
  - 重新启用 i386 支持, 修复 LTO 和旧版 LLVM 兼容性问题. 
  - 禁用 XML/cURL 字符串转换函数 (空指针检查待实现) 
- **工具修复**: 
  - 修复 `afl-showmap` 共享内存泄漏. 
  - 修复 macOS 共享内存映射的罕见问题. 



### <!!> 4.30c

+ **重大变更**: **移除老旧编译器支持**: 移除 `afl-gcc` 和 `afl-clang` 功能, 全部集成到 `afl-cc`. 

+ **afl-fuzz 改进**: **快速恢复 (Fast Resume)** 

  - 新增 `-i -` 或 `AFL_AUTORESUME=1` 时, 若目标二进制未改动, 跳过校准阶段直接加载进度 (压缩存储, 需 zlib) 

  - 禁用选项: `AFL_NO_FASTRESUME=1`. 
  
  - **CMPLOG 兼容性破坏性更新**: 修复数学运算和未定义行为问题, 需**重新编译所有 CMPLOG 目标**. 
  
  - **自定义变异器增强**: 新增 `AFL_CUSTOM_MUTATOR_LATE_SEND=1`, 支持在目标重启后调用 `send()` 函数. 
  
  - **行为调整**: 改进 `AFL_EXIT_ON_TIME` 和 `AFL_EXIT_WHEN_DONE` 的逻辑 (后者现在会真正等待任务完成) 
  

+ **二进制模糊测试模式更新**
  - **Frida 模式**: 
    - `AFL_FRIDA_PERSISTENT_ADDR` 支持任意可达地址 (不限于函数入口) 
    - 统一调试标志: `AFL_DEBUG` 等效于 `AFL_FRIDA_VERBOSE`. 
  
  - **QEMU 模式**: 
    - 新增可选钩子支持 (见 `qemu_mode/hooking_bridge`) 
  
  - **Unicorn 模式**: 
    - 修复安装和 Forkserver 问题, 锁定 Unicorn 版本. 
  
  - **Nyx 模式**: 
    - 多项稳定性修复. 


+ **工具链与生态**
  - **自定义变异器**: 
    - 新增 `custom_send_tcp` 变异器, 支持 TCP 数据流模糊测试. 
    - 改进 `aflpp-standalone` 和 `autotokens-standalone` 工具. 
  
  - **afl-cc 增强**: 
    - 新增 `AFL_OLD_FORKSERVER` 运行时变量, 兼容旧版 Forkserver (用于 SymCC/SymQEMU 等工具) 
    - 新增 `AFL_OPT_LEVEL` 编译选项, 默认优化级别为 `3`. 
    - 修复 RedHat 系统中 LLVM 定义的兼容性问题. 
  
  - **开发者体验**: 
    - 代码格式升级至 LLVM 18 标准. 
    - AFL++ 头文件现在安装到 `$PREFIX/include/afl`. 




### 4.31c

+ **核心新增功能** **SAND 模式** (Sanitizer-Aware Non-Deterministic Fuzzing): 

  - 新增文档 `docs/SAND.md`, 优化了与 Sanitizer (如 ASAN、UBSAN)协同工作的效率. 

  - 适用于需要高效检测内存错误的场景. 

+ **afl-fuzz 改进**: **拼接 (Splicing)策略调整**

  - 默认禁用拼接 (研究显示其效果不佳), 需通过 `-u` 手动启用. 

  - 若连续两轮未发现新路径, 则自动启用拼接. 

- **兼容性提升**: 

  - 支持 Python 3.13+
  - 放宽 Android 和 iPhone 上的文件/共享内存权限. 

  - LLVM 20 支持: 再次适配 LLVM 20 的 API 变动 (开发者吐槽: LLVM API 频繁变更) 


- **Sanitizer 增强**: 
  - `-fsanitize=fuzzer` 时提前链接 `libAFLDriver.a`, 解决静态库中 `LLVMFuzzerTestOneInput` 的编译问题. 
  - 新增 `__sanitizer_weak_hook_*` 函数支持 (用于特殊场景) 
- **Bug 修复**: 修复多库加载后共享内存大小计算的错误. 





## Summary

总结从 2.52b 到 4.31c 版本的重要更新和代码变动

| Version       | Updates                                                      |
| ------------- | ------------------------------------------------------------ |
| 2.52c         | 新增 AFLfast 功率调度                                        |
| 2.54c         | 统一代码结构 (`include/` 和 `src/` 目录)  拆分 `afl-fuzz` 逻辑到模块化文件 (如 forkserver、内存映射等) <br />支持 LLVM-9, Android<br />支持自定义变异库 |
| 2.53c         | 新增 MOpt                                                    |
| 2.54d - 2.57c | Qemu 持久化模式<br />支持 Win32, ARM                         |
| 2.59c         | 支持 QBDI 模式 (Android native)<br />支持 Unicorn 模式       |
| 2.61c         | 支持全 python2 / 3<br />新增 Redqueen<br />新增 cmplog 插桩  |
| 2.63c         | **迁移到 AFL++**<br />重构 fuzzing 代码为多线程模式<br />新增 LTO 无碰撞插桩 <br />新增 NGRAM 覆盖率 (`AFL_LLVM_NGRAM_SIZE`)<br />新增 上下文敏感分支覆盖 (`AFL_LLVM_INSTRUMENT=CTX`)<br />新增 控制流完整性检查 (`AFL_USE_CFISAN`) |
| 2.67c         | 新增快照模式 AFL++ snapshot LKM<br />支持 LLVM-12            |
| 3.00c         | 统一所有编译器到 `afl-cc`<br />合并 `llvm_mode/` 和 `gcc_plugin/` 到 `instrumentation/`<br />默认调度策略改为 `FAST`. |
| 3.13c         | 新增 Frida 模式 (二进制目标模糊测试, 支持持久模式和 cmplog)  |
| 4.00c         | 文档全面重构 (Google Season of Docs 支持)<br />新增 Nyx 模式 (全系统仿真快照)  <br />新增 coresight_mode (ARM aarch64 二进制模糊测试)<br />支持 LLVM-15 |
| 4.10c         | 支持注入模糊测试 (SQL/XSS/LDAP)<br />支持 LLVM-18            |
| 4.20c         | 新 forkserver 模型, 支持 40 亿覆盖边 (原 600 万)<br />`make PERFORMANCE=1` 启用 CPU 专属优化 (跨平台兼容性降低) |
| 4.30c         | 移除 afl-gcc, afl-clang 等编译器, 统一使用 afl-cc 选择不同插桩方式<br />新增 `custom_send_tcp` 变异器, 支持 TCP 数据流模糊测试 |
| 4.31c         | 默认禁用拼接 splicing 策略, 需通过 `-u` 手动启用<br />支持 Python3.13+<br />支持 LLVM-20 |

 



## Reference

[1] https://github.com/AFLplusplus/AFLplusplus

[2] https://blog.ritsec.club/posts/afl-under-hood/

