# wack 网安查课 / Redis设计

本文档基本由AI生成，本人不太了解Redis设计，请AI在编写项目时主动修正本文档。

1. Redis 只作为缓存和运行时状态存储，不作为核心业务数据唯一来源。
2. 核心业务真值仍以 MySQL 为准，尤其是用户、学生、课程、课次、考勤记录。
3. 当前项目中 Redis 优先用于首页统计缓存，其次用于登录会话。

### 1. 首页统计缓存

1. 本部分用于缓存某学期首页展示的全量考勤统计结果，避免每次打开首页都实时聚合 MySQL。
2. 初版不做复杂增量更新，采用“按学期整包缓存 + 考勤变更后删除”的简单策略。

建议 key：

```text
term:{termId}:attendance:student:summary
term:{termId}:attendance:class:summary
term:{termId}:attendance:course:summary
```

说明：

1. `student` 表示某学期全部学生出勤率汇总
2. `class` 表示某学期全部班级出勤率汇总
3. `course` 表示某学期全部课程出勤率汇总

建议 TTL：

1. 120 秒

失效时机：

1. 新增考勤记录
2. 修改考勤记录
3. 删除考勤记录

### 2. 登录会话

1. 本部分用于保存登录后的会话状态，便于服务端主动失效登录态。
2. 当用户主动退出、管理员冻结账号、管理员手动踢下线时，应删除对应会话。

建议 key：

```text
auth:session:{token}
user:{userId}:session_set
```

说明：

1. `auth:session:{token}` 保存单个 token 对应的会话信息
2. `user:{userId}:session_set` 用于保存某用户当前所有登录 token，便于统一下线

建议 TTL：

1. Web 管理端 7 天
2. 移动端 7 天到 30 天

### 3. 验证码与短期令牌

1. 如果后续需要验证码、重置密码、短信/邮箱校验，可使用 Redis 保存短期数据。

建议 key：

```text
auth:captcha:{loginId}
auth:reset_password:{token}
```

建议 TTL：

1. 验证码 5 分钟
2. 使用后立即删除

### 4. 限流与防刷

1. 如果后续要做登录防爆破、接口限流、重复提交防护，可使用 Redis 做计数。

建议 key：

```text
rate_limit:login:{loginId}
rate_limit:ip:{ip}
```

建议 TTL：

1. 登录失败计数 10 分钟到 30 分钟
2. 防重复提交 token 10 秒到 60 秒

### 5. 不放入 Redis 的数据

1. 用户密码明文
2. 用户密码哈希真值
3. 学生、班级、课程、考勤记录等核心持久化业务数据真值
4. 只保存在 Redis、断电后不可恢复的重要业务数据

### 6. 当前项目建议

1. MySQL 仍然作为唯一业务真值。
2. Redis 首先用于首页出勤率缓存。
3. Redis 其次用于登录会话管理。
4. 初版不建议把 Redis 设计得过重。
