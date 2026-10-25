# 本地运行 gin-vue-admin

## 基本信息

- 提出日期：2026-10-22
- 当前状态：`done`
- 需求类型：本地环境搭建与启动验证

## 用户原始意图摘要

先把 gin-vue-admin 在本机运行成功，包括数据库初始化、前后端启动和登录验证。

## 影响范围与约束

- 后端：`server/`，使用独立的 `config.local.yaml`，保留原始默认配置。
- 前端：`web/`，使用本机 Node.js 与 Vite 开发服务。
- 数据库：用户已明确使用 SQLite，数据库文件保存在本地，不依赖 MySQL 或 Docker。
- 保留现有业务代码与其他项目的容器、数据库和配置。

## 当前进展

- 已安装 Go 1.24.2 工具链与前端依赖，生成 `server/gva.local.exe` 和 `web/package-lock.json`。
- 已按用户纠正改用 SQLite，并清理本次误建的 MySQL 开发环境。
- 已通过初始化接口创建 `server/gva.db`，初始管理员为 `admin` / `123456`。
- 后端使用 `server/config.local.yaml`，端口 `8888`；Vite 前端端口 `8080`。
- 已验证代理健康检查与数据库就绪状态，并观察到登录、用户信息、菜单和媒体库接口成功及后台页面正常显示。
- 本机全局 `GOSUMDB=off` 阻止工具链校验，编译时在进程内临时设为 `sum.golang.org`，未修改全局环境。
- 启动和重新编译方式已记录在 `README.local.md`。

## 后续待办

- 无；后续开发继续使用 SQLite。
