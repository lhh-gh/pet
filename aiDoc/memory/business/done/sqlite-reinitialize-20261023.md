# 重新初始化项目 SQLite 数据库

- 提出日期：2026-10-23
- 当前状态：`done`
- 用户要求：重新初始化当前项目的 SQLite 数据库。
- 当前配置：`server/config.local.yaml`，SQLite 文件为 `server/gva.db`。
- 执行范围：备份旧库与原配置，通过项目 `/init/initdb` 创建全新库，验证后替换当前库；恢复默认账号、菜单和权限。
- 约束：保留 SQLite，不切换 MySQL；不修改业务源码；保留可恢复备份；完成后保持此前前后端停止状态。
- 完成内容：先在临时 SQLite 库调用项目 `/init/initdb`，验证完成后替换 `server/gva.db`；原库保留一致性备份和原始文件两份。
- 备份目录：`server/backups/sqlite-reset-20261023-070126/`，含原配置、原库、校验信息和恢复说明；本地 `.gitignore` 排除备份内容。
- 新库验证：SQLite integrity_check=ok，33 张业务 / 系统表；2 个用户、3 个角色、50 个菜单、190 条 API、291 条 Casbin 规则。
- 账号验证：管理员 admin 重新生成 UUID 与密码哈希，已通过 BCrypt 校验默认密码 123456；未绕过或变更验证码设置。
- 配置变化：保留原 SQLite、端口及依赖开关，只更新 JWT signing-key，使旧登录状态失效。
- 运行验证：使用正式配置启动，/health 返回 HTTP 200，/init/checkdb 返回 needInit=false；检查后已停止临时后端，前端保持停止。
- 恢复方式：停止服务后，按备份目录 README.md 配套恢复原数据库和原配置。
