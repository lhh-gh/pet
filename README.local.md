# gin-vue-admin 本地启动与修改后运行指南（SQLite）

- 前端：http://127.0.0.1:8080
- 后端健康检查：http://127.0.0.1:8888/health
- Swagger：http://127.0.0.1:8888/swagger/index.html
- 初始管理员：`admin` / `123456`
- 数据库：`server/gva.db`，已完成初始化。
- 后端配置：`server/config.local.yaml`（本机文件，已被 Git 忽略）。
- 本地配置关闭了 Redis 和 MongoDB，不需要启动 Docker 或其他数据库服务。

本文中的项目就是 `gin-vue-admin`。以下命令适用于 Windows PowerShell，项目路径为 `E:\go\pet`；换机器时请替换路径。前端和后端分别占用一个终端，启动命令持续运行属于正常状态。

当前机器已经完成首次安装和数据库初始化，可以直接使用“以后每天启动”或“每次修改后怎么运行”。首次准备全新环境时再执行第一次启动流程。

MySQL、Redis 的字段说明、首次初始化、Docker 连接和故障排查见 [MySQL 与 Redis 配置指南](docs/mysql-redis-configuration.md)。使用 MySQL 配置时，将本文启动参数换为已经准备好的 `-c config.mysql.local.yaml`，并先启动对应依赖服务。

## 命令速查

业务开发见 [API 编写与 Vue 页面接入详细指南](docs/api-vue-development-guide.md)，包含从 Model 到页面的增删改查示例、路由注册、权限配置和联调步骤。

| 场景 | 后端 | 前端 |
| --- | --- | --- |
| 第一次准备环境 | 下载 Go 依赖、准备配置、编译并启动；空库执行初始化 | `npm.cmd ci --no-audit --no-fund`，再 `npm.cmd run serve -- --strictPort` |
| 关机后重新启动，代码未修改 | `.\gva.local.exe -c config.local.yaml` | `npm.cmd run serve -- --strictPort` |
| 修改 Go 源码 | 停止旧后端，`go build -o gva.local.exe .`，成功后重新启动 | 已运行时继续使用 |
| 修改 Vue 页面、样式、普通 JS | 未改后端时继续使用 | 保持 Vite 运行，保存文件后通常自动更新 |
| 修改后端数据库、Redis 或监听配置 | 停止后端，用同一配置重新启动 | 如代理目标也改变，需要同步配置并重启前端 |
| 修改前端环境变量、代理或依赖 | 按后端是否发生变化处理 | 重新启动；依赖变化时先安装依赖 |

表中命令分别在 `server/`、`web/` 目录执行，完整命令见下面各节。

## 第一次启动：新机器或全新项目副本

### 1. 确认运行环境

```powershell
cd E:\go\pet
go version
node --version
npm.cmd --version
```

- 后端 `server/go.mod` 要求 Go 1.24.0 以上，并声明工具链 Go 1.24.2。本机基础 Go 是 1.21.9，进入后端目录构建时会按工具链设置下载对应版本。
- 前端使用 Vite 8，Node.js 需要 20.19+ 或 22.12+ 等受支持的新版本；本机已使用 Node.js 24.9.0、npm 11.6.0 验证启动。
- 本机全局 `GOSUMDB=off` 会阻止 Go 工具链校验，下面只在当前 PowerShell 会话临时覆盖为 `sum.golang.org`，不修改全局配置。
- 命令找不到时，先安装相应运行环境并重新打开终端，再继续后续步骤。

### 2. 准备 SQLite 本地配置

本地配置被 Git 忽略，所以新机器可能没有 `server/config.local.yaml`。只在文件不存在时从完整模板复制：

```powershell
cd E:\go\pet\server
if (-not (Test-Path -LiteralPath .\config.local.yaml)) {
    Copy-Item -LiteralPath .\config.yaml -Destination .\config.local.yaml
}
```

**只有准备全新的 SQLite 环境时**，才编辑新文件并合并以下字段。当前机器已有数据库和配置，不要把现有库名清空，也不要覆盖现有文件：

```yaml
system:
    db-type: sqlite
    addr: 8888
    use-redis: false
    use-mongo: false
    use-multipoint: false
    disable-auto-migrate: false

sqlite:
    path: "."
    db-name: ""
    max-idle-conns: 10
    max-open-conns: 100
    conn-max-lifetime: 0
    log-mode: "error"
    log-zap: false
```

这是局部片段，保留原文件中的 JWT、日志、上传等其他完整配置，并修改原有 `system:`、`sqlite:` 段，避免重复顶层键。空 `sqlite.db-name` 用于让全新环境进入初始化流程。

### 3. 后端：下载依赖、编译、启动

在第一个 PowerShell 终端中运行：

```powershell
cd E:\go\pet\server
$env:GOSUMDB = 'sum.golang.org'
go mod download
if ($LASTEXITCODE -ne 0) { throw 'Go 依赖下载失败，请修复上方错误后重试。' }
go build -o gva.local.exe .
if ($LASTEXITCODE -ne 0) { throw '后端编译失败，本次不启动。' }
.\gva.local.exe -c config.local.yaml
```

构建成功后会生成 `server/gva.local.exe`，后端监听 `8888`。启动前确保没有旧后端占用这个端口。

### 4. 前端：安装依赖、启动

另开第二个 PowerShell 终端运行：

```powershell
cd E:\go\pet\web
npm.cmd ci --no-audit --no-fund
if ($LASTEXITCODE -ne 0) { throw '前端依赖安装失败，请修复上方错误后重试。' }
npm.cmd run serve -- --strictPort
```

本仓库已有 `web/package-lock.json`，使用 `npm ci` 按锁文件安装。如果全新副本没有锁文件，改用 `npm.cmd install --no-audit --no-fund` 生成锁文件后再启动。

`npm ci` 要求 `package.json` 和锁文件一致，否则会报错；它会重建已有依赖目录。只改页面时不需要重复执行。使用 `npm.cmd` 可以避免 PowerShell 对 `npm.ps1` 的执行策略限制。

前端运行地址为 [http://127.0.0.1:8080](http://127.0.0.1:8080)。`--strictPort` 让 `8080` 被占用时明确报错，避免自动换端口后访问错地址。

### 5. 只为全新数据库执行一次初始化

两个服务都启动后，打开 [数据库初始化页面](http://127.0.0.1:8080/#/init)，点击“我已确认”并填写：

| 字段 | SQLite 本地填写方式 |
| --- | --- |
| 管理员密码 | 为 `admin` 设置密码，至少 6 位；当前已初始化环境的初始密码为 `123456` |
| 数据库类型 | `sqlite` |
| dbName | `gva`，不带 `.db` 后缀 |
| dbPath | `.`，表示后端当前工作目录 `server/` |

SQLite 不需要填写 MySQL 的 host、port、userName、password。点击“立即初始化”，成功后会创建 `server/gva.db`、写入基础数据，并把配置回写到 `config.local.yaml`。

初始化完成后，先在后端终端按 `Ctrl+C` 停止，再运行：

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.local.yaml
```

打开 [登录页面](http://127.0.0.1:8080/#/login)，使用 `admin` 和刚设置的管理员密码。以后启动或修改代码时都不需要重复初始化数据库。

## 以后每天启动：代码未修改

后端终端：

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.local.yaml
```

前端终端：

```powershell
cd E:\go\pet\web
npm.cmd run serve -- --strictPort
```

依赖和可执行文件都已准备好时，不需要再次安装依赖或重新编译。两个终端保持打开；退出时在各自终端按 `Ctrl+C`。仍在运行的服务无需重复启动。

## 每次修改后怎么运行

### 1. 修改后端 Go 文件：重新编译并启动

适用于 API、Service、Router、Model、插件或其他 `.go` 文件修改。先在旧后端终端按 `Ctrl+C`，等进程退出后执行：

```powershell
cd E:\go\pet\server
$env:GOSUMDB = 'sum.golang.org'
go build -o gva.local.exe .
if ($LASTEXITCODE -ne 0) { throw '后端编译失败，请修复错误后再启动。' }
.\gva.local.exe -c config.local.yaml
```

只运行旧的 `gva.local.exe` 不会包含刚修改的 Go 代码。本启动方式没有开启 Go 自动重载；构建失败时也不要继续运行旧文件来验证新功能。普通源码修改通常无需再次执行 `go mod download`。

如果前后端都修改了，完成后端重启后，前端保存源码即可按热更新方式验证。新增或变更接口时，还要核对前端请求参数和响应字段。

### 2. 只修改后端 YAML：重新启动已有程序

修改数据库、Redis、监听地址等配置后，先按 `Ctrl+C` 停止后端，再执行：

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.local.yaml
```

单纯修改 YAML 不需要重新编译。项目虽然监听配置文件变化，但现有数据库、Redis 连接和监听端口仍应通过重启重新创建。务必修改 `-c` 指定的实际文件。

### 3. 修改前端页面、样式或普通 JS：保存即可

前端开发服务已经运行时，修改 `web/src/` 中的 `.vue`、样式、接口调用或普通 JS 文件，保存后 Vite 通常会热更新或刷新页面。根据页面状态，必要时手动刷新浏览器验证。

前端服务没运行时，启动命令是：

```powershell
cd E:\go\pet\web
npm.cmd run serve -- --strictPort
```

日常开发不需要每次执行 `npm install` 或 `npm run build`。`build` 用于生成发布目录 `web/dist/`，不是页面保存后查看修改的必需步骤。

### 4. 修改前端环境变量、代理或构建配置：重启前端

修改 `web/.env.development`、其他正在使用的 `.env` 文件、`web/vite.config.js` 或启动脚本后，建议按 `Ctrl+C` 停止前端，再执行：

```powershell
cd E:\go\pet\web
npm.cmd run serve -- --strictPort
```

后端端口改变时，同步更新前端开发环境中的 `VITE_SERVER_PORT`；当前值为 `8888`。前端端口由 `VITE_CLI_PORT` 控制，当前值为 `8080`。

### 5. 前端依赖改变：先安装，再启动

拿到已经同步好的 `package.json` 和 `package-lock.json` 后，停止前端再运行：

```powershell
cd E:\go\pet\web
npm.cmd ci --no-audit --no-fund
if ($LASTEXITCODE -ne 0) { throw '前端依赖安装失败，本次不启动。' }
npm.cmd run serve -- --strictPort
```

如果是自己主动修改了 `package.json`，需要同步更新锁文件，则将 `npm ci` 换为 `npm.cmd install --no-audit --no-fund`，确认安装成功再启动。新增 Go 依赖则按实际依赖变更处理 `go.mod`、`go.sum`，之后仍按第 1 种方式重新构建后端。

## 可选：用 go run 开发后端

不想手动命名输出文件时，可以先停止旧后端，改用下面方式编译并运行当前源码：

```powershell
cd E:\go\pet\server
$env:GOSUMDB = 'sum.golang.org'
go run . -c config.local.yaml
```

修改 Go 代码后仍需按 `Ctrl+C`，重新执行该命令；它也不提供自动重载。`go run` 使用临时构建产物，不会更新已有 `gva.local.exe`。两种方式选一种运行即可；后续切回 `.exe` 时，应先用最新源码重新构建，避免运行旧版本。

## 启动后检查

在第三个 PowerShell 终端中执行：

```powershell
Invoke-RestMethod -Uri 'http://127.0.0.1:8888/health'
Invoke-RestMethod -Uri 'http://127.0.0.1:8080/api/health'
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8888/init/checkdb' -ContentType 'application/json' -Body '{}'
```

前两个请求预期返回 `ok`。已配置数据库时，第三个请求应返回 `code: 0`、`data.needInit: false`；还需登录并检查用户信息、菜单与实际修改的功能。

如果启动提示端口占用，先检查是否已经有服务运行。可查看监听进程：

```powershell
Get-NetTCPConnection -State Listen -LocalPort 8080,8888 -ErrorAction SilentlyContinue |
    Select-Object LocalAddress, LocalPort, OwningProcess
```

确认归属后，到原来的服务终端停止对应进程，不要因为端口占用就随意结束其他项目的服务。当前数据库已经初始化，重启和重新编译不会要求删除或重建 `gva.db`。

## 命令依据

- [后端模块与工具链](server/go.mod)、[后端入口](server/main.go)、[配置加载](server/core/viper.go)。
- [前端启动脚本](web/package.json)、[Vite 配置](web/vite.config.js)、[开发环境变量](web/.env.development)。
- [初始化页面](web/src/view/init/index.vue)、[SQLite 文件路径](server/config/gorm_sqlite.go)。
- 官方参考：[Vite 开发指南](https://vite.dev/guide/)、[Vite 环境变量](https://vite.dev/guide/env-and-mode)、[npm ci](https://docs.npmjs.com/cli/v11/commands/npm-ci)。

## 验证记录

- 2026-10-22：后端编译成功，Vite 开发服务启动成功。
- 通过前端代理请求 `/api/health` 返回 `ok`。
- `/api/init/checkdb` 返回 `code: 0`、`needInit: false`。
- 已观察到成功登录以及用户信息、菜单、媒体库列表接口成功返回，后台页面可以正常显示。
- 媒体库自带的两个演示图片使用外部网址，当前预览显示加载失败；本地列表接口正常。
