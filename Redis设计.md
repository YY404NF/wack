# wack 网安查课 / Redis设计

本文档基本由AI生成，请AI在编写项目时主动修正本文档，不要移除这句话。

本文档是架构升级阶段的 Redis 纸面计划，用于统一后续改造中 Redis 的职责边界、Key 分组、生命周期和失效策略。

文档目标：

- 对齐 [README.md](/home/kevin-zhang/projects/wack/wack/README.md) 中的用户组、统计目标与技术栈
- 对齐 [前端页面.md](/home/kevin-zhang/projects/wack/wack/前端页面.md) 中需要快速响应的页面场景
- 对齐 [后端接口.md](/home/kevin-zhang/projects/wack/wack/后端接口.md) 中已经确定的接口边界与业务强约束

本文档不再使用“先这样试试”式表达。本文档全部内容均视为本轮改造的 Redis 基线方案。

## 1. 文档定位

本文档解决以下问题：

- Redis 在本项目里承担什么职责
- 哪些数据必须进 Redis
- 哪些数据不能进 Redis
- 每类 Key 的命名、值结构、TTL、失效时机如何统一
- 后端改造时 Redis 应如何与数据库和接口计划配合

本文档不展开以下内容：

- Redis 集群、高可用、哨兵部署方案
- 云托管 Redis 运维策略
- 具体 Go 代码封装细节
- 监控告警系统细节

## 2. 本期设计范围

本期 Redis 只覆盖以下用途：

- 登录会话管理
- 首页统计缓存
- 高频元数据缓存
- 短时运行态控制
- 限流与防刷

本期 Redis 不承担以下职责：

- 核心业务真值存储
- 唯一持久化数据来源
- 复杂消息队列
- 长期审计日志存储

## 3. 设计总则

## 3.1 真值边界

本项目中业务真值固定如下：

- SQLite / 关系库：用户、学生、班级、课程、课程组、课次、考勤记录、考勤日志、学期
- Redis：缓存、会话、短期状态、限流计数

Redis 中的数据全部视为可丢失、可重建、可回源的数据。

## 3.2 使用原则

- 不在 Redis 中保存核心业务真值
- 不让业务正确性依赖 Redis 持久存在
- Redis 命中失败后，后端必须可以回源数据库并重建缓存
- Redis Key 命名统一、可枚举、可按前缀批量清理
- 同一类 Key 的 TTL 与失效策略必须统一

## 3.3 Key 命名总规范

统一采用冒号分段命名，格式固定为：

```text
{domain}:{scope}:{id}:{field}
```

命名要求：

- 全部小写
- 不使用中文
- 不混用下划线与中划线
- 同类 Key 使用稳定前缀，便于批量删除

## 4. 本期必须落地的 Redis 模块

本期固定落地以下五类模块：

- 认证会话
- 首页统计缓存
- 元数据缓存
- 查课流程短时控制
- 限流与防刷

## 5. 认证会话

## 5.1 目标

服务以下场景：

- 登录后保存服务端可控会话
- 主动退出登录
- 管理员冻结账号后强制失效

## 5.2 Key 设计

```text
auth:session:{token_id}
auth:user_sessions:{user_id}
```

说明：

- `auth:session:{token_id}` 保存单个登录会话详情
- `auth:user_sessions:{user_id}` 保存某用户当前所有有效 `token_id`

其中：

- `token_id` 使用 JWT 的 `jti`
- Redis 不直接用整段 JWT 作为 Key

## 5.3 值结构

`auth:session:{token_id}` 存储字段：

- `user_id`
- `account`
- `role`
- `status`
- `device_type`
- `issued_at`
- `expires_at`

`auth:user_sessions:{user_id}` 使用 Set，成员为当前用户的全部 `token_id`。

## 5.4 TTL

- 管理端会话：`7d`
- 移动端会话：`15d`

本期固定统一，不再拆更细粒度配置。

## 5.5 失效时机

以下场景必须删除会话：

- 用户主动退出登录
- 用户修改密码
- 管理员重置该用户密码
- 管理员冻结该用户
- 会话自然过期

## 5.6 后端约束

- JWT 校验通过后，仍需检查对应 `token_id` 是否存在于 Redis
- Redis 中不存在的会话视为已失效
- 冻结用户时，必须批量删除该用户全部会话

## 6. 首页统计缓存

## 6.1 目标

服务以下页面：

- 管理员首页 `主页`

缓存内容：

- 学生出勤率排行榜
- 班级出勤率排行榜
- 课程出勤率排行榜
- 最近完成查课课次
- 最近缺勤、迟到、请假学生

## 6.2 Key 设计

```text
dashboard:term:{term_id}:overview
```

说明：

- 管理员首页整体按学期整包缓存
- 本期不拆成多个散 Key

## 6.3 值结构

值为首页概览接口的完整响应数据体。

这样做的好处：

- 后端可直接命中返回
- 前端接口形态与缓存结构保持一致
- 失效逻辑简单

## 6.4 TTL

- `120s`

## 6.5 失效时机

以下场景必须删除对应学期首页缓存：

- 新增考勤结果
- 修改考勤结果
- 删除考勤结果
- 新增课次
- 修改课次
- 删除课次
- 课程组学生范围变化

## 6.6 后端约束

- 只做删除式失效，不做增量修补
- 删除后由下一次访问触发回源重建

## 7. 高频元数据缓存

## 7.1 目标

服务以下接口：

- `/meta/context`
- `/meta/terms`
- `/meta/terms/{term_id}/weeks`
- `/meta/sections`

这些数据变更频率低，但会被多个页面反复读取，适合做短期缓存。

## 7.2 Key 设计

```text
meta:terms:all
meta:term_weeks:{term_id}
meta:sections:current_date:{date}
meta:context:user:{user_id}
```

说明：

- `meta:terms:all` 保存学期下拉源数据
- `meta:term_weeks:{term_id}` 保存某学期周次信息
- `meta:sections:current_date:{date}` 保存某日对应的作息定义
- `meta:context:user:{user_id}` 保存用户首页初始化上下文

## 7.3 TTL

- `meta:terms:all`：`10m`
- `meta:term_weeks:{term_id}`：`30m`
- `meta:sections:current_date:{date}`：`24h`
- `meta:context:user:{user_id}`：`5m`

## 7.4 失效时机

- 学期配置变更后，删除 `meta:terms:*` 与 `meta:term_weeks:*`
- 日期跨天后，`meta:sections:current_date:{date}` 自然过期
- 用户角色、状态、`managed_class_id` 变更后，删除 `meta:context:user:{user_id}`

## 7.5 后端约束

- `meta/context` 含用户权限范围，不能做全局公共缓存
- 学委的 `meta/context` 必须与其 `managed_class_id` 同步失效

## 8. 查课流程短时控制

## 8.1 目标

服务以下场景：

- 防止移动端查课提交重复写入
- 提升查课流程的短时一致性
- 对高频短期状态做轻量控制

说明：

- 查课真值仍然落数据库
- Redis 不保存最终考勤结果

## 8.2 本期用途

本期 Redis 在查课流程中只做以下两件事：

- 幂等提交令牌
- 短时互斥锁

## 8.3 幂等提交令牌

Key：

```text
attendance:submit_token:{course_group_lesson_id}:{operator_user_id}:{token}
```

用途：

- 防止移动端因重复点击“提交查课结果”导致重复提交

TTL：

- `60s`

规则：

- 首次提交写入成功后，后续相同提交令牌直接拒绝或返回已处理

## 8.4 短时互斥锁

Key：

```text
lock:attendance_session:{course_group_lesson_id}
```

用途：

- 控制同一课次提交动作的高并发写入

TTL：

- `5s`

规则：

- 仅作为短时互斥控制，避免并发写放大
- 真正的正确性仍靠数据库唯一约束和事务保证

## 8.5 后端约束

- Redis 锁获取失败时可以短暂重试
- 即便锁机制失效，也不能影响数据库层唯一约束
- 不将浏览器本地草稿整体存入 Redis

## 9. 限流与防刷

## 9.1 目标

服务以下场景：

- 登录防爆破
- 验证码防刷
- 高频接口限流

## 9.2 Key 设计

```text
rate_limit:login:account:{account}
rate_limit:login:ip:{ip}
rate_limit:captcha:ip:{ip}
rate_limit:api:user:{user_id}:{api_name}
```

## 9.3 TTL

- 登录账号失败计数：`15m`
- 登录 IP 失败计数：`15m`
- 验证码请求限制：`10m`
- 高频接口限制：`60s`

## 9.4 规则

- 登录失败计数达到阈值后，短时间内禁止继续尝试
- 同一 IP 验证码请求达到阈值后，短时间内禁止继续获取
- 高频接口根据用户维度做窗口计数

## 9.5 后端约束

- 限流命中必须返回统一错误码和可读提示
- 限流属于保护机制，不影响数据库真值

## 10. 明确不进入 Redis 的数据

以下数据本期明确不进入 Redis：

- 用户密码明文
- 用户密码哈希真值
- 用户、学生、班级、课程、课程组、课次、考勤记录、考勤日志真值
- 管理员操作日志真值
- 只保存在 Redis 且无法回源恢复的重要业务数据
- 浏览器本地查课草稿

## 11. Key 清单总表

本期 Redis Key 收敛如下：

```text
auth:session:{token_id}
auth:user_sessions:{user_id}
dashboard:term:{term_id}:overview
meta:terms:all
meta:term_weeks:{term_id}
meta:sections:current_date:{date}
meta:context:user:{user_id}
attendance:submit_token:{course_group_lesson_id}:{operator_user_id}:{token}
lock:attendance_session:{course_group_lesson_id}
rate_limit:login:account:{account}
rate_limit:login:ip:{ip}
rate_limit:captcha:ip:{ip}
rate_limit:api:user:{user_id}:{api_name}
```

## 12. 失效策略总表

认证会话：

- 退出登录删除
- 改密码删除
- 重置密码删除
- 冻结账号删除

首页统计缓存：

- 考勤变更删除
- 课次变更删除
- 课程组学生范围变更删除

元数据缓存：

- 学期变更删除
- 用户角色或 `managed_class_id` 变更删除
- 作息缓存按日期自然过期

短时控制：

- 幂等令牌自然过期
- 互斥锁自然过期

限流：

- 全部按 TTL 自然过期

## 13. 与后端接口计划的对应关系

与 [后端接口.md](/home/kevin-zhang/projects/wack/wack/后端接口.md) 的对应关系如下：

- `auth/*` 接口依赖认证会话 Redis Key
- `meta/*` 接口依赖元数据缓存
- `admin/dashboard/overview` 依赖首页统计缓存
- `mobile/attendance/sessions/{course_group_lesson_id}/submit` 依赖幂等提交令牌和短时锁
- 登录、验证码、高频接口依赖限流计数

## 14. 落地顺序

按后端改造顺序，Redis 落地顺序固定为：

1. 认证会话
2. 首页统计缓存
3. 元数据缓存
4. 查课提交流程幂等控制
5. 限流与防刷

## 15. 本文档相对旧稿的收敛点

本稿相对旧版 Redis 设计文档，做了以下收敛：

- 从“粗略建议稿”改为“本期纸面计划”
- 明确 Redis 只承担缓存、会话、短期控制，不承担业务真值
- 将首页统计缓存收敛为单 Key 整包缓存
- 明确了与学委权限和移动端查课链路有关的缓存边界
- 增加了元数据缓存、幂等控制、短时锁和限流分组
- 将 Key 命名、TTL 和失效时机统一成可执行规则
