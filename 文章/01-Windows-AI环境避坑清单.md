# Windows 上装 AI 开发环境，我踩过的 8 个坑

> 这篇是给「装了三天还没跑起来」的人看的。每一条都写了现象、根因、怎么修。

我在 Windows 上给一个 AI 编程工具装插件、配环境，连续踩了八个坑。每个坑都花了半小时以上，所以记下来。

如果你正在装 Claude Code、DeepSeek Harness、OpenClaw 这类工具，大概率会撞上其中三四个。

---

## 坑 1：`node_modules` 只装了一半

**现象**：`pnpm install` 显示成功，但启动报 `Cannot find module`，或者报某个包的子路径解析不到。去 `node_modules` 里一看，那个目录里只有 `presets/`、`scripts/`，`package.json` 和 `plugin.mjs` 不见了。

**根因**：Windows 上应用运行时会锁住自己目录下的文件。你一边开着软件一边装插件，`pnpm` 写文件撞上 `EPERM`，进程退出码被上层忽略了，于是留下一个半截目录。

**怎么修**：

```powershell
# 1. 先看目录是不是真的残缺
Get-ChildItem node_modules\<包名> | Select-Object Name

# 2. 关掉占用它的进程（按命令行匹配，别按名字猜）
Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -like '*<包名>*' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }

# 3. 从备份或重新安装
```

**怎么预防**：装插件之前先把应用退出。装完再用 `node -e "require.resolve('包名/子路径')"` 验证，别信 install 的退出码。

---

## 坑 2：插件装了，但界面上就是不出现

**现象**：`package.json` 里 `dependencies` 有它，`bundles` 里也有它，启动日志没有任何报错，但侧栏就是没有那个按钮。

**根因**：宿主会逐个校验 bundle 内部声明的 peer 依赖版本上限。插件写的上限是 `0.2.1-rc.2`，你跑的是 `rc.3`，校验失败 → **静默禁用**。不报错，不警告，就是没有。

**怎么修**：找宿主的兼容性豁免文件（不同工具名字不一样，可能是 `compatibility.json`、`overrides`、`allowedVersions`），把这两个包加进去。

```json
{
  "exemptions": [
    { "name": "@scope/studio-panel", "reason": "peer cap too low" }
  ]
}
```

**两个坑中坑**：

- 这个文件必须是**无 BOM 的 UTF-8**。PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会写 BOM，整个文件直接失效。
- 验证方法不是看界面，是看 boot 图：用 profile 的副本起一个独立实例，检查启动日志里的 entries 列表里有没有这个包。

---

## 坑 3：Electron 应用双击秒退，退出码 0

**现象**：双击图标，进程管理器里闪一下就没了，没有任何错误弹窗，退出码是 0。

**根因**：环境里有 `ELECTRON_RUN_AS_NODE=1`。Electron 启动时读到这个变量，会把自己当成纯 Node 跑——没有窗口、没有 GUI，主进程执行完立刻退出。

这个变量常被「用 Node 跑 Electron 脚本」的工具链留下来，然后污染整个终端会话。

**怎么修**：

```powershell
# 确认
$env:ELECTRON_RUN_AS_NODE

# 清掉，再从干净环境启动
Remove-Item Env:ELECTRON_RUN_AS_NODE
```

**怎么预防**：从 IDE 或某个 agent 的终端里启动 GUI 程序之前，先检查这个变量。它的杀伤力在于「退出码 0」——所有日志都是干净的，你会以为是程序坏了。

---

## 坑 4：GitHub 下载 2KB/s，装到一半超时

**现象**：`gh release download` 或者 `Invoke-WebRequest` 拉二进制，速度卡在 2KB/s，20MB 的文件下十分钟然后超时。

**根因**：Release 资源走的是 `objects.githubusercontent.com`，国内经常被限速，不是被墙，是限速。

**怎么修**：分段并行 + Range 请求，绕开单连接限速。

```powershell
# 思路：HEAD 拿到 Content-Length，切成 12 段并行 Range 下载，最后拼起来校验 SHA256
```

配合 `gh-proxy` 之类的镜像做前置，实测能把 20MB 从「十分钟超时」变成「三十秒」。

**注意**：拼完一定要校验 SHA256。分段下载拼错一个字节，装上去就是各种玄学崩溃。

---

## 坑 5：`raw.githubusercontent.com` 解析到 0.0.0.0

**现象**：`web_fetch` 报 `getaddrinfo ENOENT`，但浏览器能打开 GitHub 网页。

**根因**：DNS 污染。`Resolve-DnsName raw.githubusercontent.com` 返回 `0.0.0.0`，浏览器走 DoH 所以不受影响，命令行不走。

**怎么修**（需要管理员）：

```
185.199.108.133 raw.githubusercontent.com
185.199.109.133 raw.githubusercontent.com
185.199.110.133 raw.githubusercontent.com
185.199.111.133 raw.githubusercontent.com
185.199.108.133 objects.githubusercontent.com
20.205.243.168 api.github.com
```

写进 `C:\Windows\System32\drivers\etc\hosts`。写之前先备份。

**临时方案**：`$env:HTTPS_PROXY="http://127.0.0.1:7897"` 走代理。

---

## 坑 6：设备码授权在后台进程里不打印

**现象**：`gh auth login --web` 或者任何 OAuth device flow，把 stdout 重定向到文件之后，文件是空的——你拿不到要输入的那串码。

**根因**：这类交互式 CLI 检测到 stdout 不是 TTY，就跳过提示直接等。重定向之后它以为没人看。

**怎么修**：手工走一遍 device flow。

```bash
# 1. 申请设备码
curl -X POST -H "Accept: application/json" \
  -d "client_id=<client_id>&scope=repo" \
  https://github.com/login/device/code

# 2. 拿 user_code 去浏览器输入

# 3. 轮询换 token
curl -X POST -H "Accept: application/json" \
  -d "client_id=<client_id>&device_code=<code>&grant_type=urn:ietf:params:oauth:grant-type:device_code" \
  https://github.com/login/oauth/access_token

# 4. 落库
echo $TOKEN | gh auth login --hostname github.com --with-token
```

---

## 坑 7：脚本里有中文路径就崩

**现象**：PowerShell 脚本里写死中文目录，运行时报「路径不存在」，或者中文变成 `??`。

**根因**：PS 5.1 默认按系统 ANSI 代码页读 `.ps1` 文件。文件是 UTF-8 存的，里面有中文，读出来就是乱码。

**怎么修**：

- 脚本存成 **UTF-8 with BOM**（PS 5.1 认 BOM）
- 或者别写中文字面量，用 `$PSScriptRoot` 往上推

```powershell
$root = Split-Path -Parent (Split-Path -Parent $MyInvocation.MyCommand.Path)
```

**反例**：`start "" /min node "C:\Users\xxx\Desktop\小说\.agents\xxx.mjs"` —— 存成 GBK 就是 `??`，存成 UTF-8 无 BOM 还是乱码，从 10 月建好到 11 月一次都没启动成功过，而 `.cmd` 文件不会报错，你只会觉得「这个功能没生效」。

---

## 坑 8：服务起得来，但功能一直是坏的

**现象**：健康检查 200，端口在监听，日志干净。但实际功能一条都没跑通。

**根因**：健康检查只验「进程活着」，不验「业务跑通」。我这周遇到两次：

- 记忆插件：客户端一直在抓消息，服务端从来没装过。1929 条全积压在本地队列，界面上完全看不出来（插件根本没有展示连接状态的 UI）。
- 记忆提炼：提交返回 `accepted`，但归档全是 `.failed.json`——输出被 4096 token 上限截断，四段内容全废，0 条记忆入库。

**怎么修**：验收标准从「服务在跑」换成「产出物存在」。

- 不是 `health = ok`，是「写入一条，能读回来」
- 不是 `commit = accepted`，是「归档目录里有 `.done`，里面有 `memory_diff.json`」

---

## 小结

这八个坑有个共同点：**都不报错**。

Windows 上的失败模式大多是静默的——退出码 0、日志干净、界面正常，功能是坏的。所以排查的第一原则不是看错误信息，是**验证产出物**。

---

*如果你卡在某一步，可以远程帮你看。*
