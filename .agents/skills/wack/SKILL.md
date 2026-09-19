---
name: wack-project
description: Work on the WACK attendance system when changes span its Vue frontend, Go backend, shared business rules, API contracts, or deployment configuration.
---

# WACK 项目上下文

WACK 是网络空间安全学院查课系统。当前代码拆分为两个 submodule：`frontend/` 是 Vue 3 + TypeScript + Vite 单页应用，`backend/` 是 Go 1.26 + Gin + GORM 单体服务。SQLite 保存业务真值，Redis 负责缓存、会话、幂等控制、短时锁和限流。

开始任务前先阅读根目录 `README.md`、`设计文档.md`，再按任务阅读 `前端页面.md`、`Redis设计.md` 或对应子项目的 README。跨端功能必须一起核对 `frontend/src/api/` 与 `backend/internal/httpserver/`、`backend/internal/service/`。

保持这些业务规则：学生只看到当前可查窗口内的课次；同一课次的首次成功考勤记录优先；管理员修改考勤必须留审计记录；已有考勤历史的对象只能逻辑删除或冻结；SQLite 是业务真值，不能把 Redis 当持久化数据库。

验证修改时，前端运行 `npm ci && npm run build`，后端运行 `go test ./...`。不要读取、提交或输出 `.env`、JWT 密钥、Redis 密码、数据库文件和部署私钥。修改 submodule 后先在子仓库提交，再在根仓库提交新的 submodule 指针。
