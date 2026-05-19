# 下一阶段任务规划

日期：2026-05-19  
仓库：`SUlTlUS/LocalTTS`  
分支：`master`  
规划依据：尝试读取 `HANDOFF.md`，但当前仓库中未找到该文件；因此本计划基于当前仓库状态、最近提交内容以及 `code-review-2026-05-19-localtts.md` 中的静态检查结论制定。

## 0. 前提说明

### 0.1 HANDOFF.md 状态

本次已尝试读取：

- `HANDOFF.md`
- `handoff.md`
- 仓库搜索关键词：`HANDOFF`、`handoff`、`plan`、`todo`

结果：未找到交接文件。

因此下一阶段需要先补齐项目交接机制，避免后续任务范围、检查点、风险状态无法追踪。

### 0.2 仓库内容异常提示

当前仓库名为 `LocalTTS`，但仓库内容看起来是 `cpp-httplib` 项目代码，包括：

- `httplib.h`
- `test/Makefile`
- `test/test.cc`
- `scripts/release.sh`
- `.github/workflows/test.yaml`

下一阶段应确认这是有意将 `cpp-httplib` 作为 LocalTTS 的依赖/子项目维护，还是仓库内容同步错误。

## 1. 下一阶段总体目标

下一阶段目标分为三层：

1. **修复明确问题**：优先处理会实际影响发布或 CI 的脚本/测试问题。
2. **补强测试覆盖**：为最近的安全修复和线程池异常路径补充边界测试。
3. **补齐项目管理文件**：建立 `HANDOFF.md`，维护计划、风险、验证结果和下一轮交接点。

目标完成后，应能达到：

- 发布脚本可在 macOS 和 Linux 上执行。
- ThreadPool 测试在默认 ASAN 构建、非 ASAN 构建、Linux/macOS 环境中行为可预期。
- chunked decoder 对畸形 chunk-size 有更完整的回归测试。
- 仓库有明确的下一轮交接记录。

## 2. 第一阶段：补齐交接文件和确认项目边界

优先级：高  
建议分支：`plan/handoff-and-scope`  
预计改动类型：文档

### 2.1 新增 `HANDOFF.md`

目的：记录当前任务状态，作为后续每轮检查/开发的入口。

建议内容结构：

```markdown
# HANDOFF

## 当前状态

- 当前分支：master
- 最近检查日期：2026-05-19
- 最近检查报告：code-review-2026-05-19-localtts.md
- 当前主要风险：release.sh Linux sed 兼容问题、ThreadPool ASAN 过滤边界、chunked decoder 边界测试不足

## 已知问题

- [ ] release.sh 使用 macOS-only sed 写法
- [ ] gh run list 未加 --limit
- [ ] test_thread_pool ASAN 过滤不完整
- [ ] pthread_create interpose 缺少 dlsym 空指针保护
- [ ] chunked decoder 缺少更多边界测试

## 下一步

- [ ] 修复 release.sh 跨平台 sed
- [ ] 增强 ThreadPool 测试健壮性
- [ ] 补充 chunked decoder 边界测试
- [ ] 运行 Linux/macOS/Windows CI

## 验证记录

- 待补充

## 下一轮交接点

- 待补充
```

验收标准：

- 仓库根目录存在 `HANDOFF.md`。
- `HANDOFF.md` 能说明当前状态、已知问题、下一步任务和验证记录。
- 后续每次修复后同步更新该文件。

### 2.2 确认仓库定位

目的：确认 `LocalTTS` 仓库中出现 `cpp-httplib` 代码是否符合预期。

检查项：

- 查看 README、提交历史、远程来源，确认该仓库是否为 `cpp-httplib` fork。
- 如果这是 LocalTTS 项目的 HTTP 组件仓库，需要在 README 或 HANDOFF 里说明其角色。
- 如果这是错误仓库，需要暂停代码改动，先切换到真正的 LocalTTS 仓库。

验收标准：

- 在 `HANDOFF.md` 或 `README` 中明确说明仓库用途。
- 如果仓库用途不匹配，创建单独记录说明不要继续修改当前仓库业务代码。

## 3. 第二阶段：修复 release.sh 发布脚本问题

优先级：高  
建议分支：`fix/release-script-portability`  
预计改动文件：`scripts/release.sh`

### 3.1 修复 `sed -i ''` 跨平台问题

当前问题：

```bash
sed -i '' "s/.../.../" httplib.h
```

该写法适用于 macOS/BSD sed，但在 Linux GNU sed 下会失败。

推荐方案 A：封装跨平台函数。

```bash
sed_inplace() {
  if sed --version >/dev/null 2>&1; then
    sed -i "$@"
  else
    sed -i '' "$@"
  fi
}
```

然后替换为：

```bash
sed_inplace "s/#define CPPHTTPLIB_VERSION \"[^\"]*\"/#define CPPHTTPLIB_VERSION \"$NEW_VERSION\"/" httplib.h
sed_inplace "s/#define CPPHTTPLIB_VERSION_NUM \"0x[0-9a-fA-F]*\"/#define CPPHTTPLIB_VERSION_NUM \"$VERSION_HEX\"/" httplib.h
sed_inplace "s/^version = \"[^\"]*\"/version = \"$NEW_VERSION\"/" docs-src/config.toml
```

推荐方案 B：使用 Python 脚本替换，避免 sed 差异。

验收标准：

- macOS 上 `./scripts/release.sh` dry-run 不报错。
- Linux 上 `./scripts/release.sh` dry-run 不报错。
- 在临时工作区执行 `./scripts/release.sh --run` 前的替换逻辑能正确修改：
  - `CPPHTTPLIB_VERSION`
  - `CPPHTTPLIB_VERSION_NUM`
  - `docs-src/config.toml` 中的 `version`

### 3.2 给 `gh run list` 增加 `--limit`

当前代码：

```bash
RUNS=$(gh run list --commit "$HEAD_SHA" --json name,conclusion,headSha)
```

建议改为：

```bash
RUNS=$(gh run list --commit "$HEAD_SHA" --limit 100 --json name,conclusion,headSha)
```

验收标准：

- 工作流数量较多时不会因为默认分页限制漏查。
- CI 检查逻辑仍能正确区分普通失败和 `abidiff` 失败。

### 3.3 增加依赖检查

建议在脚本前面增加依赖检测：

- `git`
- `gh`
- `jq`
- `sed` 或 `python3`

示例：

```bash
require_cmd() {
  command -v "$1" >/dev/null 2>&1 || {
    echo "Error: required command not found: $1" >&2
    exit 1
  }
}

require_cmd git
require_cmd gh
require_cmd jq
```

验收标准：

- 缺少依赖时脚本能给出明确错误，而不是后续出现难懂失败。

## 4. 第三阶段：改进 ThreadPool 测试健壮性

优先级：中高  
建议分支：`fix/threadpool-test-sanitizer`  
预计改动文件：

- `test/Makefile`
- `test/test_thread_pool.cc`

### 4.1 明确非 ASAN 测试目标

当前问题：

```make
THREAD_POOL_CXXFLAGS := $(filter-out -fsanitize=address,$(CXXFLAGS))
```

它只能过滤精确的 `-fsanitize=address`，不能处理：

```bash
-fsanitize=address,undefined
-fsanitize=undefined,address
```

建议方案：

- 为 interpose reproducer 设置明确的非 ASAN 编译目标。
- 或拆分成：
  - 普通 ThreadPool 单测：允许 ASAN。
  - pthread_create interpose reproducer：强制非 ASAN。

可选目标设计：

```make
test_thread_pool : test_thread_pool.cc ../httplib.h Makefile
	$(CXX) -o $@ -I.. $(CXXFLAGS) test_thread_pool.cc ...

test_thread_pool_interpose : test_thread_pool.cc ../httplib.h Makefile
	$(CXX) -o $@ -I.. $(THREAD_POOL_NO_ASAN_CXXFLAGS) -DCPPHTTPLIB_ENABLE_THREADPOOL_INTERPOSE_TEST test_thread_pool.cc ...
```

验收标准：

- 默认 `make test_thread_pool && ./test_thread_pool` 可通过。
- 带组合 sanitizer 的构建不会误触发 interpose + ASAN 冲突。
- interpose reproducer 仍能在非 ASAN Linux 环境覆盖构造失败路径。

### 4.2 给 `dlsym` 增加保护

当前风险：

```cpp
static fn_t real = reinterpret_cast<fn_t>(dlsym(RTLD_NEXT, "pthread_create"));
return real(thread, attr, start_routine, arg);
```

建议：

```cpp
if (!real) {
  return EAGAIN;
}
```

或者使用 `ASSERT_NE`/初始化检测，让测试失败信息更明确。

验收标准：

- `dlsym` 失败时不会空指针调用崩溃。
- 测试失败信息可定位到 interpose 初始化失败。

### 4.3 平台策略

建议根据 CI 结果决定：

- Linux：保留 interpose reproducer。
- macOS：如果持续不稳定，可暂时禁用 interpose reproducer，仅保留普通 ThreadPool 单测。
- Windows：不运行 interpose reproducer。

验收标准：

- Linux/macOS/Windows CI 均通过。
- 平台差异在注释或 `HANDOFF.md` 中记录清楚。

## 5. 第四阶段：补充 chunked decoder 边界测试

优先级：中  
建议分支：`test/chunked-decoder-boundaries`  
预计改动文件：

- `test/test.cc`
- 必要时 `httplib.h`

### 5.1 增加非法字符测试

建议新增测试：

```cpp
TEST_F(ServerTest, RejectsChunkSizeWithInvalidSuffix) {
  expect_chunked_body_rejected(cli_, "4x\r\ndech\r\n0\r\n\r\n");
}

TEST_F(ServerTest, RejectsChunkSizeWithHexPrefix) {
  expect_chunked_body_rejected(cli_, "0x10\r\n0123456789abcdef\r\n0\r\n\r\n");
}
```

验收标准：

- 非 RFC 格式的 chunk-size 返回 `400 Bad Request`。

### 5.2 增加溢出测试

构造超过 `size_t` 上限的十六进制 chunk-size，例如大量 `F`：

```cpp
TEST_F(ServerTest, RejectsChunkSizeOverflow) {
  expect_chunked_body_rejected(cli_, "FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF\r\nAAAA\r\n0\r\n\r\n");
}
```

验收标准：

- 超大 chunk-size 不会造成大内存分配。
- 返回 `400 Bad Request`。

### 5.3 增加 chunk-extension 测试

合法扩展：

```cpp
TEST_F(ServerTest, AcceptsChunkSizeWithExtension) {
  // 需要写对应 helper，期望 200 OK。
}
```

可测试：

```text
4;foo=bar\r\ndech\r\n0\r\n\r\n
4 ;foo=bar\r\ndech\r\n0\r\n\r\n
```

验收标准：

- 合法 chunk-extension 不被误拒绝。
- 非法 chunk-size 仍被拒绝。

## 6. 第五阶段：运行验证矩阵

优先级：高  
建议分支：随各修复分支执行

### 6.1 本地验证

建议至少执行：

```bash
cd test
make test_thread_pool
./test_thread_pool
make test
LSAN_OPTIONS=suppressions=lsan_suppressions.txt ./test --gtest_filter='ServerTest.Rejects*ChunkSize*'
```

如果修复 release 脚本：

```bash
./scripts/release.sh
```

注意：默认 dry-run，不应修改文件。

### 6.2 CI 验证

需要关注：

- Ubuntu OpenSSL / MbedTLS / wolfSSL
- macOS OpenSSL / MbedTLS / wolfSSL
- Windows without SSL / with SSL / compiled
- 32-bit compile/test job
- fuzz_test
- ThreadPool test

验收标准：

- 所有必须通过的 CI job 通过。
- 如果 `abidiff` 失败，需要确认是否为预期 ABI 变化。
- 如果 release 脚本逻辑变化，至少在 Linux 和 macOS dry-run 通过。

## 7. 第六阶段：文档和交接收尾

优先级：中  
建议分支：随修复分支或单独文档分支

### 7.1 更新 `HANDOFF.md`

每完成一个阶段，需要更新：

- 已完成事项
- 新发现问题
- 验证命令和结果
- 下一步交接点

### 7.2 更新 `code-review-2026-05-19-localtts.md`

如果代码已修复，应把原检查报告里的问题状态改为：

- 已修复
- 暂缓
- 不处理及原因

### 7.3 必要时更新 README

如果确认 `LocalTTS` 仓库实际维护的是 `cpp-httplib`，建议在 README 或仓库描述中说明原因，避免误解。

## 8. 推荐执行顺序

1. 新增 `HANDOFF.md`，确认仓库定位。
2. 修复 `release.sh`：跨平台 sed、`gh run list --limit`、依赖检查。
3. 改进 `test_thread_pool`：拆分/明确非 ASAN interpose 测试，增加 `dlsym` 保护。
4. 补充 chunked decoder 边界测试。
5. 运行本地测试和 CI。
6. 更新 `HANDOFF.md` 与检查报告。

## 9. 风险清单

| 风险 | 影响 | 优先级 | 处理方式 |
|---|---|---:|---|
| `HANDOFF.md` 缺失 | 无法追踪任务交接 | 高 | 立即新增 |
| 仓库名与内容不匹配 | 可能改错仓库 | 高 | 先确认仓库定位 |
| `sed -i ''` Linux 不兼容 | 发布脚本在 Linux 失败 | 高 | 封装跨平台替换 |
| `gh run list` 未加 `--limit` | CI 状态可能漏判 | 中 | 增加 `--limit 100` |
| ASAN 过滤不完整 | ThreadPool 单测可能在组合 sanitizer 下崩溃 | 中 | 拆分非 ASAN reproducer |
| `dlsym` 缺少保护 | 极端环境测试崩溃 | 低 | 增加空指针保护 |
| chunked decoder 边界测试不足 | 安全修复回归难发现 | 中 | 补充边界用例 |

## 10. 完成标准

下一阶段完成时应满足：

- [ ] 根目录存在 `HANDOFF.md`，并记录当前状态。
- [ ] `release.sh` 可在 Linux/macOS dry-run。
- [ ] `gh run list` 查询不依赖默认返回数量。
- [ ] ThreadPool interpose 测试不会与 ASAN 组合配置冲突。
- [ ] `pthread_create` interpose 有清晰失败处理。
- [ ] chunked decoder 新增至少 4 个边界测试。
- [ ] 本地关键测试通过。
- [ ] CI 必要矩阵通过。
- [ ] 更新检查报告，关闭已修复风险项。
