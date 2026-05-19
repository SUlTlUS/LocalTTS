# LocalTTS 新提交代码检查记录

检查日期：2026-05-19  
仓库：`SUlTlUS/LocalTTS`  
分支：`master`  
检查方式：GitHub 提交 diff 静态检查，未在本地执行编译或测试。

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

同时浏览了更早的提交标题，主要集中在 fuzz 修复、CI 修复、测试稳定性修复和安全边界修复。

## 3. 总体结论

整体看，新提交方向是合理的：

- `release.sh` 增加 `--minor` 参数，属于发布流程增强。
- `ThreadPool` 构造函数在部分线程创建失败时增加清理逻辑，修复方向正确。
- chunked decoder 不再使用 `strtoul` 解析 chunk-size，改为手动十六进制解析，能避免负号或加号导致的畸形 chunk-size 被接受。

暂未发现必须立即回滚的阻塞问题，但发现 1 个中等风险点和若干测试/兼容性建议。

## 4. 具体检查结果

### 4.1 `scripts/release.sh`：增加 `--minor` 强制小版本升级

改动摘要：

- 参数解析从只支持 `--run` 改成支持 `--run`、`--minor`。
- 当 `abidiff` 通过但传入 `--minor` 时，也会执行 minor bump。
- `gh run list` 改为使用 `--commit "$HEAD_SHA"` 过滤当前提交的 workflow runs。

检查结论：基本可接受。

注意点：

- 需要确认运行环境中的 GitHub CLI 支持 `gh run list --commit`。如果旧版本 `gh` 不支持该参数，发布脚本会直接失败。
- `gh run list` 默认返回数量可能有限，如果一个提交对应大量 workflow runs，建议显式加 `--limit`，避免漏判 CI 状态。
- 参数重复传入时不会报错，例如 `--minor --minor`，但这不是严重问题。

建议：

```bash
gh run list --commit "$HEAD_SHA" --limit 100 --json name,conclusion,headSha
```

### 4.2 `httplib.h` / `test_thread_pool.cc`：ThreadPool 构造失败清理

改动摘要：

- `ThreadPool` 构造函数创建基础线程时包了一层 `try/catch`。
- 如果中途 `std::thread` 构造失败，会：
  - 设置 `shutdown_ = true`
  - `cond_.notify_all()` 唤醒已创建 worker
  - join 已创建的 joinable 线程
  - 重新抛出异常
- 新增 POSIX 下通过 interpose `pthread_create` 模拟线程创建失败的单测。
- `test_thread_pool` 目标尝试移除 ASAN，避免 ASAN 的 `pthread_create` interceptor 与测试里的 interpose 冲突。

检查结论：主逻辑修复方向正确，能避免构造函数异常期间 `std::vector<std::thread>` 析构 joinable thread 导致 `std::terminate()`。

发现的风险：

#### 中风险：ASAN 移除逻辑不够完整

当前 Makefile 中使用：

```make
THREAD_POOL_CXXFLAGS := $(filter-out -fsanitize=address,$(CXXFLAGS))
```

这个写法只能移除单独的 `-fsanitize=address`。如果 CI 或开发者使用组合 sanitizer 参数，例如：

```bash
-fsanitize=address,undefined
```

那么 ASAN 仍然会保留，可能继续触发本提交说明里提到的 `pthread_create` interceptor 冲突。

建议处理方式：

- 最稳妥：将该 interpose 单测在 ASAN 构建下跳过，而不是尝试从通用 `CXXFLAGS` 中手动删除 ASAN。
- 或者单独定义一个明确的非 ASAN 测试目标，避免继承全局 sanitizer 参数。
- 如果继续用 Makefile 过滤，需要覆盖 `-fsanitize=address,undefined`、`-fsanitize=undefined,address` 等组合情况。

#### 低风险：`pthread_create` interpose 的平台脆弱性

测试里通过定义同名 `pthread_create` 并用 `dlsym(RTLD_NEXT, "pthread_create")` 找真实函数。Linux 下通常可行，但 macOS / 不同链接器行为可能更脆弱。

建议：

- 如果 macOS CI 出现不稳定，可以只在 Linux 启用该 reproducer。
- 或者为 Darwin 使用更标准的 interpose 方案。
- 对 `dlsym` 返回值增加保护，避免极端情况下空函数指针导致崩溃。

### 4.3 `httplib.h` / `test.cc`：chunked decoder 拒绝畸形 chunk-size

改动摘要：

- 不再使用 `std::strtoul` 解析 chunk-size。
- 改为手动解析十六进制数字：
  - 要求第一个字符必须是 HEXDIG
  - 每位解析前检查 `size_t` 溢出
  - 解析结束后只允许空白、`;` chunk-extension、行尾结束
- 新增负数 chunk-size 和前导 `+` 的测试。

检查结论：安全修复方向正确。它能阻止 `-2` 这类输入被 `strtoul` 当作超大无符号数接受。

建议补充测试：

- chunk-size 超过 `size_t` 上限时返回 400。
- chunk-size 正好等于 `SIZE_MAX` 边界值时行为明确。
- `4 ;ext=value` 这种 chunk-extension 前带空白的情况。
- 含非法字符的情况，例如 `4x`、`0x10`。

## 5. 建议优先级

### 必须尽快处理

暂无必须立即回滚或阻塞合并的问题。

### 建议尽快处理

1. 修正 `test_thread_pool` 的 ASAN 处理方式，避免组合 sanitizer 参数漏过滤。
2. 为 `gh run list --commit` 增加 GitHub CLI 版本要求或 fallback。
3. 给 release 脚本的 workflow run 查询加 `--limit`。

### 可后续优化

1. 补充 chunked decoder 的边界测试。
2. 如果这个仓库确实应该是 LocalTTS 项目，建议检查为什么内容显示为 `cpp-httplib`。
3. 新增/恢复 `handoff.md`，记录：
   - 上次检查的 commit
   - 本轮要检查的 commit 范围
   - 待办项
   - 已知风险
   - 验收标准

## 6. 建议的 handoff.md 模板

后续可以在仓库根目录添加：

```markdown
# handoff

## 当前检查点

- 上次已检查 commit：
- 本次待检查范围：
- 当前分支：

## 待办

- [ ] 

## 已知风险

- 

## 验收标准

- [ ] 能编译
- [ ] 关键测试通过
- [ ] 无明显回归
```

## 7. 最终判断

本轮新提交可以继续保留。建议优先修正 `test_thread_pool` 对 ASAN 的处理方式，因为当前写法可能无法覆盖组合 sanitizer 参数，导致本来想规避的 ASAN/interpose 冲突在某些 CI 或本地构建配置下仍然出现。
