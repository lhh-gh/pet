# gin-vue-admin：MySQL 与 Redis 配置指南

适用项目：本仓库 `server/`（Go + Gin）和 `web/`（Vue + Vite）。核对日期：2026-10-22。本文按当前仓库源码说明配置行为；后续升级框架时应重新核对对应实现。

当前本地环境已经使用 **SQLite** 完成初始化和登录验证，配置文件是 `server/config.local.yaml`，数据库文件是 `server/gva.db`。本文提供 MySQL、Redis 的配置方法，编写文档本身没有切换数据库或启动这些服务。原有启动方式见 [本地运行说明](../README.local.md)。

## 目录

- [1. 先选择需要的运行方式](#1-先选择需要的运行方式)
- [2. 配置文件在哪里，怎样加载](#2-配置文件在哪里怎样加载)
- [3. MySQL 配置项详解](#3-mysql-配置项详解)
- [4. 第一次使用 MySQL：完整初始化流程](#4-第一次使用-mysql完整初始化流程)
- [5. 已有 GVA 数据库：直接连接](#5-已有-gva-数据库直接连接)
- [6. Redis 配置与启用方式](#6-redis-配置与启用方式)
- [7. 可选：用 Docker 运行 MySQL 和 Redis](#7-可选用-docker-运行-mysql-和-redis)
- [8. 启动与验证](#8-启动与验证)
- [9. SQLite 切换、数据保留与切回](#9-sqlite-切换数据保留与切回)
- [10. 常见报错排查](#10-常见报错排查)
- [11. 配置与数据的维护](#11-配置与数据的维护)
- [12. 源码和参考文档](#12-源码和参考文档)

## 1. 先选择需要的运行方式

MySQL 和 SQLite 都用于存放用户、菜单、角色、业务表等关系数据，主数据库由 `system.db-type` 选择。Redis 是独立服务，可用于登录状态或业务主动实现的缓存；使用 Redis 不要求把 SQLite 换成 MySQL。

| 需求 | `system.db-type` | `system.use-redis` | `system.use-multipoint` |
| --- | --- | --- | --- |
| 继续使用当前 SQLite 本地环境 | `sqlite` | `false` | `false` |
| SQLite 加 Redis，供业务调用 | `sqlite` | `true` | `false` |
| MySQL，不启用 Redis | `mysql` | `false` | `false` |
| MySQL 加 Redis，供业务调用 | `mysql` | `true` | `false` |
| 启用登录拦截，同账号新登录使旧令牌失效 | `sqlite` 或 `mysql` | `true` | `true` |

本版本不能只把 `use-multipoint` 设为 `true` 而保持 `use-redis: false`，因为对应登录流程会调用 Redis 客户端。

如果只是继续本地开发，沿用现有 SQLite 配置即可。需要体验 MySQL 时，建议新建独立配置文件，完成新数据库的初始化。

## 2. 配置文件在哪里，怎样加载

### 2.1 加载顺序

后端配置选择顺序如下，前面的优先级更高：

1. 启动命令的 `-c` 参数。
2. 环境变量 `GVA_CONFIG` 指定的文件路径。
3. 按 Gin 模式选择 `config.debug.yaml`、`config.release.yaml` 或 `config.test.yaml`。
4. 第 3 步对应文件不存在时，回退到 `config.yaml`。

`config.local.yaml`、`config.mysql.local.yaml` 都不会因为文件名包含 `local` 就自动生效。显式指定的 `-c` 或 `GVA_CONFIG` 文件不存在时会报错，不会回退到默认配置。

建议始终在 `server/` 目录启动，避免配置、上传目录以及 SQLite 文件的相对路径指向其他位置：

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.local.yaml
```

也可以在当前 PowerShell 会话指定配置路径：

```powershell
cd E:\go\pet\server
$env:GVA_CONFIG = 'E:\go\pet\server\config.mysql.local.yaml'
.\gva.local.exe
```

`GVA_CONFIG` 在这里用于指定配置文件。本项目加载逻辑没有把 `MYSQL_HOST`、`REDIS_PASSWORD` 等任意环境变量自动映射到 YAML 字段，也没有配置通用的 `${变量}` 插值。

### 2.2 修改配置的约定

- 下文除 Docker Compose 示例外，YAML 都是**局部配置片段**，需要合并到已有文件的对应位置，不能拿来覆盖完整配置文件。
- 同一文件只保留一个顶层 `system:`、一个 `mysql:`、一个 `redis:`，不要把示例直接追加到文件末尾形成重复键。
- 缩进使用空格；密码和端口建议加引号，避免特殊字符或类型歧义。
- 修改数据库、Redis 地址或开关后，停止并重新启动后端，让连接重新建立。配置文件监听不会自动重建已有客户端。
- SQLite 配置可以继续保留；当 `db-type: mysql` 时，主数据库选择 MySQL。

`server/*.local.yaml` 已在 `.gitignore` 中排除，但忽略规则不能保护已被跟踪的文件。不要把真实连接密码写进公共文档或提交到仓库。

## 3. MySQL 配置项详解

下面是**已经完成 GVA 初始化的 MySQL 数据库**的连接片段。若目标数据库是空库，请先按第 4 节操作。

```yaml
system:
    db-type: mysql
    use-redis: false
    use-multipoint: false
    disable-auto-migrate: false

mysql:
    path: "127.0.0.1"
    port: "3306"
    db-name: "gva"
    username: "gva"
    password: "替换为实际MySQL密码"
    config: "charset=utf8mb4&parseTime=True&loc=Local"
    engine: "InnoDB"
    prefix: ""
    singular: false
    max-idle-conns: 10
    max-open-conns: 100
    conn-max-lifetime: 0
    log-mode: "error"
    log-zap: false
```

| 字段 | 含义与填写方法 |
| --- | --- |
| `system.db-type` | 主数据库类型，MySQL 填 `mysql`，SQLite 填 `sqlite`。 |
| `path` | MySQL 主机名或 IP，如 `127.0.0.1`；不带 `http://`、`mysql://` 或端口。 |
| `port` | MySQL 端口，按字符串填写，如 `"3306"`；Docker 映射后可能是 `"13306"`。 |
| `db-name` | 数据库名，如 `gva`。首次进入初始化流程前留空；正常运行时填写实际库名。 |
| `username` / `password` | MySQL 账号和密码，与 GVA 登录账号 `admin` 的密码是两套凭据。 |
| `config` | 驱动连接参数。示例使用 UTF-8 四字节字符集、解析时间字段和本地时区。 |
| `engine` | 表引擎，明确填写 `InnoDB`；首次初始化还应确保 MySQL 默认引擎为 InnoDB。 |
| `prefix` / `singular` | 表名前缀和单数表名策略。已有数据库应沿用原值，否则可能访问另一套表。 |
| `max-idle-conns` | 连接池最大空闲连接数，示例为 `10`。 |
| `max-open-conns` | 最大打开连接数，示例为 `100`；多后端实例需结合 MySQL 总连接上限调整。 |
| `conn-max-lifetime` | 连接最长复用时间，单位秒；`0` 表示不限制生命周期。 |
| `log-mode` | 支持 `silent`、`error`、`warn`、`info`；空值或无法识别的值按 `info` 处理。 |
| `log-zap` | 是否通过项目 Zap 日志组件输出 GORM 日志。 |
| `system.disable-auto-migrate` | `true` 跳过启动时自动迁移，`false` 允许运行自动表结构迁移。 |

当前源码按下面的结构拼接 DSN，理解它有助于排查地址或连接参数错误：

```text
用户名:密码@tcp(主机:端口)/数据库名?charset=utf8mb4&parseTime=True&loc=Local
```

自动迁移可能创建或调整表结构，但**不会代替首次初始化中的管理员、菜单、角色和权限数据写入**。生产库的结构变更应按项目迁移流程执行，再决定是否禁用自动迁移。

## 4. 第一次使用 MySQL：完整初始化流程

### 4.1 准备一个可连接的 MySQL 服务

可使用已安装的本机 MySQL、现有远程 MySQL，或者第 7 节的 Docker 示例。先确认主机、端口、账号、密码以及目标库名。

初始化会执行 `CREATE DATABASE IF NOT EXISTS`、建表并写入初始数据，账号需要目标库范围内相应的建库、建表和写入权限。可以先由管理员创建专用开发库及账号。例如，在 MySQL 管理客户端中执行：

```sql
CREATE DATABASE IF NOT EXISTS `gva`
  DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'gva'@'localhost' IDENTIFIED BY '替换为开发数据库密码';
GRANT ALL PRIVILEGES ON `gva`.* TO 'gva'@'localhost';
```

这段 SQL 适用于尚未创建该账号的本机开发环境。已有账号应核对现有授权；远程或容器连接时，账号的 MySQL `Host` 匹配范围需要与实际客户端来源一致。第 7 节的 MySQL 镜像会自动创建示例库和专用账号，无需重复执行上述 SQL。

### 4.2 新建独立配置，保留 SQLite 环境

先停止当前后端，释放 `8888` 端口。然后在 PowerShell 中复制已有配置；下面的检查可避免覆盖之前保存的 MySQL 配置：

```powershell
cd E:\go\pet\server
if (Test-Path -LiteralPath .\config.mysql.local.yaml) {
    throw 'config.mysql.local.yaml 已存在，请直接编辑现有文件。'
}
Copy-Item -LiteralPath .\config.local.yaml -Destination .\config.mysql.local.yaml
```

编辑新文件，合并以下字段，其他完整配置继续保留：

```yaml
system:
    db-type: mysql
    use-redis: false
    use-multipoint: false
    disable-auto-migrate: false

mysql:
    path: "127.0.0.1"
    port: "3306"
    db-name: ""
    username: "gva"
    password: "替换为实际MySQL密码"
    config: "charset=utf8mb4&parseTime=True&loc=Local"
    engine: "InnoDB"
    max-idle-conns: 10
    max-open-conns: 100
    conn-max-lifetime: 0
    log-mode: "error"
    log-zap: false
```

这里最关键的是 **`mysql.db-name: ""`**。当前实现看到空库名时会暂不建立主数据库连接，让前端进入初始化流程。初始化页面提交的 host、port、userName、password、dbName 才是此次建库所用的参数；页面不会自动读取你刚编辑的连接字段。

如果提前填入非空库名，后端会直接尝试连接：数据库不存在可能导致启动失败；数据库存在但没有种子数据，则可能只有表而不能正常登录。

### 4.3 启动后端并填写初始化页面

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.mysql.local.yaml
```

前端未运行时，另开 PowerShell：

```powershell
cd E:\go\pet\web
npm.cmd run serve -- --strictPort
```

打开 [初始化页面](http://127.0.0.1:8080/#/init)，点击“我已确认”，再填写：

| 页面字段 | 本机 MySQL 示例 | 使用第 7 节 Docker 示例 |
| --- | --- | --- |
| 管理员密码 | 为 GVA 的 `admin` 账号设置密码，至少 6 位 | 同左，与数据库密码独立 |
| 数据库类型 | `mysql` | `mysql` |
| host | `127.0.0.1` | `127.0.0.1` |
| port | `3306` | `13306` |
| userName | `gva` 或实际账号 | `gva` |
| password | 该 MySQL 账号的密码 | `gva_mysql_local_123456` |
| dbName | `gva` 或新开发库名 | `gva` |

点击“立即初始化”。成功后，程序会创建数据库和表、写入基础数据，并把连接信息写回**本次实际加载的配置文件**，这里是 `config.mysql.local.yaml`。同时会生成新的 JWT 签名密钥，旧登录令牌可能失效。

初始化回写会重新生成 MySQL 配置字段；完成后检查 `mysql.engine`，必要时补为 `InnoDB`，并检查端口和库名。停止后端，再用同一条启动命令重新启动，打开 [登录页面](http://127.0.0.1:8080/#/login)，使用 `admin` 和页面中设置的管理员密码登录。

### 4.4 初始化失败时

查看后端具体错误，修复地址、权限或数据库服务后再尝试。如果初始化已经部分写入，当前进程可能持有数据库句柄，再次提交时提示“已存在数据库配置”；此时先停止后端，确认配置和数据库状态，再重新启动。

不要把正在使用的业务库当作实验初始化目标。部分初始化的开发库也应先确认现有数据，再选择恢复或换用新库名。

## 5. 已有 GVA 数据库：直接连接

当 MySQL 中已经有本项目匹配版本的表、管理员及权限数据时：

1. 准备独立的 `config.mysql.local.yaml`，保留完整项目配置。
2. 按第 3 节填写连接参数，`mysql.db-name` 填实际库名。
3. 保持已有的 `prefix`、`singular` 及迁移策略一致。
4. 使用 `-c config.mysql.local.yaml` 启动，验证登录、用户信息和菜单。

不要再次执行首次初始化接口。当前接口在 `global.GVA_DB` 非空时会直接拒绝初始化。

如果能连上但没有管理员或菜单，应检查是不是连到了空库、错库或另一套命名规则的表。`needInit: false` 只能说明后端已经持有主数据库句柄，不能证明种子数据完整。

## 6. Redis 配置与启用方式

### 6.1 单实例配置

在实际加载的配置文件中合并：

```yaml
system:
    use-redis: true
    use-multipoint: false

redis:
    useCluster: false
    addr: "127.0.0.1:6379"
    username: ""
    password: ""
    db: 0
```

| 字段 | 含义 |
| --- | --- |
| `system.use-redis` | 主 Redis 客户端开关；启动时会连接并执行 `PING`，失败会导致后端启动失败。 |
| `system.use-multipoint` | 登录拦截开关；新登录会将同账号前一枚登录令牌加入黑名单，需要先启用 Redis。 |
| `useCluster` | `false` 为单实例，`true` 为 Redis Cluster；注意字段采用驼峰写法。 |
| `addr` | 单实例的 `主机:端口`，不填写 `redis://` 前缀。 |
| `username` | 默认账号通常留空；配置了 ACL 或云服务自定义账号时填写实际用户名。 |
| `password` | Redis 密码；只有服务确实未设置密码时才留空。 |
| `db` | 单实例逻辑数据库编号，整数，例如 `0`；与 MySQL 库名无关。 |

使用第 7 节 Docker 示例时，主机上运行的 Go 后端应改为：

```yaml
redis:
    useCluster: false
    addr: "127.0.0.1:16379"
    username: ""
    password: "gva_redis_local_123456"
    db: 0
```

只希望给当前 SQLite 环境增加 Redis 时，保留 `system.db-type: sqlite` 和原有 `sqlite:` 配置，只修改 Redis 相关字段即可。

启用 Redis 只会建立客户端，并启用由业务代码实际调用的能力，不会自动把全部查询、权限数据或 JWT 都迁移到 Redis。当前 JWT 黑名单会写数据库，并加载到进程内存；登录拦截使用 Redis 保存同账号的登录令牌。

### 6.2 登录拦截和 `redis-list`

需要登录拦截时，把两个开关同时设为 `true`。若没有业务使用命名 Redis 实例，可以把原有列表替换为空列表：

```yaml
system:
    use-redis: true
    use-multipoint: true

redis-list: []
```

保留第 6.1 节已配置好的主 `redis:`。本仓库启动调用关系是：

```text
use-redis = true
  -> 初始化主 redis
  -> 如果 use-multipoint = true，再初始化 redis-list 全部条目
```

所以 `use-redis: true`、`use-multipoint: false` 时，**不会初始化 `redis-list`**。这属于本仓库当前实现，不应套用其他版本的行为。

如果业务调用 `global.GetRedis("cache")`，需要初始化相应的命名实例。例如，在两个开关均开启时保留：

```yaml
redis-list:
    - name: "cache"
      useCluster: false
      addr: "127.0.0.1:16379"
      username: ""
      password: "gva_redis_local_123456"
      db: 1
```

每个 `name` 必须唯一，所有条目都要能成功连接。主 Redis 配对了密码，但列表还保留原模板的无密码配置，也会导致启动失败。命名实例和主实例可以使用同一 Redis 服务，但配置需要分别填写；登录拦截本身使用的是主实例。

如需在不开启登录拦截的情况下初始化命名实例，需要另行调整初始化调用逻辑；仅修改 YAML 无法改变当前调用条件。

### 6.3 Redis Cluster 与云服务

只有已经搭建好 Redis Cluster 时才使用下面的片段，地址需替换为实际节点：

```yaml
redis:
    useCluster: true
    clusterAddrs:
        - "redis-node-1:7000"
        - "redis-node-2:7000"
        - "redis-node-3:7000"
    username: ""
    password: "替换为实际集群密码"
```

集群模式使用 `clusterAddrs`，单实例的 `addr`、`db` 不参与客户端创建；Redis Cluster 使用 DB 0。后端还必须能访问节点通告的地址。

当前配置封装没有暴露 TLS、Sentinel 的 YAML 选项。云服务要求 TLS 或 Sentinel 时，需要扩展对应配置和客户端创建逻辑，不能通过编造 `tls:` 或 `sentinel:` 字段直接启用。

## 7. 可选：用 Docker 运行 MySQL 和 Redis

本节适合“Windows 本机运行 Go 后端，Docker 只运行依赖服务”的方式。使用 Docker Desktop 的 Linux 容器模式和 Docker Compose；以下步骤由需要该环境的人按需执行。

### 7.1 创建依赖服务配置

将下面完整 YAML 保存为项目内新文件 `deploy/docker-compose/docker-compose.deps.local.yaml`。这是文档建议的新文件，本文没有预先创建它。密码是本地开发示例，实际使用时需同步修改 Compose 和后端配置。

该 Compose 文件路径当前不在 `server/*.local.yaml` 忽略规则范围内，填写真实密码后应单独加入忽略规则，或改用妥善管理的本地环境文件。

```yaml
name: pet-gva-deps

services:
    mysql:
        image: mysql:8.0
        restart: unless-stopped
        environment:
            MYSQL_ROOT_PASSWORD: "gva_root_local_123456"
            MYSQL_DATABASE: "gva"
            MYSQL_USER: "gva"
            MYSQL_PASSWORD: "gva_mysql_local_123456"
        ports:
            - "127.0.0.1:13306:3306"
        volumes:
            - mysql-data:/var/lib/mysql
        healthcheck:
            test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "--silent"]
            interval: 5s
            timeout: 5s
            retries: 30
            start_period: 30s

    redis:
        image: redis:7.4-alpine
        restart: unless-stopped
        command: ["redis-server", "--appendonly", "yes", "--requirepass", "gva_redis_local_123456"]
        environment:
            REDISCLI_AUTH: "gva_redis_local_123456"
        ports:
            - "127.0.0.1:16379:6379"
        volumes:
            - redis-data:/data
        healthcheck:
            test: ["CMD", "redis-cli", "ping"]
            interval: 5s
            timeout: 3s
            retries: 20

volumes:
    mysql-data:
    redis-data:
```

MySQL 使用命名卷保存数据；Redis 开启 AOF 并把数据保存在命名卷中。重建容器不等于清空这些卷。

### 7.2 启动、检查和停止

在项目根目录运行，`$gvaComposeFile` 只在当前 PowerShell 会话有效：

```powershell
cd E:\go\pet
$gvaComposeFile = '.\deploy\docker-compose\docker-compose.deps.local.yaml'
docker compose -f $gvaComposeFile config --quiet
docker compose -f $gvaComposeFile up -d
docker compose -f $gvaComposeFile ps
```

首次启动 MySQL 需要等待初始化完成。查看日志：

```powershell
docker compose -f $gvaComposeFile logs --tail 100 mysql redis
```

验证 MySQL 应用账号，在交互提示中输入示例 MySQL 密码：

```powershell
docker compose -f $gvaComposeFile exec mysql mysql -u gva -p -e "SELECT 1;" gva
```

验证 Redis，容器中的 `REDISCLI_AUTH` 会供 `redis-cli` 使用，预期输出 `PONG`：

```powershell
docker compose -f $gvaComposeFile exec redis redis-cli ping
```

MySQL 的 `mysqladmin ping` 健康检查只说明服务存活，不能替代应用账号的认证验证。完成上述验证后，仍需执行第 4 节的 GVA 初始化，镜像创建一个空库不会自动创建 GVA 管理员和菜单。

暂时停止依赖服务并保留数据：

```powershell
docker compose -f $gvaComposeFile stop
```

MySQL 镜像的 `MYSQL_USER`、`MYSQL_PASSWORD`、`MYSQL_ROOT_PASSWORD` 等初始化变量主要在空数据目录首次启动时生效。已有数据卷时，修改 YAML 不会自动修改已保存的 MySQL 账号密码，应通过数据库账号管理操作更新并同步应用配置。

### 7.3 地址和端口怎么填

| Go 后端运行位置 | MySQL 配置 | Redis 配置 |
| --- | --- | --- |
| Windows 本机，数据库也原生安装在本机 | `path: 127.0.0.1`，`port: "3306"`，以实际端口为准 | `addr: 127.0.0.1:6379`，以实际端口为准 |
| Windows 本机，依赖使用上面的 Docker 示例 | `path: 127.0.0.1`，`port: "13306"` | `addr: 127.0.0.1:16379` |
| 后端容器与数据库处于同一个 Compose 网络 | `path: mysql`，`port: "3306"` | `addr: redis:6379` |
| 后端容器连接 Docker Desktop 所在的宿主机服务 | 可使用 `host.docker.internal` 和服务实际可达的端口 | 同理，并确认宿主机服务监听地址允许该连接 |

`127.0.0.1` 始终指当前进程所在的机器或容器。后端放进容器后，它不再表示 Windows 宿主机。`mysql`、`redis` 服务名通常只在共享的 Docker 网络中解析，Windows 本机 Go 进程应使用发布到主机的端口。

本节 Compose 只包含依赖服务。若后端也容器化，需要另外配置后端服务加入同一网络；仅把 `path` 改成 `mysql` 不会自动加入网络。

## 8. 启动与验证

完整的首次依赖安装、SQLite 初始化、日常启动，以及修改 Go / Vue 代码后的命令，见 [本地启动与修改后运行指南](../README.local.md)。本节使用 MySQL 配置；只在 MySQL 或 Redis 配置确实启用时启动相应依赖。

### 8.1 推荐启动顺序

1. 启动 MySQL；启用了 Redis 时也启动 Redis，先验证账号可用。
2. 在 `server/` 使用明确的 `-c` 参数启动后端，检查启动输出中的配置路径。
3. 在 `web/` 启动前端；已运行的前端无需重复启动。
4. 空库执行首次初始化；已初始化库直接登录。
5. 验证健康检查、用户信息、菜单以及需要 Redis 的实际流程。

现有本地二进制可直接启动。需要重新编译时先停止后端，再执行：

```powershell
cd E:\go\pet\server
$env:GOSUMDB = 'sum.golang.org'
go build -o gva.local.exe .
.\gva.local.exe -c config.mysql.local.yaml
```

本机 Go 1.21.9 会按项目声明下载 Go 1.24.2 工具链；这里仅在当前终端覆盖原有 `GOSUMDB=off`，避免工具链校验失败，不修改全局 Go 配置。

### 8.2 端口、后端和前端代理检查

以下命令按第 7 节 Docker 端口编写；原生安装请替换为实际端口：

```powershell
Test-NetConnection -ComputerName 127.0.0.1 -Port 13306
Test-NetConnection -ComputerName 127.0.0.1 -Port 16379
Invoke-RestMethod -Uri 'http://127.0.0.1:8888/health'
Invoke-RestMethod -Uri 'http://127.0.0.1:8080/api/health'
Invoke-RestMethod -Method Post -Uri 'http://127.0.0.1:8888/init/checkdb' -ContentType 'application/json' -Body '{}'
```

端口检查的 `TcpTestSucceeded: True` 只代表 TCP 可达，不校验数据库账号密码。健康接口预期返回 `ok`，同样不能证明数据库业务数据或 Redis 功能全部正常。

初始化检查正常时返回：

```json
{"code":0,"data":{"needInit":false},"msg":"数据库无需初始化"}
```

当前本地 `system.router-prefix` 为空，上述后端接口没有 `/api` 前缀；前端开发代理使用 `/api` 并转发到 `8888`。如修改了路由前缀或端口，需同步调整验证 URL 和前端代理配置。

### 8.3 Redis 的进一步验证

启动日志应出现 `redis connect ping response` 和 `PONG`。如果开启登录拦截，再用两个浏览器会话先后登录同一账号，确认前一会话的受保护请求会因旧令牌失效而要求重新登录。

原生安装了 `redis-cli` 时，也可以用交互方式检查密码认证，按实际端口替换：

```powershell
redis-cli -h 127.0.0.1 -p 6379 --askpass ping
```

自定义 ACL 账号还需通过客户端的 `--user` 参数指定实际用户名。未启用 Redis 时无需执行 Redis 检查。

## 9. SQLite 切换、数据保留与切回

将 `db-type` 改为 `mysql`，再完成首次初始化，会得到一套新的 MySQL 基础数据。**这不等于把 SQLite 中的原有用户和业务记录迁移到 MySQL。**

建议保持两份独立配置：

| 文件 | 用途 |
| --- | --- |
| `server/config.local.yaml` | 当前已初始化的 SQLite 环境 |
| `server/gva.db` | 当前 SQLite 数据文件 |
| `server/config.mysql.local.yaml` | 后续按本文新建的 MySQL 环境配置 |

如果需要备份 SQLite 文件，应先正常停止后端再复制，确保没有正在写入的事务；如出现 WAL 文件，应使用一致性备份方式，不能在运行中仅随意复制主 `.db` 文件。

需要切回 SQLite 时，停止使用 MySQL 配置启动的后端，再运行：

```powershell
cd E:\go\pet\server
.\gva.local.exe -c config.local.yaml
```

切换后重新登录，确认访问的是原来的用户和数据。不要同时启动两份后端占用 `8888`，也无需删除 MySQL 数据库或 SQLite 文件。

需要真正迁移业务数据时，应单独设计表结构、主键和关联关系、时间字段、布尔值、JSON、唯一约束与字符集转换，完成导出、转换、导入和校验。不要把 SQLite 导出的 SQL 原样当作 MySQL 脚本执行。

## 10. 常见报错排查

| 现象或报错 | 常见原因 | 处理方式 |
| --- | --- | --- |
| 修改配置后仍然使用 SQLite / MySQL | 启动时加载了另一个文件，或旧后端还在运行 | 查看启动输出中的配置路径，核对 `-c`，停止旧进程后重启。 |
| `fatal error config file` / 找不到 YAML | 工作目录错误，或显式指定的文件不存在 | 进入 `server/`，核对文件名；必要时为 `-c` 提供绝对路径。 |
| YAML 解析失败、重复映射键 | 缩进错误、Tab、未正确引用密码、重复顶层配置 | 合并原有配置段，使用空格缩进，密码加引号。 |
| MySQL `connection refused` / `connectex` | 服务未启动、端口错误、容器未发布端口 | 先检查服务状态和 TCP；本机连接 Docker 示例应使用 `13306`。 |
| MySQL `Access denied for user` | 密码错误、MySQL 账号 Host 不匹配、授权不足 | 使用同一账号实际登录，核对来源主机和库权限；GVA 管理员密码不能用来连接 MySQL。 |
| MySQL `Unknown database` | 未创建库，但正常启动配置已填了库名 | 新开发库按第 4 节留空配置库名并完成初始化；已有库核对库名。 |
| 后端启动成功，但 `admin` 登录失败或菜单为空 | 只完成建表，未写入初始化数据；连错库；管理员密码记错 | 核对目标库、初始化记录和页面填写的管理员密码；不要在已有业务库反复初始化。 |
| 初始化提示“已存在数据库配置” | 主数据库句柄已存在，或上次部分初始化留下进程状态 | 确认是否本来就已初始化；失败恢复时查看日志，停止后端并检查配置和目标数据后再启动。 |
| Redis `NOAUTH Authentication required` | 服务要求密码，客户端没配 | 在主 `redis.password` 以及实际启用的 `redis-list` 条目中填写对应密码并重启。 |
| Redis `WRONGPASS` | 密码错误、ACL 用户错误或被禁用 | 核对账号和密码；默认账号与自定义 ACL 账号分别处理。 |
| `redis connect ping failed` 后端退出 | Redis 地址、认证或网络错误 | 先通过 Redis 客户端认证并得到 `PONG`，再启动后端；同时检查命名实例列表。 |
| 开启登录拦截后登录失败 | `use-multipoint` 为 true，但 Redis 未正确启用 | 同时启用并正确配置 Redis；无此需求时将两个开关都关闭。 |
| `global.GetRedis("cache")` 报错 | 名称错误、条目未配置，或 `redis-list` 没被初始化 | 核对名称和当前版本的双开关初始化条件，见第 6.2 节。 |
| Docker `port is already allocated` / Windows 端口占用 | 主机端口已有其他服务 | 调整 Compose 左侧主机端口，并同步修改后端连接端口。 |
| 修改 Compose MySQL 密码后还是登录失败 | 旧数据卷保留了原账号密码 | 按原密码连接后用 MySQL 账号管理操作改密；不要把删卷当作日常改密方法。 |
| 前端 `/api` 返回 502、无法连接 | 后端没启动、端口不匹配、代理地址错误 | 先访问后端 `/health`，再检查前端代理和 `VITE_SERVER_PORT`。 |
| Go 工具链提示 `checksum database disabled` | 本机全局 `GOSUMDB=off` | 按第 8.1 节在当前会话临时设置 `sum.golang.org` 后重新编译。 |

排查顺序建议是：**实际加载的配置文件 → 服务状态和地址 → 账号认证 → GVA 初始化数据 → 页面与业务功能**。

## 11. 配置与数据的维护

- 保留不含真实密码的模板，真实账号只写入受管理的本地配置或部署配置；初始化接口会回写配置，因此相关文件需要对后端进程可写。
- 数据库账号、Redis 密码和 GVA 管理员密码分别维护。改密后同步应用配置并重新启动验证。
- 需要备份的是数据本身：MySQL 数据、SQLite 文件、Redis 中确实需要保留的数据，以及上传文件。仅复制 YAML 不能恢复业务数据。
- 调整连接池时结合后端实例数量和数据库总连接数；不要直接把开发参数当作所有环境的容量标准。
- 依赖服务没有准备好时，先保持 `use-redis: false` 和 `use-multipoint: false`，完成主数据库初始化后再逐步启用 Redis。

## 12. 源码和参考文档

本文的项目行为依据以下源码核对：

| 主题 | 源码位置 |
| --- | --- |
| 配置路径选择与监听 | [core/viper.go](../server/core/viper.go)、[core/internal/constant.go](../server/core/internal/constant.go) |
| MySQL 字段、日志级别与 DSN | [config/db_list.go](../server/config/db_list.go)、[config/gorm_mysql.go](../server/config/gorm_mysql.go) |
| 主数据库选择与自动迁移 | [initialize/gorm.go](../server/initialize/gorm.go)、[initialize/gorm_mysql.go](../server/initialize/gorm_mysql.go) |
| 初始化接口及配置回写 | [api/v1/system/sys_initdb.go](../server/api/v1/system/sys_initdb.go)、[service/system/sys_initdb_mysql.go](../server/service/system/sys_initdb_mysql.go) |
| 初始化请求字段 | [model/system/request/sys_init.go](../server/model/system/request/sys_init.go) |
| Redis 字段及连接创建 | [config/redis.go](../server/config/redis.go)、[initialize/redis.go](../server/initialize/redis.go) |
| Redis 开关与列表初始化条件 | [core/server.go](../server/core/server.go) |
| 登录拦截与 JWT 黑名单 | [api/v1/system/sys_user.go](../server/api/v1/system/sys_user.go)、[service/system/jwt_black_list.go](../server/service/system/jwt_black_list.go) |
| 初始化页面和 Hash 路由 | [view/init/index.vue](../web/src/view/init/index.vue)、[router/index.js](../web/src/router/index.js) |

外部参考：[gin-vue-admin 官方文档](https://www.gin-vue-admin.com/guide/start-quickly/env.html)、[Docker Compose 网络说明](https://docs.docker.com/compose/how-tos/networking/)、[Docker Compose 配置检查命令](https://docs.docker.com/reference/cli/docker/compose/config/)。框架在线文档可能对应其他版本，具体开关与字段行为以本仓库代码为准。

本文 MySQL、Redis 示例按源码核对并进行配置语法检查；没有在当前 SQLite 环境上执行 MySQL 初始化或 Redis 联调。
