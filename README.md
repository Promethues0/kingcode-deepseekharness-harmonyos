# KingCode 原生跑在鸿蒙 PC（HiShell，不经虚拟机）

把 [KingCode](https://github.com/Promethues0/kingcode)（基于 DeepSeek Harness 的编程智能体）
**原生**装进鸿蒙电脑：Node 是 Harmonybrew 装的 `platform=openharmony` 构建，跑在系统自带终端
HiShell 里，没有虚拟机、没有容器。

> 设备：HUAWEI MateBook 14 / HarmonyOS 7.0.0.102 / API 26 / HongMeng Kernel 1.13.0 / aarch64
> Node 26.8.1（`process.platform === 'openharmony'`，V8 非 lite、JIT 正常）
> dsh `0.1.2-alpha.5`（上游 alpha 通道；`kc-hmos install-web` 装的就是这一版）
>
> 注意一处分叉：下面「验证到哪一步了」表里 B 级那次真机记录跑的是**全局 alpha.3**，
> 主仓库随后才升到 alpha.5。五处补丁的锚点在 alpha.5 上逐字命中且唯一（已核），
> 但 B 级链路本身尚未在 alpha.5 上复跑。

## 验证到哪一步了

| 形态 | 状态 | 证据 |
|---|---|---|
| **CLI（A 级）** | ✅ 全通 | 整棵组合树 boot 成功；无钥烟测恰好死在 `MISSING_CREDENTIAL`；`npm test` 全绿；带钥 `say hi` 退出码 0；一个真用工具的任务里 **bash 工具经 node-pty 在原生内核上跑 `uname -a`**（回 `HongMeng Kernel 1.13.0`）、**grep 工具经 ripgrep** 数出 README 26 行 |
| **Web（B 级）** | ✅ 全通 | 全局 dsh 原生 boot；`GET /` 401 → `?token=` 303 换 cookie → 200；`/api` 到网关；鸿蒙 ArkWeb 壳连 `127.0.0.1:3081` 加载出完整工作区（品牌层、侧栏、KingCode 预设、设置→模型页「API 密钥已配置」）；**在壳里发一条真消息拿到回答**——2 次工具调用，bash 经 node-pty 回 `HongMeng Kernel 1.13.0 … aarch64 Toybox`，grep 经 ripgrep 数出 26 行，20K token / 11 秒 / 89 tok/s / 缓存命中 46%（DeepSeek-V4-Pro High，完全权限） |
| 上架应用市场（C 级） | ❌ 未做 | 应用沙箱（normal_hap 域）有 neverallow 限制，是另一条路 |

> **这张表说的是「这台设备上跑通过」，不是「clone 下来就跑得通」。** A/B 级的证据产于
> 一套手工命令序列，`scripts/kc-hmos` 是事后封装的；从零 `git clone` → 能用这条整链
> 尚未一次性复跑过。2026-09-03 的一轮审计（六维度 + 对抗性核实）在其中挖出并修掉了
> 若干真缺陷，最硬的一个是 README 教的 `ln -sf` 装法会让脚本找不到自己的 `env.sh`
> ——除 `url`/`stop` 外每个子命令都死在第一句（`$0` 是软链、`HERE` 没解引用）。
> 现在 `kc-hmos` 会 `readlink -f "$0"`，`install-web` 的三处静默跳过全改成硬失败，
> `patch-node-modules.sh` 的自检也进退出码了。
>
> **A 级已经从零跑通一次（09-03 15:21–15:23，真机 HiShell，全程经软链调用）**：全新 clone
> 的仓库 + 全新 `DSH_HOME`，`deps` rc=0 → `install-cli` rc=0（`npm ci --ignore-scripts` 293 包、
> node-pty 现编出 `pty.node`、**八处补丁全 PATCHED**、自检全 OK）→ `smoke` **PASS，整棵树
> boot 成功、恰好死在 `MISSING_CREDENTIAL`**。经软链这条路正是修之前必死的那条。
>
> **B 级还没跑完**：`install-web` 把全局 dsh 从 alpha.3 装到 alpha.5 用了 8 分钟（走 npmmirror），
> 然后**卡在 sharp WASM 那一步二十多分钟没有任何进展**（不是失败、是挂住——比失败更难排查，
> 因为 `die` 等不到）。这一条记进「还没做的」。
>
> 另外，当天 `github.com` / `registry.npmjs.org` / `nodejs.org` 全部 `http=000`，而
> `registry.npmmirror.com` 200、`api.deepseek.com` 401、`gitee`/`gitcode`/`atomgit` 200
> ——**第 0 步的 `git clone` 在这种网络下直接走不了**。这轮是在另一台机器上做 `git clone --depth 1`
> 再把整个目录（含 `.git`）搬进设备来等价替代的，后面每一步都是设备上原样跑的。


## 三步装完

前提：**Harmonybrew 已装好**（见下节门槛与安装入口）。`git` 和 `curl` 也由它提供，所以先装它们再 clone。

```sh
brew install git curl                                  # clone 与体检要用；deps 里也会再装一次（幂等）

git clone https://github.com/Promethues0/kingcode-deepseekharness-harmonyos.git ~/kc-hmos
git clone https://github.com/Promethues0/kingcode.git ~/kingcode
ln -sf ~/kc-hmos/scripts/kc-hmos ~/.harmonybrew/bin/kc-hmos

kc-hmos deps          # Harmonybrew 依赖（ohos-sdk 2.7 GB，慢）
kc-hmos install-cli   # A 级：仓库依赖 + 现编 node-pty + 打补丁
kc-hmos smoke         # 判据：恰好死在 MISSING_CREDENTIAL

kc-hmos install-web   # B 级：全局 dsh + sharp WASM + 补丁 + profile/preset
DEEPSEEK_API_KEY=sk-... kc-hmos start    # 起 Web 引擎（key 见下）
kc-hmos url           # 打印带 token 的地址，浏览器/鸿蒙壳里首次用它
```

**`install-cli` 不能跳。** 它不只是「CLI 形态」——`profile/setup.sh` 是把仓库 **link** 进
profile 而不是拷贝，preset 里 `kingcode/plugins/env-context.js` 的
`import z from '@deepseek-ai/schemastery'` 从仓库原位往上解析，命中的是
`~/kingcode/node_modules`。跳过它，Web 引擎能起、工作区能开，一开会话就
`ERR_MODULE_NOT_FOUND`。`install-web` 现在会先检查再往下走。

### 第一次怎么把 API key 弄进去

三条路，按省事排：

1. **环境变量（推荐）**：`DEEPSEEK_API_KEY=sk-... kc-hmos start`。dsh 的凭证层里进程环境
   优先级最高，`nohup` 起的引擎继承调用 shell 的环境。CLI 侧同理：`DEEPSEEK_API_KEY=sk-... kc-hmos smoke`。
2. **在界面里填**：引擎起来后进设置 → 模型 → DeepSeek → 编辑，填 key 保存。
   **只有 loopback 能这么干**——dsh 客户端按页面 hostname 自己关闸
   （`dsh-client-ui-settings` 里 `persistence = $host.isLoopback ? "host" : "memory"`），
   非 loopback 的页面 describe 镜像根本不发请求，「模型」页会报
   `settings are unavailable in this browser`。原生形态连的正是 `127.0.0.1`，所以这条通。
3. **落盘**：写 `$DSH_HOME/.credentials.yaml`（默认 `/data/storage/el2/base/kingcode-home/.credentials.yaml`）。
   **格式不是 `deepseek: apiKey:`**，是：

   ```sh
   cat > "$DSH_HOME/.credentials.yaml" <<'YAML'
   version: 1
   refs:
     DEEPSEEK_API_KEY: sk-你的key
   YAML
   chmod 600 "$DSH_HOME/.credentials.yaml"      # 少这一步 dsh 直接拒绝启动
   ```

   那个 `chmod 600` 是硬的：`dsh-credentials-local` 见到组/其他位不为 0 就
   `throw ... is readable beyond its owner`。这也正是 `DSH_HOME` 必须放 el2 的原因（见第 2 节）。

`kc-hmos doctor` 一次性打印工具链、平台事实（platform / V8 lite / chmod 落位 / 硬链接）、
**三处 dsh 版本是否分叉**、**profile / preset / sharp 三件必需品**、**凭证在不在与它的 mode**、
以及补丁是否在位。`kc-hmos start` 超时时先看它。

## 依赖清单

| 包 | 用途 | 备注 |
|---|---|---|
| `node` | 运行时 | 26.8.1，`process.platform === 'openharmony'` |
| `ohos-sdk` | 提供 clang 15.0.4（`aarch64-unknown-linux-ohos`） | 2.7 GB；**`/data/service/hnp/bin/clang` 在这台机器上不存在**，别照抄那个路径 |
| `llvm-gcc-compat` | 把 clang 软链成 `gcc`/`cc`/`ld`/`ar` | node-gyp 的 Makefile 默认 `CC=cc`，缺它报 `cc: command not found` |
| `make` `cmake` `ninja` | 原生模块构建 | 本套件只现编 node-pty；koffi 走桩，不构建（见第 1 节对照） |
| `python@3.12` | node-gyp 需要 | |
| `ripgrep` | 文件搜索 | `@vscode/ripgrep` 没有 openharmony 平台包 |
| `git` `bash` `pnpm` | 仓库、`profile/setup.sh`、`dsh plugin` | 系统只有 zsh，`profile/setup.sh` 是 bash 脚本；`git` 在第一条 clone 就要用，所以三步块里先单独装它 |
| `curl` | `kc-hmos status` 的端口探活与出网体检 | 缺了不报错，只是那两行静默消失——排障时看到的是一份残缺报告 |

Harmonybrew 本身的安装见 <https://harmonybrew.atomgit.com>；门槛是 HarmonyOS 6.1.0.117 以上，且要开**开发者选项**与**运行来自非应用市场的扩展程序**。本文所有实测都在 7.0.0.102 / API 26 / Kernel 1.13.0 的一台 MateBook 14 上——**这套方案的四条地基（el2 是 hmfs、家目录 hmdfs 的 chmod 是 no-op、`/tmp` 只读 erofs、全盘禁硬链接）在别的机型或 6.1 上是否同样成立，没有验过**。`kc-hmos doctor` 里的 chmod 与硬链接两项探针就是拿来当场核这件事的，红了就别往下走。

---

## 五处补丁与它们对应的硬事实

`scripts/patch-node-modules.sh` 幂等，原件留 `.kc-orig`，`--revert` 原样放回。
**`npm install` / 升级 dsh 之后要重打**（`kc-hmos patch` / `kc-hmos patch --global`）。

| 补丁 | 真机事实 | 做法 |
|---|---|---|
| ① koffi 桩 | `dsh-subprocess-local` 顶层 `import koffi` 且加载期就调 `koffi.pointer("void")`；全局树里 `dsh-win32-process` 还在加载期断言两个 Win32 结构体大小（`STARTUPINFOW` 104、`PROCESS_INFORMATION` 24）。openharmony 无原生产物，加载即炸；而 koffi 的**全部实际调用都在 win32 分支** | 惰性桩：`struct()` 按名字查一张大小表让加载期断言过，`load/alloc/encode/decode` 一律 throw，桩不碰任何 `.node` |
| ② 平台闸（**两处**） | 同一个文件里有两处 `platform === "linux"`：`createProcessInspector` 对非 linux/darwin/win32 直接 throw（首次开终端才触发，响亮）；`treeAlive` 里「进程组只剩僵尸就判死」那条精化（静默降级——判据退回只剩 `kill(-pid,0)`，`observeTreeExit` 的循环不收敛，表现为 bash 工具调用迟迟不收尾） | 两处都放开给 openharmony，走 `LinuxProcessInspector` / 走僵尸组判定 |
| ③ 会话落盘 | `dsh-session-persistence-jsonl` 用 `link()` 做原子发布。**鸿蒙全盘禁硬链接**：家目录（hmdfs）EPERM、el2/base（hmfs）EACCES、`/tmp` EROFS | link 失败即 `open(wx)` 占位 + `rename`，保住「不存在才创建」的语义 |
| ④ 写工具新建文件 | `dsh-fs-local` 的 createIfAbsent 同样走 `link()` | 同 ③ |
| ⑤ ripgrep | `@vscode/ripgrep` 按 `ripgrep-${platform}-${arch}` 解析平台包，没有 openharmony；`dsh-tool-fs-search` 的 `execPath-rg` 侧车**只在 pkg 打包的二进制里生效**，普通 node 进程走不到 | 造一个本地平台包 `@vscode/ripgrep-openharmony-arm64`，`bin/rg` 软链到 Harmonybrew 的 rg |

Web 形态（`--global`）多一处：`dsh-attachment-local` 的附件落盘也走 `link()`，同 ③。
**这一处要连着改两个地方**：回退用 `rename` 把 `temporary` 搬走之后，紧跟其后的裸
`await unlink(temporary)` 必抛 `ENOENT`，被外层 catch 统一翻成 `ATTACHMENT_WRITE_FAILED`——
字节其实已经落盘了。症状是「任何新图第一次贴报保存失败、同一张图重贴一次就成功」
（第二次走 `EEXIST` 分支，`temporary` 还在），非常像间歇性故障。

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

## 6. 鸿蒙客户端壳（可选，而且**装不了的人只能走浏览器**）

先把话说死：**本项目不提供、也无法提供预编译的 `.hap`。** 这不是懒，是 HarmonyOS 的分发模型：

- **调试签名绑设备**。我从本机产物里解出 provision profile 核过：`type: debug`、
  `issuer: app_gallery`、`bundle-name: com.kingcode.client`、`debug-info.device-ids` 是一串
  **具体的设备 UDID**、有效期整 365 天。换台机器装，UDID 不在名单里就被拒。调试设备还有
  100 台/年的额度上限。
- **未签名的 hap 直接被拒**：`code:9568320 error: no signature file`。
- **发布证书签的 hap 反而更装不上**：`hdc install` 报 `9568322`（签名来源不受信任），
  release 只走应用市场。
- 「任何人下载即可装」的形态只有 AGC 内测/公测或上架——那是 C 级，要实名开发者账号 + 审核。

所以要用壳，你得自己构建：**另一台 Windows 或 macOS 电脑**（DevEco Studio 没有鸿蒙 PC 版，
仓库里也没有 `hvigorw`，鸿蒙 PC 自己造不出这个 hap）+ **一个华为开发者账号**（自动签名要
向 AGC 换取调试证书与 profile）+ 把鸿蒙 PC 连上去 Run。还有一处会撞车：`com.kingcode.client`
这个 bundleName 已被本项目作者的 AGC 账号占用，**你要在 `AppScope/app.json5` 里换成自己的**。
入库的工程是 `"signingConfigs": []`，首次 Build 不会失败，只会打一条
`WARN Will skip sign 'hos_hap'` 然后产出 unsigned hap——照着装就是 9568320。

**没有壳也能用**：`kc-hmos url` 那条地址在系统自带浏览器里同样能开（原生形态是 `127.0.0.1`，
loopback 属安全上下文，剪贴板与设置面都不降级）。**但这条路我们只在虚拟机路线的模拟器上验过，
真机 + 127.0.0.1 + alpha.5 的 token→cookie 组合一次都没跑过**，见「还没做的」。

[KingCode 仓库的 `harmony/`](https://github.com/Promethues0/kingcode/tree/main/harmony) 是那个
ArkTS + ArkWeb 壳。原生形态下它连 `127.0.0.1:3081`，
比连虚拟机 IP 少一整类问题：不用绑 `0.0.0.0`、没有 `/api` 信任名单与 IP 漂移、loopback 是安全
上下文所以剪贴板可用、设置面不再降级成 memory 模式。

首次在地址页填**带 `?token=` 的完整地址**（`kc-hmos url` 打印的那条）。实测能加载出完整工作区，
并在壳里发消息拿到带工具调用的真回答。

**引擎重启之后不用重贴地址**：cookie 的 signing secret 落盘在 `$DSH_HOME/.credentials.yaml`，
壳里保存的旧地址照样能进，只是 WebSocket 断了——点一次左下角「连接异常，点击立即重连」即可。

壳已接 `onHttpErrorReceive`（`onErrorReceive` 只认网络错误，HTTP 401/403 不触发它）：cookie 过期或
换了端口时会回到地址页并提示「用带 token 的地址换一次 cookie」，而不是白页。另外它在 `onPageEnd`
调 `WebCookieManager.saveCookieSync()` 强制落盘——ArkWeb 的 cookie 每 30 秒才周期性写盘，
换到 cookie 后 30 秒内被杀就丢了。（主仓库 commit 22d6820）

---

## 引擎的生命周期绑死在 HiShell 上（2026-09-03 实测）

三档对照，`nohup` 与 `setsid` 都试过：

| 操作 | 引擎 | 端口 3081 |
|---|---|---|
| HiShell 切到后台（比如切去用鸿蒙壳） | **活着** | 仍 LISTEN |
| 关掉 HiShell 窗口（点标题栏的 ✕） | **立刻死** | 连 TIME_WAIT 都不留 |
| `aa force-stop com.huawei.hmos.hishell` | **立刻死** | TIME_WAIT 后消失 |

即使用 `setsid` 让引擎自成会话、父进程被 init 收养（实测 `ppid=1`、`sid` 等于自身 pid），
也挡不住——鸿蒙在应用终止时收走整个应用沙箱的进程组，跟 POSIX 那套「脱离控制终端就活得下去」
不是一回事。所以 **正常使用形态是「HiShell 窗口留着、切后台」**，不是「起完就关」；
而参考项目里那套四层开机自启钩子解决的是「怎么自动跑起来」，解决不了「窗口关了怎么办」。

## 已知环境坑

1. **CapsLock 会让自动化输入整体反转**。用 `uitest uiInput text` 驱动 HiShell 时，如果系统
   CapsLock 开着，`sh ~/x.sh` 会变成 `SH ~/X.SH`；查 `~/.zsh_history` 能看到实际落地的命令。
   `uitest uiInput keyEvent 2074`（KEYCODE_CAPS_LOCK）不一定能改变它。**`uitest uiInput inputText
   <x> <y> '<文本>'` 也救不了**——09-03 实测它同样被反转（本文早先写它「按坐标直接写入、大小写
   如实」，撤回）。可用的只有一条：**反着打**（要 `sh ~/x.sh` 就输入 `SH ~/X.SH`）。
   另外 Ctrl+A 在 WebView 里会选中整页并弹出上下文菜单，把后续输入全吃掉。
2. **官方 npm 源不稳**。参考项目直接说官方源不可用、必须 `--registry=https://registry.npmmirror.com`；
   我们 09-02 晚用官方源装成功，09-03 早上两个域名都 `http=000`。`kc-hmos` 留了 `KC_REGISTRY`，
   但**它只盖住 `kc-hmos` 自己发的三条 npm 命令**——`profile/setup.sh` 里 `dsh plugin add` 走的是
   pnpm，不在伞下。要一次盖全就写进 `~/.npmrc`：`npm config set registry <mirror>`（npm 与 pnpm 都读它）。
   还有第三条独立路径：**node-gyp 取 node headers 走 `disturl`，既不跟 registry 也不跟镜像**，
   而这条路上要现编两次 node-pty。`env.sh` 认 `KC_NODE_DISTURL`（姊妹路径
   `deploy/harmonyos-pc/install.sh` 早就有这个旋钮，原生路线以前漏了）。**注意 npm 11 已经
   把 `npm_config_disturl` 判为未知配置**（每条 npm 命令都打一行 `npm warn Unknown env config
   "disturl"`，并声明下个大版本停止支持），所以 `env.sh` 同时导出 node-gyp 自己认的
   `NODEJS_ORG_MIRROR`。可用的镜像值：`https://registry.npmmirror.com/-/binary/node`
   （09-03 实测该镜像上有 v26.8.1 的 headers）。
3. **出网 TLS 层会整体断**。09-03 上午实测：DNS 解析正常（19 ms）、TCP 连接成功（36 ms），但
   HTTPS 拿不到响应，`api.deepseek.com` 与 `registry.npmjs.org` 同时 `http=000`。表现在 Web UI 里
   就是 `本轮运行失败 DeepSeek API request to https://api.deepseek.com failed` / `TRANSPORT`，
   重试 5 次全败。**这不是引擎的问题**——同一天网络恢复后（`api.deepseek.com` 回 `http=401`、
   `tls=0.055`）原样重试即通。`kc-hmos status` 会单独把出网列成一项，先看这里再怀疑引擎。
4. **`/tmp` 是只读 erofs**，`TMPDIR` 要指别处（`env.sh` 指到家目录）。
5. **`timeout` 是 toybox 版，会真的挂死**。09-03 实测 `timeout 20 node -v`：版本号打出来了，
   `timeout` 自己永不返回，占着前台，**Ctrl+C 也打不断**，只能开新标签页 `pkill`。
   脚本里一律别用它。
6. **长任务必须前台跑，而且要完全脱离终端**。`sh install.sh &` 这种后台作业会被信号挂起
   （zsh 报 `[1] + suspended (signal)`），现象是安装到一半彻底不动、`ps` 里子进程还在但没有
   CPU 时间。原因是只重定向了 stdout，stderr/stdin 还连着 tty。要么老老实实前台跑，
   要么脚本里写全 `exec > log 2>&1 < /dev/null`。
7. **屏保一上，HiShell 里的活会被冻住**。装 ohos-sdk、`npm ci`、编 node-pty 这类几十分钟的
   步骤，中途锁屏就停在那儿。装之前先把息屏关掉（或 `hdc shell power-shell setmode 602`）。
8. **hdc shell 与 HiShell 是两个环境**。hdc 进的是 `uid=2000 shell`、`u:r:sh:s0`，看不到
   `/storage/Users/`，`/dev/ptmx` 连 `ls` 都拒绝，硬链接拒绝，PATH 里没有 node——全是悲观值，
   **不能拿它的探针结果代表 HiShell**。两边的共享目录是 `/storage/media/100/local/files/Docs`
   （= HiShell 的家目录），递脚本、取日志走这里。

## 与另外两个鸿蒙化项目的对照

同一时期至少有三份鸿蒙 PC 上跑 DeepSeek Harness 的记录，思路一致、解法各有取舍：

| 项 | [shd101wyy/deepseek-harness-harmonyos](https://github.com/shd101wyy/deepseek-harness-harmonyos) | [u010189254/dsh-harmonyos-deploy](https://gitcode.com/u010189254/dsh-harmonyos-deploy) | 本项目 |
|---|---|---|---|
| 对象 | 上游 dsh 原版 Web UI | 上游 dsh | **KingCode**（自有组合树、品牌层、preset、eval），CLI + Web + ArkWeb 壳 |
| dsh 版本 | 0.1.0-rc.6（据其 README 第 12 节的对照式表述，非独立核对） | 0.1.0-rc.6（转引自 shd101wyy README 第 12 节，未一手核对） | **0.1.2-alpha.5** |
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

**最值得从它们那里抄的，只有一件：`dsh-hmos` 的版本门控。** 它挡的是「重复安装把补丁冲掉」——
而这在我们这边有一个更硬的版本：`npm i -g @deepseek-ai/dsh@0.1.2-alpha.5` **只钉住顶层那一个包**，
它下面 ~70 个 `dsh-*` 依赖全是 `^0.1.2-alpha.3`，全局安装目录里连 `.package-lock.json` 都没有。
今天恰好装出 alpha.5（因为它就是最高发布版），上游一发 alpha.6，五处补丁的锚点包就换了树。
锚点变了会 fail-loud（NOMATCH → `install-web` die），锚点没变而语义变了则是静默不对。
**这是这套东西目前最短的一根引信。**

另两件被我们早先列为「值得抄」的，其实解决的不是「即装即用」：`setsid` + mkdir 锁对我们无效
（实测 `setsid` 挡不住关窗口，见上一节）；开机自启钩子解决的是「怎么自动跑起来」，
解决不了「窗口关了怎么办」——参考项目自己也记了 1–3 分钟延迟、有时还要手动开一次 HiShell。

**我们这边多出来的**：`dsh-fs-local` 那处硬链接（写工具新建文件会 `FS_IO_ERROR`，前两家都会踩）、
`dsh-attachment-local` 回退之后那处必抛 ENOENT 的清理（症状是「新图第一次贴必报保存失败、
重贴一次就成功」，像间歇性故障）、`dsh-subprocess-local` 里**第二处** `platform === "linux"` 闸
（`treeAlive` 的僵尸组精化，不开是静默降级成工具调用不收尾）、alpha 通道的会话认证，
以及 el2 换 home 这条免补丁的路。

## 还没做的

**卡在「即装即用」上的：**

- **B 级（Web）那半条整链还没跑通**：`install-web` 会挂在 sharp WASM 那一步（09-03 实测，
  20+ 分钟无进展、无报错）。A 级（CLI）已经从零跑通，见上面的验证说明。
- **`install-web` 的 sharp 那步只防得住「失败」，防不住「挂住」**。现在失败会 `die`，
  但挂住时用户看到的还是一个不动的终端。toybox 的 `timeout` 又不能用（坑 5），暂无好办法。
- **浏览器那条路没在真机上验过**。壳对大多数人装不上（第 6 节），所以「Web 工作区里发一条
  消息拿到回答」实际上压在系统自带浏览器上——而它能不能打 `127.0.0.1`、地址栏吃不吃带
  `?token=` 的长地址、cookie 会不会有落盘窗口，一条记录都没有。
- **全局树没有锁**：`npm i -g dsh@<版本>` 只钉顶层，~70 个子依赖是 caret，没有
  `.package-lock.json`。上游一发新 alpha，补丁锚点就换了树（见对照那节）。
- **B 级链路没在 alpha.5 上复跑**：那次真机记录跑的是全局 alpha.3（补丁锚点已在 alpha.5 上
  逐字复核过、且唯一，但链路本身没重跑）。

**别的：**

- ~~关掉 HiShell 窗口后引擎是否存活~~ —— **已验，答案是不存活**（见「引擎的生命周期」一节）。
- 开机自启：能做的只有「开机自动打开 HiShell 并拉起引擎」，窗口仍必须留着。
- 卸载 / 备份：没有 `kc-hmos uninstall`；而 `DSH_HOME` 在 el2，是 HiShell 的应用数据，
  **清 HiShell 数据 = 会话、凭证、设置一起没**，怎么备份这个目录也还没写。
- 多用户：`DSH_HOME` 与 `~/.harmonybrew` 都是 per-user 的，一台机器换系统用户会怎样，没验过。
- 向上游提三件事（只开 Discussions）：把 `openharmony` 当 POSIX 认（注意
  `dsh-subprocess-local` 里有**两处** `platform === "linux"`，不只 `createProcessInspector`）、
  `link()` EPERM 回落 rename、koffi 改惰性加载——合入后 ①②③④⑤ 都不再需要。

## 许可

MIT。补丁改的是 `node_modules` 里的上游代码，各自遵循其原有许可。
