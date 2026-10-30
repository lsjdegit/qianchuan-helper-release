# INSTALL-AI.md · 让对话前端把千川管家装好（**给 AI 执行的手册**）

> **给谁看**：WorkBuddy / DSH 这类**能跑命令、能读本机文件**的对话前端。
> 人不用逐条读这份；人只需要往对话里贴**仓库地址 + 一句「帮我装千川管家」**。
>
> **你（AI）的目标**：按本文把程序装好 → 起控制台 → 报设备码 → 等用户导入授权 → 装技能 → 冒烟 → 用四句话汇报。
> 每一步都有「**怎么判断成了**」。判断不了就**停下来问用户**，不要猜。
>
> **仓库**：`https://github.com/lsjdegit/qianchuan-helper-release`
> **更新源根**：`https://github.com/lsjdegit/qianchuan-helper-release/releases/latest/download/`
> **默认安装目录**：`%LOCALAPPDATA%\Programs\千川管家`（可以换盘；**不要**装进 `C:\Program Files`）

## 0. 三条铁律（违反任何一条都算装失败）

| # | 铁律 | 具体 |
|---|---|---|
| 1 | **不猜** | token、设备码、路径、版本号一律从**输出 / 接口 / 文件**里拿；拿不到就问用户。**不许自己拼 token、不许编设备码** |
| 2 | **不假装** | 授权只能由提供方签发。你**不能**签、不能伪造、不能在授权前对用户说「已经能跑了」 |
| 3 | **不动账户** | 安装全程只做：读网页 / 下载 / 解压 / 起控制台 / 读状态 / 复制技能文件。**不要**跑 `run` 真执行、不要点发布、不要删任何东西 |

**版本判据只有一个**：`node qh.mjs --version`。
`node qh.mjs console --version` 是**错的**（`console` 没有这个参数）—— 真机上已经有人这么写错过。

---

## 1. 取最新版（先读 `latest.json`，不要看网页上的版本号）

```powershell
$base = 'https://github.com/lsjdegit/qianchuan-helper-release/releases/latest/download'
$tmp  = Join-Path $env:TEMP ('qh-install-' + (Get-Date -Format 'yyyyMMdd-HHmmss'))
New-Item -ItemType Directory -Path $tmp -Force | Out-Null

Invoke-WebRequest -UseBasicParsing "$base/latest.json" -OutFile (Join-Path $tmp 'latest.json')
$latest = Get-Content -Raw -Encoding UTF8 (Join-Path $tmp 'latest.json') | ConvertFrom-Json
$latest | ConvertTo-Json
```

**你会看到**：`version`（形如 `2.0.2`）、`zip`（形如 `qianchuan-helper-2.0.2.zip`）、`sha256`（64 位十六进制）、`notes`（更新说明）。

**怎么判断成了**
- `version` 是三段数字（`X.Y.Z`）；
- `zip` 是 `qianchuan-helper-<同一个 version>.zip`；
- `sha256` 是 64 个十六进制字符。

**注意**
- Public Release 页面**只有两个附件**：这个 zip 和 `latest.json`。页面底下 GitHub 自动生成的 `Source code (zip/tar.gz)` **不是安装包**，别下。
- 网页上显示的版本可能滞后；一律以 `latest.json` 为准。

---

## 2. 检查**系统 PATH 里**的 node（不要拿你自己的执行环境顶替）

在**普通 cmd / PowerShell 窗口**里确认（不是「你的运行环境里能跑」）：

```powershell
where node
node -v
```

**怎么判断成了**
- `where node` 打出一条真实路径（例如 `C:\Program Files\nodejs\node.exe`）；
- `node -v` 打出的版本 **≥ v18**。

**通过 → 直接继续，不要再劝用户装 Node。**（2026-10-30 真机实测：那台机器普通 cmd 里就是 `C:\Program Files\nodejs\node.exe`、`v22.23.2`，双击启动器也正常。）

**没过 → 才让用户装**：
- 官网 <https://nodejs.org>（LTS，一路下一步）；或
- `winget install OpenJS.NodeJS.LTS`。
装完让用户在**新开的**窗口里再跑上面两条命令。

> ⚠ **别把 `<18` 直接判死**（2026-10-30 本机复测）：有的机器 PATH 里是旧版（本机 `node -v` = `v14.16.1`），
> 但 nvm / 这类运行环境里还装着新版 —— 程序自己会去找 ≥18 的 node。所以 `node -v <18` 时
> **先做第 5 步的 `node qh.mjs --version`**：能打出版本号 = 能跑，**不必装**；这一步也失败，才让用户装 Node 18+。

> 诚实口径：**AI 自己能跑 node ≠ 用户以后双击启动器能跑**。所以先认「系统 PATH / 启动器真能跑起来」。
> 但也不要把它写成前置条件去挡用户 —— 有 WorkBuddy 的机器很可能已经满足。

---

## 3. 下载 zip + 校验 sha256（**这步不许跳**）

```powershell
$zipName = $latest.zip
$zipPath = Join-Path $tmp $zipName
Invoke-WebRequest -UseBasicParsing "$base/$zipName" -OutFile $zipPath

$sha = (Get-FileHash -Algorithm SHA256 -LiteralPath $zipPath).Hash.ToLower()
Write-Output "期望 $($latest.sha256)"
Write-Output "实际 $sha"
```

**cmd 等价物**（没有 PowerShell 时）：`certutil -hashfile "<zip 路径>" SHA256`（输出大写，比较时**忽略大小写**）。

**怎么判断成了**：两串字符串**逐字相同**（大小写不算差异）。

**不相同 → 停**：不要解压、不要换个参数重试绕过。重新下载一次；仍不同就把两串哈希贴给用户（可能是代理/网络中间层改写了文件）。

---

## 4. 解压到默认目录（可以换盘）

> ⚠ **必须用下面这条 PowerShell `Expand-Archive`**。2026-10-30 两次真机实测：用 `unzip`（含 Git Bash 自带那个）
> 会把 `启动千川管家.cmd` 和 `千川任务执行表.xlsx` 解成乱码 —— 不是包坏了（zip 里这两个名字都带 UTF-8 标志位），
> 是解压工具忽略了标志位。事后改名也能救，但多一步就多一次出错机会。

```powershell
$bin = Join-Path $env:LOCALAPPDATA 'Programs\千川管家'
New-Item -ItemType Directory -Path $bin -Force | Out-Null
Expand-Archive -LiteralPath $zipPath -DestinationPath $bin -Force
```

**怎么判断成了**：解压后的**第一层**就有这些：

```
qh.mjs   package.json   version.json   启动千川管家.cmd
INSTALL.md   README.md   SKILL.md   千川任务执行表.xlsx   manifest.json
core\   platform\   console\   updater\   tools\   node_modules\
```

**中文文件名乱码怎么办**（解压工具忽略 UTF-8 标志位的锅，**不是包坏了**）：
- 名字乱码的 `.cmd` → 就是启动器，改回 `启动千川管家.cmd`；
- 名字乱码的 `.xlsx` → 就是任务表模板，改回 `千川任务执行表.xlsx`。
（我们的 zip 里这两个中文名都带 UTF-8 标志位，标准工具解出来是对的。）

**不要**解压到 `C:\Program Files`（普通权限写不了，以后升级会失败）。

**告诉用户装在哪**：把 `$bin` 的**绝对路径**记下来，汇报时要说（用户以后要找程序、备份、重装都靠它）。
如果你把程序装到了**别的地方**（不是默认的 `%LOCALAPPDATA%\Programs\千川管家`），**必须在汇报里写明装在哪**。

---

## 5. 版本判据：`node qh.mjs --version`

```powershell
Set-Location $bin
node qh.mjs --version
```

**怎么判断成了**：输出的版本号 = `latest.json` 里的 `version`。
不是 → 停（多半是解压到了旧包或错目录），不要继续。

---

## 6. 起控制台，并抓住带 token 的地址

```powershell
Set-Location $bin
node qh.mjs console --no-open
```

**你会看到**（黑窗口 / stdout）：

```
千川管家 v2 — 控制台（本机）
================================================================
程序版本：2.0.x
监听：http://127.0.0.1:8787   ★ 只绑 127.0.0.1，同一局域网的其它机器访问不到
token：<这次启动的随机口令>
概览页：http://127.0.0.1:8787/?token=<同一个口令>
```

**怎么判断成了**：出现 `概览页：http://127.0.0.1:8787/?token=…` 这一行。

**怎么取**（**不要猜 token、不要自己拼地址**）：
- 把控制台**在后台**起起来（你的工具支持后台任务就放后台；不支持就把 stdout 收到临时日志再轮询），
  等到 `概览页：` 出现，**整行**取出地址；
- 从这个地址里取 `token=` 后面的那一段，供下面调接口用；
- 取完**立刻删掉临时日志**（口令不要留在磁盘上，也不要写进仓库 / 群 / 文档）；
- 然后把概览页地址用默认浏览器打开：`Start-Process $overviewUrl`。

**注意**
- `--no-open` = 不自动打开浏览器（我们要自己拿到地址再开）。
- 如果报「端口 8787 已被占用」：说明**已经有一个控制台在跑**——不要重复起，先问用户 / 复用那个窗口；
  确实要换端口：`node qh.mjs console --port 51888 --no-open`，**后面所有地址都换成新端口**。
- 控制台是**常驻进程**：关掉它控制台就停了。别在装完就把它杀掉。

---

## 7. 报设备码（**用接口取，不要让用户读屏**）

把第 6 步的端口和 token 带进来：

```powershell
$consoleBase = 'http://127.0.0.1:8787'     # 换端口了就把这里改掉
$token = '<第 6 步拿到的 token>'
$st = Invoke-RestMethod -UseBasicParsing "$consoleBase/api/license/status?token=$token"
$st | ConvertTo-Json -Depth 4
```

**你会看到**（未授权时；下面是**字段示意**，`message` 的实际文字还会带本机 license.json 路径，以命令输出为准）：

```json
{ "ok": false, "code": "LICENSE_MISSING", "state": "missing",
  "machine": "QH-XXXX-XXXX-XXXX", "machinePartial": false,
  "message": "这台机器未授权：… —— 这台机器还没导入授权。把设备码 QH-XXXX-XXXX-XXXX 发给提供方；收到授权文件后到「授权」页粘贴或上传。" }
```

**怎么判断成了**
- `machine` 是 `QH-` 开头的 12 位设备码（`QH-XXXX-XXXX-XXXX`）；
- 未授权时 `ok=false`、`code=LICENSE_MISSING` —— 这是**正常状态**，不是装坏了；
- `machine` 是 `null` → 看 `machineError`，把原样报给用户（设备码读不出来），**不要自己编一个**。

**把设备码原样报给用户**，一句话说清下一步：把这一串发给提供方。

---

## 8. 授权（**人往返**；AI 不许假装能签）

流程固定三步，一步都不能替：

1. **用户**把设备码发给提供方；
2. **提供方**签发 `license.json`（或一段 `{` 开头的文本）发回用户；
3. **用户在授权页导入**（粘贴或上传都行）：
   - 打开授权页：`Start-Process "$consoleBase/license?token=$token"`；
   - 页面上填/选，点「导入粘贴的文本」或「导入选中的文件」；
   - 成功后顶部红条变绿条「已授权至 YYYY-MM-DD」，**立刻生效，不用重启**。

导入完**重新查一次**：

```powershell
$st2 = Invoke-RestMethod -UseBasicParsing "$consoleBase/api/license/status?token=$token"
$st2.ok; $st2.code; $st2.message; $st2.expiresAt
```

**怎么判断成了**：`ok=true`、`code=OK`、`message` 形如「已授权至 2027-10-29」。

### 8.1 看到 `LICENSE_MISSING` **不要重试**

未授权时（**装完本来就是这样**）：

| 你跑什么 | 会看到 |
|---|---|
| `node qh.mjs run --excel "<任务表路径>"`（**连试运行也拒**） | 配置还没填时先提示「还没有个人配置…先跑 console」（退出码 1）；配置填好、授权没导入时是 `错误码：LICENSE_MISSING` + 本机设备码 + 去授权页的提示（退出码 1） |
| 控制台「跑任务表」点真执行（`POST /api/task/run`） | HTTP **403** + `code=LICENSE_MISSING` + `machine` + `licensePage=/license` |

**正确动作只有一个**：让用户去授权页导入授权。**不要**重试、不要换参数、不要找「放行开关」（没有这种开关）。

> 2026-10-30 我自己在干净目录实测：控制台 `POST /api/task/run` → **403 / LICENSE_MISSING**；无 token 同一个接口 → **401 / UNAUTHORIZED**；
> 全新目录跑 `run --dry-run` → 配置闸门先拒（退出码 1）。三种都是「不能动账户」，不是装坏了。

### 8.2 你签不了，也不用找

签发工具和私钥**只在提供方机器上**；用户这台机器上「签发授权」页（`/issue`）**路由根本不存在**（404，不是权限问题）。
不要假装能签、不要教用户去跑签发命令（他机器上没有私钥，跑也签不出来）。

---

## 9. 装技能（把 `SKILL.md` 交给对话前端）

包里带 `SKILL.md`（**第一层就有**）。它开头有三行元数据（`name` / `description` / `description_zh`，照 v1 的做法），
**这是它被识别成技能的前提，不要删这三行**；里面的命令都要求在**程序安装目录**里跑（用户问装在哪，报第 4 步的路径）。

**路径 A · DSH（DeepSeek Harness）**：复制到全局技能目录

```powershell
$dst = Join-Path $env:USERPROFILE '.dsh\skills\qianchuan-helper'
New-Item -ItemType Directory -Path $dst -Force | Out-Null
Copy-Item -LiteralPath (Join-Path $bin 'SKILL.md') -Destination (Join-Path $dst 'SKILL.md') -Force
Test-Path (Join-Path $dst 'SKILL.md')
```

**怎么判断成了**：`Test-Path` 返回 `True`，且目标文件内容与包里那份一致。

**路径 B · WorkBuddy（或别的对话前端）**：
1. **先让它自己报技能目录**：问它「你的技能 / 插件目录在哪？路径是什么？」；
2. 它报得出来 → 把 `SKILL.md` 放进去（按它说的结构）；
3. 它报不出来、或它不认这个格式 → **退回最稳的办法**：每次新会话把 `SKILL.md` 的内容发给它（或让它读安装目录里的 `SKILL.md`）。

> **诚实边界**：WorkBuddy「能跑命令、能读本机文件」**已实测**（2026-10-30）；「能自动加载这个技能」**还没验证**。
> 所以**不要向用户保证**「装完就能被触发语唤起」——只说「按这条做；触发不了就用路径 B 的第 3 条」。

**后续新会话怎么用**
- 技能放进**全局**技能目录后，**新会话自动可见**，用户直接说触发语即可：
  `打开控制台` / `跑一下千川` / `建个计划` / `改计划` / `改预算` / `改投产` / `补素材` / `查余额` / `有阻碍吗` / `更新一下`。
- **升级只换程序目录，不动技能文件**；如果新版本的 `SKILL.md` 有变化（更新说明会写），再复制一次覆盖。

---

## 10. 「打开控制台」引导（给用户用的话术）

- **控制台是你在第 6 步起的**：你手上有这次的口令 → 直接把概览页地址打开给他看；告诉他「地址每次启动都会变，别当书签存」。
  这时**不要**再让他双击 `启动千川管家.cmd`（会提示端口被占用）。
- **以后用户自己说「打开控制台」**（新会话、你不知道口令）：按 `SKILL.md` §3 引导 ——
  让他双击程序目录里的 `启动千川管家.cmd`，看黑窗口里那行 `概览页：http://127.0.0.1:8787/?token=…`；
  黑窗口已经开着就让他看那行；找不到就关掉重开一次。
  **你不知道口令时，不许猜、不许自己拼地址。**

---

## 11. 冒烟（装完必须跑，三步）—— **跳过就不算装好**

```powershell
# ① 控制台活着：/ 跟随后 200（新机器首启会 302 到配置向导，正常）
curl.exe -s -L -o NUL -w "%{http_code}" "$consoleBase/?token=$token"
#    再核一个 JSON：/api/status 要 ok=true、version 对
Invoke-RestMethod -UseBasicParsing "$consoleBase/api/status?token=$token"

# ② 授权页打开是 200，页面上有设备码（未授权/已授权都算过）
Invoke-WebRequest -UseBasicParsing "$consoleBase/license?token=$token"

# ③ 环境自检（必需项必须全过）
Set-Location $bin
node qh.mjs doctor
```

**怎么判断成了**
1. ① `/` 跟随后 **200**；`/api/status` 返回 JSON 且 `ok=true`、`version` 是最新版；
2. ② `/license` **200**，正文里有「设备码」（状态是未授权还是已授权都算过，只要能读出来）；
3. ③ 输出里有 `必需项：8/8 通过`（或「✔ 必需项全部通过」），退出码 0。

> **「装好了」的判据**（比「命令跑完了」更严）：**控制台能开 + 授权页能显示设备码 + 导入授权后红条变绿**。
> 只把程序文件解压完**不等于**装好了。

**常见误判**
- ① 看到 `302 → /setup`：那是**新机器还没有个人配置**，正常（跟随后 200 即过）。
- ③ 里「计划文档（00/01/02）缺 3 份」是**正常的**（发布包不带 `docs\`，属非必需项），不要当故障。
- 任一步不过：把**原始输出**贴给用户，不要自己编原因、不要反复重试。

---

## 12. 汇报（对用户就四句话）

1. **装在哪**：`<安装目录绝对路径>`，版本 `<node qh.mjs --version 的输出>`；
2. **控制台**：`<这次带 token 的概览页地址>`（说明每次启动都会变）；
3. **设备码**：`QH-XXXX-XXXX-XXXX` + 下一步（发给提供方；拿到授权后在授权页粘贴/上传）；
4. **技能**：哪条路装上了；没装上就说明「每次新会话把 `SKILL.md` 内容发给我」也能用。

再加一句边界的实话：**授权前只能看不能动**（控制台页面 / `doctor` / `--version` 都能用；`run` 连试运行都会被拒）。

---

## 13. 出问题怎么办

| 现象 | 怎么办 |
|---|---|
| `where node` 找不到 / 版本 <18 | 让用户装 Node 18+（nodejs.org 或 `winget install OpenJS.NodeJS.LTS`），**新开窗口**复查 |
| 下载 404 | 用 `latest.json` 里的 `zip` 字段名拼地址；确认 Release 里有这个附件；别下 `Source code` 压缩包 |
| sha256 不一致 | 停。重下一次；仍不一致就把两串哈希贴给用户（中间层可能改写文件） |
| 解压后中文名乱码 | 按扩展名认：`.cmd`→`启动千川管家.cmd`、`.xlsx`→`千川任务执行表.xlsx`。**包是合规的**，不是损坏 |
| 端口 8787 被占用 | 已经有一个控制台在跑 → 不要重复起；或换端口 `--port 51888`（后面地址全换） |
| 浏览器 401 / token 错 | 控制台重启过 → 旧地址作废；用**本次** stdout 里那行 `概览页：…` 的地址，别用旧的 |
| 页面红条「这台机器未授权」 | 正常。按第 8 步走：报设备码 → 等授权 → 导入 |
| `run` 报 `LICENSE_MISSING` | **不要重试**。先导入授权（第 8 步）；授权前不许说「已经能跑了」 |
| `/issue` 打不开（404） | 正常。签发页只在提供方机器上存在 |
| `node qh.mjs console --version` 报错 | 本来就**不存在**这个用法。版本判据是 `node qh.mjs --version` |
| 控制台起不来 | 把黑窗口 / stdout 的原始报错贴给用户；先跑 `node qh.mjs doctor` 看缺什么 |

---

## 附录 A：`docs/02`「T6.9 必写清单」13 条 → 本文哪一节

| # | 必写内容 | 落在本文 |
|---|---|---|
| 1 | 检查系统 PATH 里的 node（`where node` + `node -v` ≥18）；通过就继续，失败才装 | **第 2 节** |
| 2 | 默认装到 `%LOCALAPPDATA%\Programs\千川管家`，并把路径告诉用户 | **第 4 节**（路径写入汇报，见第 12 节） |
| 3 | 中文文件名乱码怎么办（`*.cmd` / `*.xlsx` 按扩展名认并改名） | **第 4 节** |
| 4 | 版本判据 `node qh.mjs --version`（不是 `console --version`） | **第 0 节末 + 第 5 节** |
| 5 | AI 取 token：后台 `console --no-open` + 抓 stdout 的 `概览页：…?token=…` | **第 6 节** |
| 6 | 设备码用 `/api/license/status` 的 `machine` 字段 | **第 7 节** |
| 7 | 授权仍人往返；AI 不许假装能签 | **第 8 节** |
| 8 | 装完冒烟（`/` 200、`/license` 状态、`doctor` 必需项） | **第 11 节**（① 控制台 ② 授权状态 ③ doctor） |
| 9 | 技能元数据头（`name` / `description` / `description_zh`） | **第 9 节**（「开头有三行元数据…不要删」） |
| 10 | 技能装到哪：DSH 全局技能目录；WorkBuddy 让 AI 自己报目录，拿不到就退回贴内容 | **第 9 节**（路径 A / 路径 B） |
| 11 | 程序目录定位：所有命令在安装目录里跑，不知道就问用户 | **第 9 节**（并见包内 `SKILL.md` 文件头） |
| 12 | 后续新会话：技能在全局目录 → 新会话自动可见 → 说触发语即可；升级只换程序 | **第 9 节末**（「后续新会话怎么用」） |
| 13 | 未授权也能装、但不能执行；看到 `LICENSE_MISSING` 不重试、不说「已经能跑了」 | **第 8.1 节**（+ 第 11 节的「装好了」判据） |

**「装好了」的判据**（比「命令跑完了」更严）：**控制台能开 + 授权页能显示设备码 + 导入授权后红条变绿**。
只到前三步之前（程序文件解压完）**不等于**装好了。

## 附录 B：这个安装过程**不做**的事（安全边界）

- 不跑 `node qh.mjs run …` 真执行（那是动账户）；不点「发布计划」；不改用户账户 / 配置。
- 不读、不打印用户的任何密钥（AdsPower Key / 飞书凭据）。
- 不签发授权、不伪造 license、不把 `license.json` 发到群里。
- 不把本次的 token 写进仓库 / 文档 / 群消息；取完就删临时日志。
- 公开仓库里**只有两个附件**（zip + `latest.json`）：没有源码、没有配置、没有私钥、没有授权、没有数据。
