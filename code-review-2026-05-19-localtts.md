# LocalTTS 新提交代码检查记录

检查日期：2026-05-19  
仓库：`SUlTlUS/LocalTTS`  
分支：`master`  
检查方式：GitHub 提交 diff + 当前文件静态检查，未在本地执行编译或测试。

## 1. handoff.md 读取结果

本次按要求优先读取 `handoff.md`，但在当前仓库 `master` 分支根目录没有找到：

- `handoff.md`：未找到
- `HANDOFF.md`：未找到
- 代码搜索关键词 `handoff / HANDOFF`：未找到匹配文件

因此本次无法依据 handoff 文件中的上次交接点、待办项或验收标准做精确对照。下面的检查范围改为“当前仓库最近提交中有实际代码 diff 的新提交”。

另外需要注意：当前 `LocalTTS` 仓库内容看起来是 `cpp-httplib` 项目代码，而不是典型的 TTS 应用代码。后续如果这是一个错误同步/错误 fork，建议先确认仓库内容来源。

## 2. 本次检查范围

重点查看了最近 3 个有实际改动的提交：

1. `e8e6528` — Add `--minor` flag to `release.sh` for forced minor bumps
2. `7d5082c` — Make `ThreadPool` ctor exception-safe on partial thread creation
3. `87d62db` — Reject malformed chunk-size in chunked decoder

同时复核了当前文件：

- `scripts/release.sh`
- `test/Makefile`
- `test/test_thread_pool.cc`
- `test/test.cc`
- `.github/workflows/test.yaml`
- `httplib.h`

## 3. 总体结论

本轮代码没有发现需要立即回滚的严重功能错误。核心修复方向是合理的：

- `ThreadPool` 构造函数中途创建线程失败时，现在会先通知并 join 已创建线程，再重新抛出异常，避免析构 joinable `std::thread` 导致 `std::terminate()`。
- chunked decoder 改为手写十六进制解析，能拒绝 `-2`、`+4` 等畸形 chunk-size。
- `release.sh` 的 `--minor` 参数设计合理，可以用于 ABI 不变但行为破坏兼容的发布。

但静态检查发现以下问题/风险：

1. **明确兼容性问题：`release.sh` 使用 `sed -i ''`，在 Linux GNU sed 下会失败。**
2. **中风险：`test_thread_pool` 的 ASAN 移除逻辑只过滤精确 `-fsanitize=address`，不能处理组合 sanitizer 参数。**
3. **中低风险：`release.sh` 的 `gh run list` 没有显式 `--limit`，理论上可能漏掉 workflow run。**
4. **低风险：`pthread_create` interpose 单测在 macOS/不同链接器下较脆弱。**
5. **测试覆盖不足：chunked decoder 缺少溢出、非法字符、chunk-extension 边界测试。**

## 4. 具体问题

### 4.1 `scripts/release.sh`：`sed -i ''` 不是跨平台写法

当前代码：

```bash
sed -i '' "s/#define CPPHTTPLIB_VERSION \"[^\"]*\"/#define CPPHTTPLIB_VERSION \"$NEW_VERSION\"/" httplib.h
sed -i '' "s/#define CPPHTTPLIB_VERSION_NUM \"0x[0-9a-fA-F]*\"/#define CPPHTTPLIB_VERSION_NUM \"$VERSION_HEX\"/" httplib.h
sed -i '' "s/^version = \"[^\"]*\"/version = \"$NEW_VERSION\"/" docs-src/config.toml
```

这是 macOS/BSD sed 的常见写法。Linux GNU sed 中 `-i ''` 会把空字符串当成脚本/文件参数处理，通常会失败。

影响：

- 如果维护者在 Linux 上执行 `./scripts/release.sh --run`，发布流程会在更新版本号阶段失败。
- GitHub Actions 常用 Ubuntu runner，因此如果未来把发布脚本放到 CI 上跑，也会踩这个问题。

建议修复：

```bash
sed_inplace() {
  if sed --version >/dev/null 2>&1; then
    sed -i "$@"
  else
    sed -i '' "$@"
  fi
}

sed_inplace "s/#define CPPHTTPLIB_VERSION \"[^\"]*\"/#define CPPHTTPLIB_VERSION \"$NEW_VERSION\"/" httplib.h
sed_inplace "s/#define CPPHTTPLIB_VERSION_NUM \"0x[0-9a-fA-F]*\"/#define CPPHTTPLIB_VERSION_NUM \"$VERSION_HEX\"/" httplib.h
sed_inplace "s/^version = \"[^\"]*\"/version = \"$NEW_VERSION\"/" docs-src/config.toml
```

或者使用 Python/Perl 做跨平台替换。

### 4.2 `scripts/release.sh`：`gh run list` 建议加 `--limit`

当前代码：

```bash
RUNS=$(gh run list --commit "$HEAD_SHA" --json name,conclusion,headSha)
```

问题：

- GitHub CLI 默认返回数量有限。
- 当前 workflow 有多个 matrix job，如果一个 commit 上 workflow run 数量很多，默认数量可能不足。
- 如果漏掉失败 run，可能误判可以发布。

建议：

```bash
RUNS=$(gh run list --commit "$HEAD_SHA" --limit 100 --json name,conclusion,headSha)
```

另外建议在 README 或脚本开头注明最低 GitHub CLI 版本，因为 `--commit` 依赖 GitHub CLI 对该参数的支持。

### 4.3 `test/Makefile`：ASAN 过滤逻辑不完整

当前代码：

```make
THREAD_POOL_CXXFLAGS := $(filter-out -fsanitize=address,$(CXXFLAGS))
```

当前项目默认 `CXXFLAGS` 包含 `-fsanitize=address`，所以这个过滤在默认情况下有效。

但如果开发者或 CI 使用组合参数，例如：

```bash
EXTRA_CXXFLAGS="-fsanitize=address,undefined"
```

该过滤不会移除 ASAN，仍可能触发提交说明里提到的 `pthread_create` interceptor 冲突。

建议：

- 更稳妥的做法是让 interpose reproducer 在 ASAN 环境下跳过。
- 或者给 `test_thread_pool` 设定独立的、明确无 ASAN 的编译参数，不继承可能带 sanitizer 的全局 `CXXFLAGS`。
- 如果继续过滤，至少补充处理 `-fsanitize=address,%`、`-fsanitize=%,address` 等组合形式。

### 4.4 `test/test_thread_pool.cc`：`dlsym(RTLD_NEXT, "pthread_create")` 未做空指针保护

当前代码定义了一个同名 `pthread_create`，内部通过：

```cpp
static fn_t real = reinterpret_cast<fn_t>(dlsym(RTLD_NEXT, "pthread_create"));
```

查找真实函数，然后直接调用：

```cpp
return real(thread, attr, start_routine, arg);
```

问题：

- 正常 Linux/glibc 下通常没问题。
- 但在链接器、运行时库或 macOS 环境行为变化时，如果 `dlsym` 返回空指针，会直接崩溃。

建议补充保护：

```cpp
if (!real) { return EAGAIN; }
```

或者在测试启动时 assert `real != nullptr`，让失败更清楚。

### 4.5 `httplib.h`：ThreadPool 修复本身基本正确

当前构造函数中已经在异常路径里：

- 设置 `shutdown_ = true`
- `cond_.notify_all()`
- join 已创建线程
- `throw` 重新抛出异常

该逻辑能解决“构造函数中途失败导致 joinable thread 析构触发 terminate”的问题。

暂未发现这里有明显死锁或资源泄漏问题。

### 4.6 `httplib.h`：chunked decoder 修复方向正确

当前解析逻辑：

- 要求第一个字符必须是十六进制数字。
- 循环解析十六进制数字。
- 使用 `chunk_len > (chunk_len_max >> 4)` 检测左移前溢出。
- 解析结束后只允许空白、`;`、行尾。

该逻辑能拒绝负数、加号、明显非法 chunk-size。整体方向正确。

建议补充测试：

- 超过 `size_t` 最大值的 chunk-size。
- `4x\r\n....`。
- `0x10\r\n....`。
- `4 ;ext=value\r\n....`。
- 最大边界值。

## 5. 建议修复优先级

### 高优先级

1. 修复 `release.sh` 的 `sed -i ''` 跨平台问题。

### 中优先级

1. 给 `gh run list` 增加 `--limit 100`。
2. 改进 `test_thread_pool` 的 ASAN 处理，避免组合 sanitizer 漏过滤。

### 低优先级

1. 给 `pthread_create` interpose 增加 `dlsym` 空指针保护。
2. 补充 chunked decoder 边界测试。
3. 如果仓库确实应该是 LocalTTS 项目，确认为什么当前内容是 `cpp-httplib`。

## 6. 最终判断

代码主体没有明显大问题，可以继续保留；但 `release.sh` 的 `sed -i ''` 是实际会影响 Linux 发布的兼容性问题，建议优先修。ThreadPool 和 chunked decoder 的主逻辑基本正确，主要问题集中在测试/脚本健壮性和边界覆盖。