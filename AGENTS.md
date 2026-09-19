# WACK 项目协作指南

WACK 是网络空间安全学院查课系统。仓库根目录是协作入口，业务代码分别位于两个 Git submodule：

- `frontend/`：Vue 3 + TypeScript + Vite + Pinia + Element Plus 的单页前端，包含管理员 PC 端和学生移动端。
- `backend/`：Go 1.26 + Gin + GORM 的单体后端；SQLite 保存业务真值，Redis 用于缓存、会话、幂等控制、短时锁和限流。

## 先读哪些文件

- 全局业务边界、角色、时间规则和统计目标：`README.md`
- 页面和交互设计：`前端页面.md`
- 系统设计和接口/数据约束：`设计文档.md`
- Redis 使用约束：`Redis设计.md`
- 前端实现：`frontend/README.md` 和 `frontend/src/`
- 后端实现：`backend/README.md`、`backend/internal/`、`backend/ops/`

需要修改跨端行为时，同时检查前端 `frontend/src/api/` 的类型与请求封装，以及后端 `backend/internal/httpserver/`、`backend/internal/service/` 对应实现。

## 开发和验证

首次获取完整工作区使用：

```bash
git clone --recurse-submodules <wack-repository-url>
git submodule update --init --recursive
```

前端：

```bash
cd frontend
npm ci
npm run build
```

后端：

```bash
cd backend
go test ./...
```

不要提交 `.env`、数据库文件、部署私钥、JWT 密钥或 Redis 密码。配置修改应同步更新对应的 `.env.example` 或 `backend/ops/env/wack-backend.env.example`。

## Submodule 规则

前端和后端各自拥有提交历史、依赖和部署工作流。修改子项目后，先在子模块仓库提交，再回到根仓库提交新的 submodule 指针：

```bash
git -C frontend add -A && git -C frontend commit
git -C backend add -A && git -C backend commit
git add frontend backend
git commit
```

根仓库只记录子模块提交指针，不把两个项目的文件复制进来。更新已有工作区前先确认子模块没有未提交修改。

## 设计约束

- 数据真值在 SQLite；Redis 不能替代业务持久化。
- 考勤首次成功写入后，普通提交不能覆盖；管理员修改必须保留审计记录。
- 删除已有考勤历史的业务对象时使用逻辑删除或冻结。
- 认证、CORS、会话失效和权限检查应沿用现有服务层与 HTTP 中间件，不在页面中复制业务规则。
