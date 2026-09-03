# KingCode 原生跑在鸿蒙 PC（HiShell，不经虚拟机）

把 [KingCode](https://github.com/Promethues0/kingcode)（基于 DeepSeek Harness 的编程智能体）
**原生**装进鸿蒙电脑：Node 是 Harmonybrew 装的 `platform=openharmony` 构建，跑在系统自带终端
HiShell 里，没有虚拟机、没有容器。

> 设备：HUAWEI MateBook 14 / HarmonyOS 7.0.0.102 / API 26 / HongMeng Kernel 1.13.0 / aarch64
> Node 26.8.1（`process.platform === 'openharmony'`，V8 非 lite、JIT 正常）
> dsh `0.1.2-alpha.3`（上游 alpha 通道）

## 验证到哪一步了

| 形态 | 状态 | 证据 |
|---|---|---|
| **CLI（A 级）** | ✅ 全通 | 整棵组合树 boot 成功；无钥烟测恰好死在 `MISSING_CREDENTIAL`；`npm test` 全绿；带钥 `say hi` 退出码 0；一个真用工具的任务里 **bash 工具经 node-pty 在原生内核上跑 `uname -a`**（回 `HongMeng Kernel 1.13.0`）、**grep 工具经 ripgrep** 数出 README 26 行 |
| **Web（B 级）** | 🟡 引擎与界面全通，差一条真回答 | 全局 dsh 原生 boot；`GET /` 401 → `?token=` 303 换 cookie → 200；`/api` 到网关；鸿蒙 ArkWeb 壳连 `127.0.0.1:3081` **加载出完整工作区**（品牌层、侧栏、KingCode 预设、设置→模型页显示「API 密钥已配置」）。**尚未**在 Web 里拿到一次真回答——复验当天设备出网的 TLS 层断了（见「已知环境坑」第 3 条），不是引擎问题 |
| 上架应用市场（C 级） | ❌ 未做 | 应用沙箱（normal_hap 域）有 neverallow 限制，是另一条路 |

## 三步装完

```sh
git clone https://github.com/Promethues0/kingcode-deepseekharness-harmonyos.git ~/kc-hmos
git clone https://github.com/Promethues0/kingcode.git ~/kingcode
ln -sf ~/kc-hmos/scripts/kc-hmos ~/.harmonybrew/bin/kc-hmos

kc-hmos deps          # Harmonybrew 依赖（ohos-sdk 2.7 GB，慢）
kc-hmos install-cli   # A 级：仓库依赖 + 现编 node-pty + 打补丁
kc-hmos smoke         # 判据：恰好死在 MISSING_CREDENTIAL

kc-hmos install-web   # B 级：全局 dsh + sharp WASM + 补丁 + profile/preset
kc-hmos start         # 起 Web 引擎
kc-hmos url           # 打印带 token 的地址，浏览器/鸿蒙壳里首次用它
```

带钥跑：把 `.credentials.yaml` 放进 `$DSH_HOME`（**必须在 el2 上**，见第 2 节），或
`DEEPSEEK_API_KEY=sk-... kc-hmos smoke`。

`kc-hmos doctor` 一次性打印工具链、平台事实（platform / V8 lite / chmod 落位 / 硬链接）与补丁是否在位。

## 依赖清单

| 包 | 用途 | 备注 |
|---|---|---|
| `node` | 运行时 | 26.8.1，`process.platform === 'openharmony'` |
| `ohos-sdk` | 提供 clang 15.0.4（`aarch64-unknown-linux-ohos`） | 2.7 GB；**`/data/service/hnp/bin/clang` 在这台机器上不存在**，别照抄那个路径 |
| `llvm-gcc-compat` | 把 clang 软链成 `gcc`/`cc`/`ld`/`ar` | node-gyp 的 Makefile 默认 `CC=cc`，缺它报 `cc: command not found` |
| `make` `cmake` `ninja` | 原生模块构建 | 本套件只现编 node-pty；koffi 走桩，不构建（见第 1 节对照） |
| `python@3.12` | node-gyp 需要 | |
| `ripgrep` | 文件搜索 | `@vscode/ripgrep` 没有 openharmony 平台包 |
| `git` `bash` `pnpm` | 仓库、`profile/setup.sh`、`dsh plugin` | 系统只有 zsh，`profile/setup.sh` 是 bash 脚本 |

Harmonybrew 门槛：HarmonyOS 6.1.0.117 以上，且要开**开发者选项**与**运行来自非应用市场的扩展程序**。

---

## 五处补丁与它们对应的硬事实

`scripts/patch-node-modules.sh` 幂等，原件留 `.kc-orig`，`--revert` 原样放回。
**`npm install` / 升级 dsh 之后要重打**（`kc-hmos patch` / `kc-hmos patch --global`）。

| 补丁 | 真机事实 | 做法 |
|---|---|---|
| ① koffi 桩 | `dsh-subprocess-local` 顶层 `import koffi` 且加载期就调 `koffi.pointer("void")`；全局树里 `dsh-win32-process` 还在加载期断言两个 Win32 结构体大小（`STARTUPINFOW` 104、`PROCESS_INFORMATION` 24）。openharmony 无原生产物，加载即炸；而 koffi 的**全部实际调用都在 win32 分支** | 惰性桩：`struct()` 按名字查一张大小表让加载期断言过，`load/alloc/encode/decode` 一律 throw，桩不碰任何 `.node` |
| ② 终端检查器 | `createProcessInspector` 对非 linux/darwin/win32 直接 throw（首次开终端才触发）；`/proc` 在，293 项可读 | openharmony 走 `LinuxProcessInspector` |
| ③ 会话落盘 | `dsh-session-persistence-jsonl` 用 `link()` 做原子发布。**鸿蒙全盘禁硬链接**：家目录（hmdfs）EPERM、el2/base（hmfs）EACCES、`/tmp` EROFS | link 失败即 `open(wx)` 占位 + `rename`，保住「不存在才创建」的语义 |
| ④ 写工具新建文件 | `dsh-fs-local` 的 createIfAbsent 同样走 `link()` | 同 ③ |
| ⑤ ripgrep | `@vscode/ripgrep` 按 `ripgrep-${platform}-${arch}` 解析平台包，没有 openharmony；`dsh-tool-fs-search` 的 `execPath-rg` 侧车**只在 pkg 打包的二进制里生效**，普通 node 进程走不到 | 造一个本地平台包 `@vscode/ripgrep-openharmony-arm64`，`bin/rg` 软链到 Harmonybrew 的 rg |

Web 形态（`--global`）多一处：`dsh-attachment-local` 的附件落盘也走 `link()`，同 ③。

不是补丁、但必须做的三件：`DSH_HOME` 放 el2、`KINGCODE_LSP=0`、node-pty 用本机 clang 现编。

---

## 1. 为什么 koffi 用桩而不是源码构建

koffi 在这条路上只被两个 win32 专用模块用到（`dsh-win32-process`、`dsh-sandbox-windows-acl`），
但它们是**顶层 import**，所以在 openharmony 上「永远用不到」和「加载期必炸」同时成立。三种解法：

| 解法 | 代价 | 本套件 |
|---|---|---|
| **惰性桩**（本套件） | 一个 1.9 KB 的 JS 文件；桩要实现 `pointer/struct` 并让 `struct()` 返回正确的大小，否则 `dsh-win32-process` 的加载期断言不过（`STARTUPINFOW layout mismatch: koffi computed undefined`） | ✅ |
| 源码构建 | 要 cmake + ninja + 自写 toolchain 文件（cmake 不认识 HarmonyOS，`CMAKE_SYSTEM_PROCESSOR=unknown` 会 fallback 到 x86-64 汇编），还要**改 CMakeLists 关掉 POST_BUILD strip**——鸿蒙的 dlopen 拒绝加载被 strip 过的 `.so` | ❌ 太重 |
| 让 import 变惰性 | 改 `dsh-sandbox-local` 把 windows-acl 的顶层 import 改成 win32 条件 `await import()` | 也可行；但治不了 `dsh-subprocess-local` 自己那处 |

> **「鸿蒙 dlopen 拒绝 stripped 库」这条规律不是我们发现的**，来自 [shd101wyy/deepseek-harness-harmonyos](https://github.com/shd101wyy/deepseek-harness-harmonyos)
> 的对照实验（见下方对照表）。它同时解释了另一件事：`@vscode/ripgrep-linux-arm64` 自带的静态
> 二进制在鸿蒙上 `chmod +x` 后执行仍 `Permission denied`——也是 strip 过的。所以补丁⑤ 必须软链
> Harmonybrew 的 rg，而不能借用 linux-arm64 包里的那个。

## 2. 为什么 DSH_HOME 必须放 `/data/storage/el2/base`

`dsh-credentials-local` 启动时校验凭证文件权限：组/其他位不为 0 就拒绝启动。
而**家目录是 hmdfs，chmod 是 no-op**——`chmod 600` 之后 `stat` 仍是 `660`，华为分布式文件系统
的权限模型固定如此。两条路：

- 打补丁让 openharmony 跳过这个检查（参考项目的做法）；
- **换 home**：`/data/storage/el2/base` 是 hmfs，mode 位如实生效（实测 600）。本套件选这条，
  因为它不用改上游代码，升级后不用重打。

`scripts/env.sh` 里 `DSH_HOME` 默认就指那儿。代价是这个目录属于应用数据区，卸载 HiShell 会一起没。

## 3. 为什么必须 `node --expose-internals`

起服务会报 `Cannot find package 'kingcode-web-brand'`——profile 层的包一个都解析不到。

根因在 `cordis-plugin-loader`：它要拿 Node 的**内部 ESM 加载器**来解析 profile 里的包，
取法有两条——`process.execArgv` 里有 `--expose-internals` 就直接 `require("internal/modules/esm/loader")`，
否则退回 `node-addon-require-builtin`。而后者是个原生 addon，**没有 openharmony 构建**
（`No usable native binding found for node-addon-require-builtin-openharmony-arm64`），退化之后
loader 拿不到内部加载器，profile 包就解析不到。

所以：

```sh
node --expose-internals "$(readlink -f "$(command -v dsh)")" --profile kingcode --port 3081 --no-open
```

`NODE_OPTIONS="--expose-internals"` 不行（node 明确拒绝在 NODE_OPTIONS 里出现它），只能作为
命令行旗标。`kc-hmos start` 已经这么做了。

## 4. 沙箱：只能 `danger-full-access`

`dsh-sandbox-local` 的后端链在 Linux 上是 `bwrap` → `landlock`，鸿蒙上两者都不可用
（内核禁 user namespace；未启用 Landlock LSM），于是任何受限模式都 fail-closed 抛
`SANDBOX_UNAVAILABLE`。上游留了部署侧开关：

```sh
export DSH_PERMISSION_MODE=danger-full-access
```

`env.sh` 里默认就是它。副作用要知道：`dsh-base` 会把权限审批从 `ask` 切到 `never`，
工具调用不再逐次询问。**这台机器上等于没有进程级围栏**，KingCode 自己的破坏性命令闸门
（`plugins/command-guard.js`）是护栏、不是沙箱。

## 5. 浏览器会话认证（dsh 0.1.2-alpha.2 起，rc.6 时代没有）

整个 Host API 要一枚签名 cookie，**loopback 也不豁免**：

- 裸访问 `GET /` → **401**（正文 `dsh web authentication required; reopen the URL printed by dsh web.`）
- `GET /?token=<每进程随机>` → **303** + `Set-Cookie: dsh-auth-<sha256(authority)>`，`Max-Age` 30 天、`HttpOnly`、`SameSite=Strict`
- 之后 `GET /` → 200，`/api` 放行；静态资源（`/favicon.svg`）本来就公开

要点两条：**签名密钥落盘在 `$DSH_HOME/.credentials.yaml`**，所以重启服务只换 launch token，
已发的 cookie 30 天内继续有效（拿旧 token 的地址访问，只要 cookie 有效也会被 303 到 `/`）；
但 **cookie 与 `host:port` 绑定**，换端口或换 IP 就要重新拿带 token 的地址。

`kc-hmos status` 因此把首页 **401 判成「活着」**——那是认证生效的正常态，只有连不上（000）才是死了。

## 6. 鸿蒙客户端壳（可选）

[KingCode 仓库的 `harmony/`](https://github.com/Promethues0/kingcode/tree/main/harmony) 是个
ArkTS + ArkWeb 壳（DevEco 打开、自动签名、真机 Run）。原生形态下它连 `127.0.0.1:3081`，
比连虚拟机 IP 少一整类问题：不用绑 `0.0.0.0`、没有 `/api` 信任名单与 IP 漂移、loopback 是安全
上下文所以剪贴板可用、设置面不再降级成 memory 模式。

首次在地址页填**带 `?token=` 的完整地址**（`kc-hmos url` 打印的那条）。实测能加载出完整工作区。
壳目前还没接 401 判读（`onErrorReceive` 不认 HTTP 错误，要 `onHttpErrorReceive`），所以 token 过期
或换端口时是白页，得手动改地址。

---

## 已知环境坑

1. **CapsLock 会让自动化输入整体反转**。用 `uitest uiInput text` 驱动 HiShell 时，如果系统
   CapsLock 开着，`sh ~/x.sh` 会变成 `SH ~/X.SH`。查 `~/.zsh_history` 能看到实际落地的命令。
2. **官方 npm 源不稳**。参考项目直接说官方源不可用、必须 `--registry=https://registry.npmmirror.com`；
   我们 09-02 晚用官方源装成功，09-03 早上两个域名都 `http=000`。`kc-hmos` 留了 `KC_REGISTRY`。
3. **出网 TLS 层会整体断**。09-03 上午实测：DNS 解析正常（19 ms）、TCP 连接成功（36 ms），但
   HTTPS 拿不到响应，`api.deepseek.com` 与 `registry.npmjs.org` 同时 `http=000`。表现在 Web UI 里
   就是 `本轮运行失败 DeepSeek API request to https://api.deepseek.com failed` / `TRANSPORT`，
   重试 5 次全败。**这不是引擎的问题**，`kc-hmos status` 会单独把出网列成一项。
4. **`/tmp` 是只读 erofs**，`TMPDIR` 要指别处（`env.sh` 指到家目录）。
5. **`timeout` 是 toybox 版**，实测不一定能按时杀掉 node 子进程；脚本里别依赖它做硬超时。
6. **hdc shell 与 HiShell 是两个环境**。hdc 进的是 `uid=2000 shell`、`u:r:sh:s0`，看不到
   `/storage/Users/`，`/dev/ptmx` 连 `ls` 都拒绝，硬链接拒绝，PATH 里没有 node——全是悲观值，
   **不能拿它的探针结果代表 HiShell**。两边的共享目录是 `/storage/media/100/local/files/Docs`
   （= HiShell 的家目录），递脚本、取日志走这里。

## 与另外两个鸿蒙化项目的对照

同一时期至少有三份鸿蒙 PC 上跑 DeepSeek Harness 的记录，思路一致、解法各有取舍：

| 项 | [shd101wyy/deepseek-harness-harmonyos](https://github.com/shd101wyy/deepseek-harness-harmonyos) | [u010189254/dsh-harmonyos-deploy](https://gitcode.com/u010189254/dsh-harmonyos-deploy) | 本项目 |
|---|---|---|---|
| 对象 | 上游 dsh 原版 Web UI | 上游 dsh | **KingCode**（自有组合树、品牌层、preset、eval），CLI + Web + ArkWeb 壳 |
| dsh 版本 | 0.1.0-rc.6 | 0.1.0-rc.6 | **0.1.2-alpha.3** |
| 设备 | MateBook Pro / Kernel 1.12.0 | 鸿蒙 PC 7.0 / API 26 / Kernel 1.13.0 | MateBook 14 / 7.0.0.102 / API 26 / Kernel 1.13.0 |
| koffi | **源码构建**（cmake toolchain 文件 + 关掉 POST_BUILD strip） | patch `dsh-sandbox-local`，把 windows-acl 的顶层 import 改成 win32 惰性 | **惰性桩**（带结构体大小表，同时治 subprocess-local 与 win32-process） |
| ripgrep | patch `@vscode/ripgrep/lib/index.js` 回退到 brew 的 rg | 手工放 `~/.local/bin/rg` | 造本地平台包 `@vscode/ripgrep-openharmony-arm64` |
| 硬链接 | 只 patch session-persistence | patch session-persistence + attachment-local | **三处**：+ `dsh-fs-local`（写工具新建文件，前两家都没治） |
| 凭证权限 | patch `dsh-credentials-local` 跳过 mode 校验 | 同（DSH_HOME 放 el2） | **DSH_HOME 放 el2**，不改上游代码 |
| sharp | `@img/sharp-wasm32` | 同 | 同（注意连带 `@emnapi/runtime`、`tslib`） |
| `--expose-internals` | 因 HMR 服务要求 | — | 因 **profile 包解析**（`node-addon-require-builtin` 无 openharmony 构建） |
| 沙箱 | `DSH_PERMISSION_MODE=danger-full-access` | 同 | 同 |
| 会话认证 | rc.6 没有这道闸 | 同 | **有**（alpha.2 起 token→cookie，loopback 不豁免） |
| 起停 | `dsh-hmos` 脚本，多实例 / status / restart / 版本门控 | `setsid` + mkdir 原子锁 + HTTP 探活 | `kc-hmos`，含 `doctor` 与出网体检 |
| 开机自启 | 无 | 四层钩子（/etc/profile、.zshenv、.zshrc、XDG autostart），实测 1–3 分钟延迟 | 无 |

**最值得从它们那里抄的**：`dsh-hmos` 的版本门控（npm reify 每次都会重解包整棵树、把补丁冲掉，
所以要先比对版本再决定装不装）、`setsid` + mkdir 锁比 nohup + PID 文件抗环境差异、以及那套开机自启钩子。
**我们这边多出来的**：`dsh-fs-local` 那处硬链接（写工具新建文件会 `FS_IO_ERROR`，前两家都会踩）、
alpha 通道的会话认证、以及 el2 换 home 这条免补丁的路。

## 还没做的

- Web 形态里的一次真回答（等设备出网恢复）。
- 从零 `npm ci --ignore-scripts` 的完整顺序没复跑过——现在设备上这棵树是逐步摸出来的。
- 关掉 HiShell 窗口后引擎是否存活、开机自启。
- 壳里接 `onHttpErrorReceive` 判 401/403。
- 向上游提三件事（只开 Discussions）：把 `openharmony` 当 POSIX 认、`link()` EPERM 回落 rename、
  koffi 改惰性加载——合入后 ①②③④ 都不再需要。

## 许可

MIT。补丁改的是 `node_modules` 里的上游代码，各自遵循其原有许可。
