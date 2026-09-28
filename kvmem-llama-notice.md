# kvmem-llama 说明: 仓库结构, 引用反转, patch / merge 历史

本文档说明 `llama.cpp-kvmem` 与 `kvmem-llama.cpp` 两个仓库现在的关系, 为什么把引用方向反转,
以及 llama.cpp 侧 KVMem patch 的来源与合并历史。构建命令和运行参数在最后。
`kvmem-llama.cpp` 是独立的外部项目, 两个仓库各有自己的上游, 见 1.1 节 (最容易搞错的地方)。

日期: 2026-09-26; 1.1 节、5 节、7.1-7.2、7.11-7.14 与第 8 节在 2026-09-28 补过。当前分支: 本仓库 `kvmem`。

## 1. 现在的结构

```
llama.cpp-kvmem/            本仓库 = 打过 KVMem patch 的 llama.cpp (分支 kvmem)
  src/, common/, ggml/, ... 上游 llama.cpp 代码 + KVMem patch
  tools/server/CMakeLists.txt   llama-kvmem-server target (源码引用子模块)
  src/CMakeLists.txt            llama target 追加 adapter 源文件
  CMakeLists.txt                LLAMA_KVMEM / LLAMA_KVMEM_ROOT
  kvmem-llama.cpp/          子模块 = 外部项目 kvmem-llama.cpp 本体 (host 策略 + adapter + server/CLI 源码)
```

两个仓库各自编一部分, 最后链接成一个可执行文件:

| 产物 | 源码来自 | 说明 |
| --- | --- | --- |
| `llama.dll` | 本仓库 `src/` | 上游 llama.cpp + KVMem patch, 并额外编译子模块的 `src/adapter/*.cpp` (与 `llama-kvmem-stagein.cu`) |
| `llama-common.dll` | 本仓库 `common/` | 含 patch 改过的 `speculative.cpp` / `reasoning-budget.cpp` 等 |
| `mtmd.dll`, `server-context.lib` | 本仓库 `tools/mtmd/`, `tools/server/` | `server-common.cpp` 只编一份 |
| `kvmem.dll` | 子模块 `kvmem/src/host/*.cpp` | 纯 host 策略: block 表, 选择, tier, raw-K, runtime |
| `llama-kvmem-server.exe` | 子模块 `tools/llama-kvmem-server.cpp`, `kvmem-spec.cpp`, `kvmem-vision.cpp`, `kvmem-responses.cpp`, `kvmem-responses-stream.cpp` | 参数解析, chat/vision/responses 流程, prefill/retrieval 编排 |
| `kvmem_store_test` 等 | 子模块 `kvmem/tests/` | 无 GPU 单测 |

关键点: 改子模块里的文件, 本仓库的构建会直接编进去 (不存在第二份拷贝);
改 llama.cpp patch (例如 `src/llama-graph.cpp`) 只需要改本仓库一处。

## 1.1 两个上游, 别搞混 (2026-09-28)

**"同步上游" 说的是哪个上游, 取决于说的是哪个仓库**。两个仓库各有自己的 `origin`,
合并要在两个不同的地方做, 不要互串:

| 说的是 | 上游 `origin` | 在哪做 merge | 命令 |
| --- | --- | --- | --- |
| llama.cpp 上游 | 本仓库的 `origin` = `https://github.com/ggerganov/llama.cpp.git` | 本仓库, 分支 `kvmem` | `git fetch origin` + `git merge origin/master` |
| kVMem 上游 | 子模块自己的 `origin` = `https://github.com/kvmem/kvmem-llama.cpp.git` | 子模块目录 `kvmem-llama.cpp/`, 它自己的分支 `master` | `git -C kvmem-llama.cpp fetch origin` + 在子模块里 merge |

- `kvmem-llama.cpp` 是**独立存在的外部项目**。本仓库只是调整了它的结构 (删掉里面嵌套的
  `llama.cpp` 子模块, 引用方向反转, 见第 2 节) 并把它当子模块用; 它自己仍要跟自己的上游同步。
  改了结构不等于"上游变成 llama.cpp", 也不等于不用同步。
- 在本仓库里 "fetch 上游 + merge" 只会拿到 llama.cpp 的提交, 同步不到 kVMem 的代码;
  在子模块里 merge 也不会动本仓库的 patch。两件事要分别做。
- 顺序 (不能反, 因为子模块合并会移动它的 HEAD, 父仓库 pin 要在移动之后重新记一次):
  1. 在子模块 `kvmem-llama.cpp/` 里合并它的上游 (它的 pin 现状见 7.2 末尾);
  2. 回到本仓库, 按 7.2 的 cacheinfo 做法把新的子模块提交记进父仓库 pin;
  3. 最后才是本仓库合 llama.cpp 上游 (历史见第 6 节), 然后编译、运行、提交。

## 2. 引用反转 (2026-09-26)

**旧结构**: `kvmem-llama.cpp` 里嵌套一个 `llama.cpp` 引用, 它是一个 `git worktree`,
挂在 `temp/kvmem-llama.cpp/llama.cpp`, 指向本仓库 `kvmem` 分支的某个提交。子模块的
`CMakeLists.txt` 用 `add_subdirectory(llama.cpp)` 把补丁过的 llama.cpp 一起编,
所以 kvmem 仓库能独立出 `llama-kvmem-server.exe`。KVMem 源码本体放在被 `.gitignore`
忽略的 `temp/` 目录里。

**新结构**: 本仓库 `llama.cpp-kvmem` 用 submodule 引用 `kvmem-llama.cpp`。
子模块里那个嵌套的 `llama.cpp` 引用已删除 (`KVMEM_BUILD_LLAMA=ON` 不再可用),
llama.cpp 只由本仓库编译。

**为什么反转**:

- 旧结构下 `temp/` 被忽略, 本仓库无法记录"适配器用哪个版本", 也没有任何 commit 说明
  两者的配对关系; 换机器或重装就得靠 `temp/kvmem-merge-notes.md` 这种 scratch 文档。
- 旧结构里那份嵌套 llama.cpp 是本仓库的 worktree, 于是子模块永远显示 `M llama.cpp`
  (worktree 提交与 `.gitmodules` 记录的上游 pin 不一致), 状态脏且容易误判。
- 反转后本仓库一次 commit 就同时锁定"llama.cpp patch + 适配器提交", 结构单一。

**相关提交**:

| 仓库 | 提交 | 内容 |
| --- | --- | --- |
| 本仓库 | `c4a4101ff` | `kvmem : track kvmem-llama.cpp as a submodule` (新增 `.gitmodules` + gitlink, 修 `LLAMA_KVMEM_ROOT` 默认值与报错提示) |
| 子模块 | `487c2e8` | `kvmem : drop the nested llama.cpp submodule` |
| 子模块 | `8818ec7` | `kvmem : score prefill pressure with the newest prefilled user span` |

**迁移动作** (已执行, 备查):

```bat
:: 1. 子模块侧先提交 (功能 -> 删嵌套引用)
git -C kvmem-llama.cpp add ... && git -C kvmem-llama.cpp commit
git -C kvmem-llama.cpp rm llama.cpp && git -C kvmem-llama.cpp add .gitmodules && commit

:: 2. 把 kvmem 克隆从 ignored 的 temp/ 移到仓库根
Move-Item llama.cpp-kvmem\temp\kvmem-llama.cpp llama.cpp-kvmem\kvmem-llama.cpp

:: 3. 清掉指向 temp 的 worktree 记录, 再把已有仓库登记为子模块
git worktree prune
git submodule add https://github.com/kvmem/kvmem-llama.cpp.git kvmem-llama.cpp
:: "Adding existing repo at 'kvmem-llama.cpp' to the index"

:: 4. 本仓库配置项同步 (见第 4 节) 并提交
```

## 3. 构建

`LLAMA_KVMEM_ROOT` 现在默认指向 `${CMAKE_SOURCE_DIR}/kvmem-llama.cpp`, 所以**不需要**
再传 `-DLLAMA_KVMEM_ROOT=...` (旧命令里那个路径已经被移除, 传了反而会因路径不存在而报错):

```bat
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON -DGGML_NATIVE=ON ^
  -DCMAKE_CUDA_ARCHITECTURES=native -DGGML_CUDA_FA_ALL_QUANTS=ON -DLLAMA_KVMEM=ON

cmake --build . --config Release -j 5
```

- 只编 KVMem server / 单测: `cmake --build . --config Release --target llama-kvmem-server -j 5`
  或 `--target kvmem_store_test kvmem_runtime_test`。
- 换 KVMem 源码目录时才加 `-DLLAMA_KVMEM_ROOT=<dir>` (cache 变量, 改默认值不会影响已有 build 目录)。
- 报错提示: 找不到源码时会提示 `run 'git submodule update --init kvmem-llama.cpp' or set LLAMA_KVMEM_ROOT`。
- 新克隆的仓库要先 `git submodule update --init kvmem-llama.cpp` (见第 6 节的限制)。
- 已验证 (2026-09-26): 用上面的命令 (去掉 root 参数) 重编通过; 另外在一个干净目录只给
  `-DLLAMA_KVMEM=ON` (不传 CUDA) 也能配置、编译 `kvmem_store_test` 并 `OK`, 说明默认路径生效。
- 产物: `build/bin/Release/{llama-kvmem-server.exe, llama.dll, llama-common.dll, kvmem.dll, ...}`;
  `build/kvmem-from-llama/` 是子模块 kvmem 库在本仓库 build 目录下的 binary dir。
- 环境: CUDA 12.4 / MSVC 19.44 / CMake 4.4.3; 本机 `CMAKE_CUDA_ARCHITECTURES=native` (= V100 sm_70)。
- V100 16 GiB 的推荐模型与日常运行参数见第 8 节。

## 4. 运行 (llama-kvmem-server)

IQ3/IQ4 的既有配方 (16 GiB 卡) 与限制见子模块的 `README.md`、`docs/architecture.md`。
本例 (V100 16 GiB, 冒烟用) 的参数:

```bat
llama-kvmem-server.exe -m Qwen3.8-27B-GSQ-RCO-IQ3_XXS.gguf ^
  --host 127.0.0.1 --port 8080 --device CUDA0 ^
  -c 16384 --kvmem-budget 1024 --kvmem-gen-reserve 512 ^
  -ngl 99 -fa on -ctk q8_0 -ctv q4_0 -b 512 -ub 512 ^
  --kvmem-trace
```

与 prefill 判据相关的参数 (2026-09-26 新增/调整):

- `--kvmem-prefill-method recency|retrieval` 默认 **retrieval**。prefill 过程中超池换出时,
  用"已经 prefill 过的最后一个 user 消息"的 mean-Q 打分重选, 判据与 decode 侧一致:
  sink + mandatory + `--kvmem-recent-tokens` 后缀 + 分数。没有已 prefill 的 user 消息
  (system prompt 段) 时回落 recency。`recency` 是旧行为, 可回退。
- `--kvmem-prefill-query-max-tokens N` 默认 128: 每个 user span 只取尾部 N 个 token 做 query,
  控制 Q 的 D2H 与图重建开销。
- `--kvmem-recent-tokens N` 现在同时作用于 prefill 压力和 decode 选择 (以前压力路径忽略它)。
- `--kvmem-gen-exceed error|retrieval` 默认 **retrieval** (2026-09-26 新增)。decode 写满
  `--kvmem-gen-reserve` 时不再直接报 `no free GPU slot` 失败: 对整个池 (`budget + gen_reserve`)
  重选一次, query 块 + 本轮 incoming 行 + 正在写的尾块保持 mandatory。**本轮已写块不按规则先出**:
  它和历史块走同一个打分选择 (query mean-Q vs block mean-K, 同分时新块优先), `--kvmem-sink-tokens`
  前缀和 `--kvmem-recent-tokens` 后缀策略与普通 retrieval 完全一致, 只有落选块才先 harvest 再下到
  host store。select 窗口回到 `--kvmem-budget` 内, `gen_reserve` 区重新空出来继续 decode。需要
  `-fa on` (packed V); 关闭 FA 时回落旧的 error 行为。旧行为可用 `--kvmem-gen-exceed error` 保留,
  该模式下 server 仍把 `max_tokens` 夹到 `gen_reserve`; retrieval 模式下 generation 只受 `-c` 限制。
  省略 `max_tokens` 时的默认输出按模式取值: retrieval 是整个池 (`budget + gen_reserve`), error 是 `gen_reserve`。

诊断 (需要 `--kvmem-trace`):

```
KVMEM_TRACE prefill_spans method=1 n=2 first=[13,26) last=[3239,3247)
KVMEM_TRACE append ... need_offload=1 policy=retrieval q_rows=13 spans=2 free_slots=4
KVMEM_TRACE prefill_pressure stage_in=4 stage_out=4 skip=3 gpu_reused=4 window=1024
```

`policy=` 取值 `recency` / `retrieval` / `pinned` (retrieval 已 pin, decode 阶段不换出);
`skip=` 越大说明压力事件搬家越少, 也就是重排/重量化的次数越少 (漂移来源之一)。

`gen_exceed` 的换池行 (需要 `--kvmem-trace`):

```
KVMEM_TRACE gen_exceed policy=retrieval rows=3456..3457 resident=1152 mandatory=3 free_slots=0
[KVMEM_TRACE retrieval stage_in=... stage_out=... skip=... ]   :: 复用同一条 retrieval 计划
```

`resident` / `mandatory` / `free_slots` 是换出前的状态; `rows=a..b` 是本次 ubatch 占的行区间。
换池后返回 `budget` 内的窗口, `gen_reserve` 区空出来。

临时显存策略 (默认全开; 置 0 / 0MB 可回到旧的"一次分配"行为做 A/B):

| 开关 | 默认 | 作用 |
|---|---|---|
| `KVMEM_TEMP_BUDGET_MB` | 512 | 池上限 (`cap_blocks`) 里预留多少卡上显存给 harvest staging / layout scratch; 0 = 不预留 |
| `KVMEM_STAGING_TRIM` | 1 | 显存紧张时归还空闲的 D2H staging slot (layout 前, 以及"兄弟 slot 比本次需要更大"时); 0 = 一直缓存 |
| `KVMEM_LAYOUT_PRUNE` | 1 | layout 只 stage 会被别的块覆盖的源, 其余直接 D2D; 0 = 全量 gather (旧行为) |
| `KVMEM_LAYOUT_BOUNDED` | 1 | 大 scratch 分配失败时用 1 块 spare 轮转环/链; 0 = 回落 host roundtrip |
| `KVMEM_LAYOUT_SCRATCH_KB` | 0 | 强制 layout scratch 上限 (KiB); 0 = 按需要多少分配多少 |
| `KVMEM_Q_WINDOW` | 1 | Stage 5: harvest 只 stage 检索打分要读的 Q 行窗口 (query / prefill span); 0 = 全量拷 Q (旧行为) |

诊断行 (`KVMEM_TRACE=1`): `KVMEM_TRACE layout_scratch moves=.. staged=.. blocks=.. stride=..`,
`KVMEM stagein layout path=batched|bounded moves=.. staged=.. blocks=..`,
`KVMEM_STAGING_TRIM slot=.. freed_bytes=.. reason=layout|grow_retry`,
`KVMEM_CAPTURE_MEMORY` (两个 staging slot 的实际容量 + pinned 总量)。

## 5. 单测

```bat
cd build
cmake --build . --config Release --target kvmem_store_test pinned_kv_tier_test -j 5
build\bin\Release\kvmem_store_test.exe     :: 期望 OK
build\bin\Release\pinned_kv_tier_test.exe  :: 期望 OK
```

`kvmem_store_test` 覆盖 prefill 压力的 recency 旧契约、新的分数选择、`recent_blocks` 后缀钉住、
mandatory(incoming) 必须留在预算内, 以及压力预算独立于语义预算。

Windows 上只有 `kvmem_store_test` / `pinned_kv_tier_test` 有 target: 子模块 `kvmem/CMakeLists.txt`
把 `nvme_kv_tier_test` / `kvmem_runtime_test` / `raw_kv_store_test` 归为 POSIX-only (走 unistd.h 与
/tmp), 在 Windows 上直接跳过。`build/bin/Release/kvmem_runtime_test.exe` 是以前留下的产物,
不是本仓库现在能编出来的 target。

## 6. patch / merge 历史 (本仓库 `kvmem` 分支)

下表的 merge 都是把 **llama.cpp 上游** (`origin/master`) 合进本仓库; 子模块 `kvmem-llama.cpp`
跟自己上游的同步是另一件事, 不在这里记录 (见 1.1 节)。

| 提交 | 内容 |
| --- | --- |
| `b81c99b47` | KVMem 使用的上游 llama.cpp pin (2026-09-02, 当时落后 origin/master 402 个提交) |
| `be81c1b5a` | `kvmem : replay kvmem-llama.cpp v0.16.0-rc3 patch on pinned b81c99b`, 即把子模块的累计补丁 `patches/llama-kvmem-current.patch` 干净地应用到 pin 上 |
| `d39e9df12` | `Merge branch 'master' into kvmem` (上游新代码 + patch 的合并, 4 个冲突文件) |
| `13bca759b` | `kvmem : add the llama-kvmem-server target` (本仓库直接引用子模块源码, 复用 `server-context`) |
| `6b2c22f1c` | `kvmem : fix Windows link against the kvmem library` (`WINDOWS_EXPORT_ALL_SYMBOLS`) |
| `bb57e9065` | `kvmem : provision the chat UI for llama-kvmem-server` |
| `c4a4101ff` | `kvmem : track kvmem-llama.cpp as a submodule` (本文档的这次反转) |

合并时的 4 个冲突与处理方式:

1. `common/speculative.cpp`: 上游把 `common_speculative_draft_params::n_past` 改名为 `pos0`,
   patch 在其上加了 `n_past_logical` (multimodal 的 cache row 游标)。两边都保留:
   位置参数用 `dp.pos0`, 逻辑位置用 `dp.n_past_logical >= 0 ? dp.n_past_logical : dp.pos0`。
2. `src/llama-context.cpp`: 上游给 graph reuse 条件加了 `gf_res_prev_active == res`,
   patch 在同一条件里加了 `llama_kvmem_capture_can_reuse(...)`。合成一个条件, kvmem 调用
   仍包在 `#if defined(LLAMA_KVMEM)` 里。
3. `src/models/qwen35.cpp`: 上游把 `wq/wk/wv` 的 `build_lora_mm` 抽成 `build_qkv(...)`,
   patch 在此插入 `kvmem_capture_q/k/v`。采用上游 `build_qkv`, 保留 3 个 capture 调用
   (主 attn 与 MTP 两条路径)。
4. `tools/server/server-common.cpp`: patch 把媒体处理抽成 `oaicompat_chat_process_media()`,
   上游在同一函数里新增 schema 分支并改了 `handle_media()` 签名与 `video_url` 别名。
   保留抽出的函数 (内部用上游的媒体循环), `oaicompat_chat_params_parse()` 用上游版本 + 新增分支。

KVMem patch 对 llama.cpp 的主要改动面 (便于日后 rebase 时定位):

- memory factory hook (`src/llama-model.cpp` + `src/llama-kvmem-factory.h`)
- KV cell 扩展与 `seq_rm_logical` / skip-hole purge / SWA purge gate (`src/llama-kv-cache.cpp`,
  `llama-kv-cells.h`), `get_v_storage`
- batch 的 `logical_pos` / `embd_nextn` 透传 (`src/llama-batch.cpp/h`)
- capture 钩子 (`src/llama-graph.cpp/h` 的 `kvmem_capture_k/q/v`, `llama-context.cpp` 的图复用与 harvest)
- recurrent FP32 record/fold、GDN replay (`src/llama-memory-recurrent.*`, `src/models/delta-net-base.cpp`)
- CUDA `gated_delta_net` 与 `llama-kvmem-stagein.cu`
- `common/{speculative,reasoning-budget,sampling}.cpp`, `tools/mtmd/mtmd-helper.*`,
  `tools/server/server-common.*`

子模块侧的 patch 工作流: `patches/llama-kvmem-current.patch` 是累计补丁 (v0.16.0-rc3);
`patches/0001`-`0004` 是历史版本, 不要和累计补丁一起 apply;
`scripts/rebase-llama.sh` 负责把补丁在 pin 上重放, `scripts/apply-patches.sh` 用
`KVMEM_LLAMA_DIR` 指定目标 llama.cpp (以前指向那个 worktree, 现在应指向本仓库)。

## 7. 注意事项与未完成项

1. **子模块 pin 的来源**: 子模块 HEAD `04cba63` (2026-09-28 的 docs 提交; 之前是 `f860546`, 更早是 `e152e59` / `68fbc13`)
   已在 2026-09-28 推送到自己的 fork `https://github.com/kikoqiu/kvmem-llama.cpp` 的 `master`。
   这是 force push: 推送前那个 fork 的 `master` 是上游的 `b8ad6ded` (PR #81/#83 合完的状态),
   那些提交在上游仓库 (`kvmem/kvmem-llama.cpp`) 里仍然存在, 下次合上游会重新进入本地历史。
   父仓库 `.gitmodules` 的 url 还指向 `https://github.com/kvmem/kvmem-llama.cpp.git`, 那里没有这个提交,
   所以新克隆后 `git submodule update --init kvmem-llama.cpp` 会失败; 换机时要么把 url 改成 fork,
   要么用 `-DLLAMA_KVMEM_ROOT=<本地子模块目录>` 指过去。
2. 子模块的本地 `master` 领先它自己的 `origin/master` (数字见本项末尾的 "上游现状"); 并且本轮把它的 `.gitmodules`
   改成空文件 (原来记录 `llama.cpp` 子模块)。从上游 `git pull` 会重新带回那个条目,
   建议把这些提交放到自己的分支上维护, 而不是继续直接堆在 `master`。
   **注意**: `kvmem-llama.cpp/.git` 是嵌套的独立仓库 (不是标准的 `.git/modules/...` gitdir),
   所以主仓库自己看不到子模块 HEAD 的移动 - `git add kvmem-llama.cpp` 是空操作、
   `git status` 保持 clean、`git submodule status` 一直显示旧 pin。更新 pin 需要显式:
   `git update-index --cacheinfo 160000 <sha> kvmem-llama.cpp` 后再 commit
   (本轮即如此记录 `68fbc13`)。想根治可以跑 `git submodule absorbgitdirs kvmem-llama.cpp`,
   把 `.git` 收进 `.git/modules/`。
   **上游现状 (2026-09-28 合完)**: 子模块已 fetch 并把自己的 `origin/master` (`95a2155b`) 合进本地
   `master`, 结果是 merge 提交 `f860546` (合并前状态 `e152e59` 打了 tag `kvmem-sub-premerge-20260928`),
   父仓库 pin 已按上面的 cacheinfo 做法更新到 `f860546`。上一状态那 8 个本地提交现在是这个 merge 的
   第一个 parent, 仍在历史里; 这些提交已在 2026-09-28 随 fork 一起 push 出去 (见 7.1)。
3. 子模块不再自带 llama.cpp, 所以 kvmem 仓库不能再用 `KVMEM_BUILD_LLAMA=ON` 独立构建
   (要独立构建需自己恢复那个嵌套子模块)。
4. `temp/` 目录仍是 scratch, 不参与版本管理 (`temp/kvmem-merge-notes.md` 是旧记录,
   新的权威说明就是本文档; 冒烟脚本也放在 `temp/` 下, 未入库, 可复跑)。
5. 临时显存优化 A/B (2026-09-27, 未提交; 脚本与日志都在本地 `temp/` 下, 未入库):
   24k prompt + budget 20480 (触发 retrieval) + 27B IQ3 + `-ctk q8_0 -ctv q4_0` + ub 1024 上,
   默认组 / 四个开关全关组 / 强制 1 块 spare 组 / 强制 trim 组 的 `content_sha` 完全相同;
   staging 峰值 226 MiB + 64 MiB; layout scratch: 旧 60 块 -> 新 29 块 (stride 208 KiB),
   强制组 1 块且 `path=bounded` (layout 117 ms vs batched 113 ms), trim 组在 layout 前归还 290 MiB;
   `cap_blocks` 2498 -> 2341 (512 MiB 预留, 本配置池由 budget 决定所以池大小不变)。
   实现细节 + P2' (Q capture 归约, 本轮未做) 计划: `kvmem-llama.cpp/docs/retrieval-stagein-optimization.md`
   末尾的 "2026-09-27 - Temp VRAM bounds" 与 `prefill-harvest-optimization.md` 的 "Stage 5"。
   子模块提交 `df58ac9` (本地, 未 push), 父仓库 pin 已同步 (见 §7.1/§7.2 的 cacheinfo 做法)。
6. 验证状态 (2026-09-26): `kvmem_store_test` / `kvmem_runtime_test` = OK;
   27B IQ3 (V100 16 GiB, 小池 `--kvmem-budget 1024 --kvmem-gen-reserve 512`) 冒烟通过,
   默认 `prefill=retrieval`, 压力行形如 `policy=retrieval q_rows=13 spans=2`,
   回答正确。真实漂移收敛量需要在 27B + mmproj 的大池配置 (如
   `--kvmem-budget 60000 --kvmem-gen-reserve 20480 -c 262144`) 上按同一对话
   "KV 保留 vs KV 丢弃重 prefill" 对比。
7. gen_reserve 超限验证 (2026-09-26, 本地 scratch 脚本, 27B IQ3 V100):
   - `--kvmem-gen-exceed retrieval` + `--kvmem-gen-reserve 128` + `max_tokens 300`:
     `KVMEM_STARTUP ready` 里 `generation_limit=16384` (等于 `-c`), `default_max_tokens=128`;
     日志出现 `KVMEM_TRACE gen_exceed policy=retrieval rows=3456..3457 resident=1152
     mandatory=3 free_slots=0`, 随后 `stage_in/stage_out` 换池, 请求 `finish_reason=length`
     且输出 300 token (超过 reserve), 无 `no free GPU slot` / `llama_decode failed`。
   - `--kvmem-gen-exceed error`: `generation_limit=128`, 请求被夹到 `content_len=128`,
     日志无 `gen_exceed` 行, 旧行为保持。
   - 换池对象确认 (用本地 scratch 脚本解析同一个 log, `--kvmem-budget 1024
     --kvmem-gen-reserve 128`, `max_tokens 600`, prompt 3268 token -> 本轮从 block 26 起):
     ```
     --- swap 1 rows=3456..3457 ---
       kept:  history=[0 1 2 3 4 25] gen=[26 27]
       out:   history=[5 8]          gen=[]
     --- swap 2 rows=3712..3713 ---
       kept:  history=[0 1 2 25]     gen=[26 27 28 29]
       out:   history=[3 4]          gen=[]
     ```
     即本轮生成块与历史块同场评分, 这两次落选的都是历史块 (sink block 0 按 sink 策略始终保留)。
8. 回归修复 (2026-09-26): 上一版把 retrieval 模式的 `generation_limit` 提到 `-c` 之后, 请求不带
   `max_tokens` 时 `kvmem_output_limit()` 仍用 *上限* 兜底, 于是 `cr.max_tokens = n_ctx`,
   被 `toks.size() + cr.max_tokens > llama_n_ctx()` (`llama-kvmem-server.cpp:2315`) 挡成
   400 `prompt + max_tokens exceeds n_ctx` - 一条 "你好" 就会触发。两处改动:
   (a) 兜底改用 `default_max_tokens` (= `min(默认输出, 上限)`);
   (b) 该守卫改成上游 llama-server 的语义: 超出上下文时把 `max_tokens` 夹到 `n_ctx - prompt`
   并打 `max_tokens N with M prompt tokens exceeds n_ctx K; using F` 警告, 只有 prompt 本身
   超过 `n_ctx` 才 400 `prompt exceeds n_ctx` (UI 的"最大输出"上限取自 `generation_limit`,
   所以夹紧后 UI 填大值也能正常出结果)。
   小池复现 (`-c 8192 --kvmem-budget 2048 --kvmem-gen-reserve 1024`): `props` 显示
   `generation_limit=8192 default_max_tokens=1024`; 省略 `max_tokens` -> 200,
   `max_tokens=512` -> 200, `max_tokens=8192` -> 200 (夹到 8151); 用户原配置
   (`-c 262144 --kvmem-budget 50000 --kvmem-gen-reserve 20480 --api-key ...`, CUDA1 + mmproj + MTP)
   上"你好" -> 200 (本地 scratch 脚本)。`--kvmem-gen-exceed error` 行为不变
   (`default_max_tokens=128` 等于 reserve, 无换池行, 请求夹到 128)。
9. Stage 5 Q capture 窗口 (2026-09-27, 子模块提交 `30c4183`): harvest staging slot 原来按
   `K + Q` 定尺寸 (24k / `-ub 1024` 实测 226 MiB, K-only ubatch 是 64 MiB), 现在 `d2h_submit`
   只拷 `query_contains` / `prefill_query_contains` 接受的行窗口 (第一次到最后一次命中的行),
   `row0` 放在 `CaptureD2hPipe::Item`, `harvest_from_host` 从 `cur_pos_[row0 + i]` 开始归约。
   窗口是 commit 会接受的行集的超集, 归约的行顺序不变, 所以结果应逐位一致。窗口在 submit 时选、
   在 commit 时归约, 因此 `set_prefill_query_spans` 也补了 `harvest_flush()` (另外两个 span
   setter 本来就有)。`KVMEM_Q_WINDOW=0` 回到全量 staging 做 A/B; `KVMEM_TRACE harvest` 多一个
   `q_rows=`, `KVMEM_HARVEST_SUM` 有 `q_rows=` 合计。若实测 `q_rows` 接近 ubatch 大小
   (一个 ubatch 跨过很多 user span), 下一轮才考虑按 run 分别 staging 或设备侧归约。
   编译通过 (`llama-kvmem-server`), `kvmem_store_test` / `kvmem_runtime_test` OK; T0/T0s/T1/T2/T5
   门禁未跑, 需要按 `kvmem-llama.cpp/docs/prefill-harvest-optimization.md` 的 Stage 5 表在
   5050 / 5090 上执行。父仓库 pin 已同步 (做法见 §7.2)。
10. 默认输出改为整个池 (2026-09-27, 子模块提交 `e152e59`, 已随 7.1 的 fork push 出去): 上一版把 retrieval 的 `generation_limit` 提升到
   `-c`, 但请求省略 `max_tokens` 时默认输出仍回落到 `gen_reserve` (注释原文 "An omitted max_tokens keeps
   the recipe's reserve as the output default")。现在默认输出按模式取值: `--kvmem-gen-exceed retrieval` =
   `budget + gen_reserve` (整个池, 再被 `-c` 夹紧), `error` = `gen_reserve`, 未开 KVMem 或
   `--kvmem-budget 0` (identity) = `-c`; 显式 `-n` / `--n-predict` 优先级不变。
   两个启动脚本原来显式传 `-n <reserve>` (Windows `scripts/windows/start-server.ps1`,
   Linux/WSL `scripts/start-server.py`), 会盖住服务端默认值, 现在删除 (IQ3 池 53248 = 36864 + 16384,
   IQ4 45056 = 32768 + 12288), 让默认值生效。
   文档同步: README (`Generation length` 段 / `--kvmem-gen-exceed` 表项 / `-n` 段 / recipe 参数块)、
   `docs/architecture.md`「Generation length vs gen_reserve」、`docs/modification-plan.md`、
   `docs/recommended-config-performance.md`、`docs/user-feedback-triage.md`、`scripts/windows/README.md`。
   测试同步: `scripts/windows/test-launcher.ps1` 增加 "`-n` 必须缺席" 断言,
   `scripts/test_start_server.py` 把 `-n` 归入 "用服务端默认值" 的断言列表。
   编译通过 (`llama-kvmem-server`)。真机待验证: `/props` 里 `default_max_tokens` 在 retrieval 下应等于
   `min(-c, budget + gen_reserve)` (IQ3 配置 -> 53248), `--kvmem-gen-exceed error` 下等于 `gen_reserve`。
   父仓库 pin 已按 §7.2 的 cacheinfo 做法从 `30c4183` 更新到 `e152e59`。
11. 子模块合上游 (2026-09-28, merge 提交 `f860546`): 6 个冲突文件的处理方式:
    - `README.md` / `tools/kvmem-server-env.h` / `tools/kvmem-server-options.h`: 取并集 - 上游新增条目
      与本仓库的 `--kvmem-swap-ui` 等条目都保留 (env 别名表两边都在加), README 去掉重复的
      `--kvmem-gen-reserve` 表项。
    - `src/adapter/llama-memory-kvmem.{h,cpp}`: 上游带来多卡 batch D2H 路径 (`MultiD2hPipe` /
      `multi_d2h_submit`, 整 tensor 拷贝) 与 tensor-split 的 Meta device 枚举 (见 7.12), 本仓库有
      Q 行窗口 `row0`、bounded layout scratch 与 staging trim。合并后 `d2d_ok` 只在单 GPU tensor
      路径成立 (`n_res > 0 && !multi_gpu_`, 多卡走 host 回落), scratch 也只在那条路径分配;
      `harvest_from_host` 保留本仓库的 `row0` 参数, batch 路径整 tensor 拷贝所以 `row0` 保持 0
      (这个调用点在合并时漏了参数, 编译报 C2660, 已补 `c.row0`)。
    - `tools/llama-kvmem-server.cpp`: 保留本仓库的 swap-status / gdn checkpoint / diag 代码,
      接上游的同名函数重构 (中间被合并脚本弄坏过一次, 已手工修)。
12. 更新后的 patch 重放到本仓库 (2026-09-28, 5 个 llama.cpp 侧文件): 子模块的累计补丁
    (`patches/llama-kvmem-current.patch`) 改了 llama.cpp 侧, 本仓库要跟着改:
    - 新增 `ggml/include/ggml-backend.h` + `ggml/src/ggml-backend-meta.cpp`: 把上游的
      `static ggml_backend_meta_dev_n_devs` / `ggml_backend_meta_dev_simple_dev` 改名成公开的
      `ggml_backend_meta_device_count` / `ggml_backend_meta_device_get` (adapter 的 Meta device 枚举
      要调); 顺手修了构造函数里 `simple_devs` 被同名参数遮蔽的问题 (name/description 用 `this->`)。
    - 新增 `ggml/src/ggml-hip/CMakeLists.txt` 的两个 HIP 7.15 workaround: RDNA4 Q2_K MMQ unroll 上限
      与 Windows gfx1200 IQ3_XXS partial-register rewrite, 都是 option, 默认 ON, 只按 arch 生效。
    - `ggml/src/ggml-cuda/gated_delta_net.cu`: `gdn_fold_f32` 改成按 physical warp size (32/64) 取
      rows + `static_assert`, HIP 侧按设备 warp size 启动 (CUDA 侧仍 32x4)。
    - `src/CMakeLists.txt`: `find_package(CUDAToolkit)` / `CUDA::cudart` 包进 `if (GGML_CUDA)`。
    - 本仓库自己还要补 `tools/server/CMakeLists.txt`: 上游把 responses API 拆成新文件,
      llama-kvmem-server 的源文件表要加 `tools/kvmem-responses.cpp` 与
      `tools/kvmem-responses-stream.cpp` (不加就是 4 个 `kvmem_responses_*` 的 LNK2019)。
13. 本轮验证 (2026-09-28): `cmake --build . --config Release --target llama-kvmem-server` 0 error;
    `kvmem_store_test` / `pinned_kv_tier_test` = OK; 27B IQ3 (V100 16 GiB, `--kvmem-budget 1024
    --kvmem-gen-reserve 512`, 本地 scratch 脚本) 冒烟通过, 回答 `APPLE-11`, trace 仍是
    `policy=retrieval q_rows=13 spans=2`。
14. 本仓库 (llama.cpp 侧) 这一轮还没合上游: HEAD `b52ebb728` (tag `kvmem-premerge-20260928`) 落后
    `origin/master` (`136887b66`, 2026-09-27) 68 个提交, 上次合上游是 `d39e9df12` (2026-09-25)。
    下一步按 1.1 节的顺序做本仓库的 merge (冲突面见第 6 节), 合完重跑 13 的验证。

## 8. 实验 fork 的定位与推荐配置 (2026-09-28)

本 fork 的实验目标只有一个: 把 **V100 16 GiB (sm_70)** 压到极限。

- 性能优先: 小 context 下做到 **1000+ t/s prefill** 与 **60+ t/s decode**。
- 长 context: 用 KVMem 的有界 GPU KV 工作集 + host 存储, 在 16 GiB 上跑 **260k+ context**。
- remote:
  - 本仓库 (llama.cpp 侧 patch): `git@github.com:kikoqiu/llama.cpp-kvmem.git`, 分支 `kvmem`。
  - 子模块 (host 策略 + adapter + server): `https://github.com/kikoqiu/kvmem-llama.cpp`, 分支 `master`。
  - 上游: llama.cpp = 本仓库 `origin`; kVMem = 子模块自己的 `origin` (见 1.1 节)。

推荐编译 (CUDA, 在 build 目录里):

```bat
cmake .. -DCMAKE_BUILD_TYPE=Release -DGGML_CUDA=ON -DGGML_NATIVE=ON ^
  -DCMAKE_CUDA_ARCHITECTURES=native -DGGML_CUDA_FA_ALL_QUANTS=ON -DLLAMA_KVMEM=ON
cmake --build . --config Release -j 5
```

推荐模型: **`Qwen3.8-27B-GSQ-RCO-IQ3_XXS.gguf`** (视觉再配 `mmproj-Qwen3.8-27B-BF16.gguf`)。

推荐运行参数 (262144 context + mmproj + MTP, V100 16 GiB):

```bat
llama-kvmem-server.exe -m Qwen3.8-27B-GSQ-RCO-IQ3_XXS.gguf ^
  --mmproj mmproj-Qwen3.8-27B-BF16.gguf --no-mmproj-offload --image-min-tokens 1024 ^
  --device CUDA0 -c 262144 --kvmem-budget 60240 --kvmem-gen-reserve 10240 ^
  -ngl 99 -fa on -ctk q8_0 -ctv q4_0 --spec-type draft-mtp --spec-draft-n-max 2 ^
  -b 2048 -ub 1024 ^
  --enable-thinking --reasoning-budget 10240 --reasoning-effort low ^
  --top-k 20 --temperature 0.5 --repetition-penalty 1 ^
  --kvmem-swap-ui --kvmem-conversations 16 --kvmem-conversations-gb 16
```

- `--kvmem-budget 60240 --kvmem-gen-reserve 10240`: 块表池 `budget + gen_reserve` (128 token / 块),
  `-c 262144` 是 llama.cpp 侧的上下文上限; 4 个 KV dtype 走 `-ctk q8_0 -ctv q4_0` (需要 `-fa on`)。
- `--kvmem-conversations-gb 16`: `--kvmem-session-ram-gb` 的别名, host RAM 里的会话缓存上限;
  `--kvmem-conversations 16` 是会话条数上限。
- `--kvmem-swap-ui`: 打开 swap-status 页面; `--spec-type draft-mtp --spec-draft-n-max 2`: MTP 推测解码 (draft KV 为 F16)。
- `--repetition-penalty 1`: `llama-kvmem-server` 自己认这个别名 (`--repeat-penalty` 是上游的写法,
  upstream `common/arg.cpp` 里没有 `--repetition-penalty`, 只有 kvmem server 的 chat-sampling 路径认它)。
- `--reasoning-effort low` + `--reasoning-budget 10240`: 思考预算按 token 计, 想直接出答案可以调到更低。



