# 网络空间安全学院查课系统 `wack`

## 用户组

- 管理员组
- 查课学生组

## 界面

### 管理员端

适配 PC 端，横向大屏。

#### 1. 全院课程表

- 查看/创建/修改/删除课程
- 修改上课学生

#### 2. 查课结果美化数据看板

- 用于直观查看各种统计数据
- 界面尽量丰富

#### 3. 查课结果表格看板

- 列表界面
- 可查看所有应到、未到明细
- 支持修改查课结果

#### 4. 查课学生空余时间看板

- 日历界面
- 直观查看所有查课学生的空闲时间

#### 5. 设置

- 创建查课学生账号
- 冻结/解冻查课学生账号
- 设置账号角色（可将其他账号设为管理员）
- 冻结/解冻其他管理员账号
- 创建/管理班级
- 编辑上课学生列表
- 修改密码

### 查课学生端

适配移动端，竖向小屏。

#### 1. 可查列表

- 列表界面
- 显示当天全学院课程
- 可直接进入查课流程

#### 2. 查课流程界面

- 针对当天某门课逐个学生进行查课
- 状态包括：签到、迟到、缺勤、请假

#### 3. 设置

- 空余时间设置
- 修改密码

## 数据结构

### 1. 用户表

```sql
CREATE TABLE user (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,               -- 主键，系统内部唯一用户 ID
    student_id VARCHAR(32) NOT NULL UNIQUE,            -- 学号/账号，仍作为登录账号，如 "202231104021059"
    password_hash VARCHAR(255) NOT NULL,               -- 登录密码，建议存加密后的密码摘要，不直接存明文
    real_name VARCHAR(50) NOT NULL,                    -- 姓名
    role TINYINT NOT NULL,                             -- 角色，1=管理员，2=查课学生
    status TINYINT NOT NULL DEFAULT 1,                 -- 账号状态，1=正常，2=冻结
    last_login_at DATETIME DEFAULT NULL,               -- 上次成功登录时间
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    KEY idx_role_status (role, status)                 -- 常用索引，便于按角色、状态筛选用户
);
```

备注：

- `id` 是系统内部稳定主键，所有业务关系表都引用 `id`，不直接引用学号。
- `student_id` 仍然保留为唯一登录账号，因此修改学号或姓名时，不会影响查课记录等历史数据。
- `role` 和 `status` 直接落表，不在代码里硬编码。
- `last_login_at` 仅在成功登录时更新，用于系统用户管理展示最近登录时间。
- 业务上不提供账号删除功能，默认不设计账号删除接口。

### 2. 班级表

```sql
CREATE TABLE class (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    class_code VARCHAR(50) NOT NULL UNIQUE,            -- 班级编号，如 "xa2301"
    class_name VARCHAR(100) NOT NULL,                  -- 班级名称，如 "信安2301班"
    grade SMALLINT NOT NULL,                           -- 年级，如 2023
    major_name VARCHAR(100) NOT NULL,                  -- 专业名称，如 "信息安全"
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    KEY idx_grade_major (grade, major_name)            -- 常用索引，便于按年级、专业筛选班级
);
```

备注：

- 你已经有“创建/管理班级”的需求，所以班级表单独存在。
- 删除策略采用物理删除。

### 3. 用户班级关系表

```sql
CREATE TABLE user_class (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    user_id BIGINT NOT NULL,                           -- 用户内部 ID
    class_id BIGINT NOT NULL,                          -- 班级 ID
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_user_class (user_id, class_id),      -- 唯一约束，同一学生和同一班级关系只能有一条
    KEY idx_class_id (class_id),                       -- 常用索引，便于查询某个班级下的学生
    CONSTRAINT fk_user_class_user FOREIGN KEY (user_id) REFERENCES user(id),
    CONSTRAINT fk_user_class_class FOREIGN KEY (class_id) REFERENCES class(id)
);
```

备注：

- 用户和班级的关系单独建表，不直接塞进 `user` 表。
- 删除策略采用物理删除。

### 4. 学生空闲时间表

```sql
CREATE TABLE student_free_time (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    term VARCHAR(20) NOT NULL,                         -- 学期，如 "2025-2026-2"
    user_id BIGINT NOT NULL,                           -- 用户内部 ID
    weekday TINYINT NOT NULL,                          -- 星期几，1=周一 ... 7=周日
    section TINYINT NOT NULL,                          -- 时间片编号，1=上午1-2节，2=上午3-4节，3=下午5-6节，4=下午7-8节，5=晚上9-10节
    free_weeks VARCHAR(100) NOT NULL,                  -- 空闲周，如 "1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16"
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_term_student_time (term, user_id, weekday, section), -- 唯一约束，同一学期同一学生同一星期同一时间片只能有一条记录
    KEY idx_student_term (user_id, term),              -- 常用索引，便于查询某学生某学期空闲时间
    CONSTRAINT fk_student_free_time_user FOREIGN KEY (user_id) REFERENCES user(id)
);
```

备注：

- `id` 是主键，`uk_term_student_time` 是业务唯一约束，两者职责不同。
- 这张表表示某个学生在某学期固定时间片的空闲周分布。
- 删除策略采用物理删除。

### 5. 课程表

```sql
CREATE TABLE course (
    id BIGINT PRIMARY KEY,                             -- 主键，课程 ID / 开课编号，如 "202520262004210"
    term VARCHAR(20) NOT NULL,                         -- 学期，如 "2025-2026-2"
    course_name VARCHAR(100) NOT NULL,                 -- 课程名称
    teacher_name VARCHAR(50) NOT NULL,                 -- 授课老师姓名
    attendance_student_count INT NOT NULL DEFAULT 0,   -- 应考勤学生人数
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    KEY idx_term_teacher (term, teacher_name),         -- 常用索引，便于按学期和教师筛课
    KEY idx_term_course_name (term, course_name)       -- 常用索引，便于按学期和课程名搜索
);
```

备注：

- 课程主表只保留课程本身相对稳定的信息。
- “星期几、时间片、教室、周次”不直接放这里，放到“第几次课”的表里。
- 删除策略采用物理删除。

### 6. 课程上课学生关联表

```sql
CREATE TABLE course_student (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    course_id BIGINT NOT NULL,                         -- 课程 ID / 开课编号
    user_id BIGINT NOT NULL,                           -- 用户内部 ID
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_course_student (course_id, user_id), -- 唯一约束，同一门课下同一学生只能出现一次
    KEY idx_student_id (user_id),                      -- 常用索引，便于查询某学生上的所有课程
    CONSTRAINT fk_course_student_course FOREIGN KEY (course_id) REFERENCES course(id),
    CONSTRAINT fk_course_student_user FOREIGN KEY (user_id) REFERENCES user(id)
);
```

备注：

- 这张表是“谁实际上这门课要上”的基准表。
- 这样可以解决“不是班上所有人都去上”的问题。
- 你已经说明这部分一般不改，所以按稳定关联处理。
- 删除策略采用物理删除。

### 7. 课程上课班级关联表

```sql
CREATE TABLE course_class (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    course_id BIGINT NOT NULL,                         -- 课程 ID / 开课编号
    class_id BIGINT NOT NULL,                          -- 班级 ID
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_course_class (course_id, class_id),  -- 唯一约束，同一门课下同一班级只能出现一次
    KEY idx_class_id (class_id),                       -- 常用索引，便于查询某班级关联的课程
    CONSTRAINT fk_course_class_course FOREIGN KEY (course_id) REFERENCES course(id),
    CONSTRAINT fk_course_class_class FOREIGN KEY (class_id) REFERENCES class(id)
);
```

备注：

- 这张表主要用于管理端按班级展示和筛选课程。
- 真正计算应到学生时，仍然以 `course_student` 为准。
- 删除策略采用物理删除。

### 8. 课程上课次数表

```sql
CREATE TABLE course_session (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    course_id BIGINT NOT NULL,                         -- 课程 ID / 开课编号
    session_no INT NOT NULL,                           -- 第几次课，从 1 开始递增
    week_no INT NOT NULL,                              -- 第几周
    weekday TINYINT NOT NULL,                          -- 星期几，1=周一 ... 7=周日
    section TINYINT NOT NULL,                          -- 时间片编号，1=上午1-2节，2=上午3-4节，3=下午5-6节，4=下午7-8节，5=晚上9-10节
    building_name VARCHAR(50) NOT NULL,                -- 教学楼，如 "教4"、"双创大楼"
    room_name VARCHAR(50) NOT NULL,                    -- 教室，如 "509"、"803"
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_course_session_no (course_id, session_no), -- 唯一约束，同一门课的第几次课只能有一条记录
    UNIQUE KEY uk_course_week_time (course_id, week_no, weekday, section), -- 唯一约束，避免同一门课在同一周同一时间片重复排课
    KEY idx_weekday_section_week (weekday, section, week_no), -- 常用索引，便于按周次和时间片查待查课程
    CONSTRAINT fk_course_session_course FOREIGN KEY (course_id) REFERENCES course(id)
);
```

备注：

- 这里按“第几次课”建表，能处理假期挪课、补课、临时调教室。
- 查课针对 `course_session`，也就是某一次具体上课，而不是只针对课程。
- 修改某一次课的信息时，其他记录跟随变化，直接读取最新的 `course_session` 数据。
- 通常正在查某节课时不会修改课程信息，这个并发问题先忽略。
- 删除策略采用物理删除。

### 9. 查课记录表

```sql
CREATE TABLE attendance_check (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    course_session_id BIGINT NOT NULL,                 -- 课程上课次数 ID，表示查的是哪一次课
    started_by_user_id BIGINT NOT NULL,                -- 首次开始查这节课的查课人内部用户 ID
    started_at DATETIME NOT NULL,                      -- 首次进入查课界面的时间
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_course_session (course_session_id),  -- 唯一约束，同一次课只允许存在一条查课记录
    KEY idx_started_by_time (started_by_user_id, started_at), -- 常用索引，便于查询某查课学生发起的查课记录
    CONSTRAINT fk_attendance_check_session FOREIGN KEY (course_session_id) REFERENCES course_session(id),
    CONSTRAINT fk_attendance_check_started_by FOREIGN KEY (started_by_user_id) REFERENCES user(id)
);
```

备注：

- 这张表表示“某一次课当前这轮查课”的主记录。
- 同一节 `course_session` 只保留一条 `attendance_check`，用于承载当前查课进度。
- 首次进入查课界面时创建记录，`started_by_user_id` 记录第一个开始查这节课的人。
- 如果查课中途异常退出，只要当前时间仍在允许查课窗口内，再次进入时继续使用这条记录。
- 如果原查课人未完成，其他查课学生也可以进入同一条记录继续补查。
- 是否允许进入查课界面，不再依赖主表状态字段；直接按 `course_session` 的下课时间判断。
- 当前课程下课 30 分钟后，查课学生端不再允许进入该门课界面；后续如需修正，由管理员在管理端修改。

### 10. 查课明细表

```sql
CREATE TABLE attendance_detail (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    attendance_check_id BIGINT NOT NULL,               -- 关联查课记录表
    user_id BIGINT NOT NULL,                           -- 被查学生内部用户 ID
    status TINYINT NOT NULL DEFAULT 0,                 -- 查课状态，0=未设置，1=签到，2=迟到，3=缺勤，4=请假
    status_set_by_user_id BIGINT DEFAULT NULL,         -- 最后一次设置当前状态的人内部用户 ID，可为查课学生或管理员
    status_set_at DATETIME DEFAULT NULL,               -- 最后一次设置当前状态的时间
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_check_student (attendance_check_id, user_id), -- 唯一约束，一次查课中同一学生只能有一条明细
    KEY idx_student_status (user_id, status),          -- 常用索引，便于查询某学生考勤情况
    KEY idx_status (status),                           -- 常用索引，便于统计签到、迟到、缺勤、请假人数
    KEY idx_status_set_by (status_set_by_user_id),     -- 常用索引，便于查询某个操作人最后设置了哪些状态
    CONSTRAINT fk_attendance_detail_check FOREIGN KEY (attendance_check_id) REFERENCES attendance_check(id),
    CONSTRAINT fk_attendance_detail_user FOREIGN KEY (user_id) REFERENCES user(id),
    CONSTRAINT fk_attendance_detail_status_set_by FOREIGN KEY (status_set_by_user_id) REFERENCES user(id)
);
```

备注：

- 这张表只保留每个学生在当前这次查课中的“最新状态”。
- 建议在创建 `attendance_check` 时，按应到学生批量初始化 `attendance_detail`，默认 `status=0`，这样异常退出后可以直接恢复进度。
- `status_set_by_user_id` 记录最后一次把当前状态写入的人，不区分查课学生还是管理员。
- 管理员后续修改结果，直接更新这里；历史变化不在本表累积。
- “请假”在本系统里只是一个考勤状态，不要求填写备注，也不负责请假审批、附件和证明管理。

### 11. 查课明细日志表

```sql
CREATE TABLE attendance_detail_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    attendance_detail_id BIGINT NOT NULL,             -- 对应的查课明细 ID
    attendance_check_id BIGINT NOT NULL,              -- 对应的查课记录 ID
    user_id BIGINT NOT NULL,                          -- 被修改学生内部用户 ID
    operator_user_id BIGINT NOT NULL,                 -- 本次修改人内部用户 ID，可为查课学生或管理员
    old_status TINYINT DEFAULT NULL,                  -- 修改前状态，首次设置时可为空
    new_status TINYINT NOT NULL,                      -- 修改后状态
    operation_type VARCHAR(50) NOT NULL,              -- 操作类型，如 "set_status"
    operated_at DATETIME NOT NULL,                    -- 实际操作时间
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    KEY idx_check_operated_at (attendance_check_id, operated_at), -- 常用索引，便于查看某次查课的修改轨迹
    KEY idx_student_operated_at (user_id, operated_at), -- 常用索引，便于查看某学生的状态变更历史
    KEY idx_operator_operated_at (operator_user_id, operated_at), -- 常用索引，便于查看某个操作人的修改记录
    CONSTRAINT fk_attendance_detail_log_detail FOREIGN KEY (attendance_detail_id) REFERENCES attendance_detail(id),
    CONSTRAINT fk_attendance_detail_log_check FOREIGN KEY (attendance_check_id) REFERENCES attendance_check(id),
    CONSTRAINT fk_attendance_detail_log_user FOREIGN KEY (user_id) REFERENCES user(id),
    CONSTRAINT fk_attendance_detail_log_operator FOREIGN KEY (operator_user_id) REFERENCES user(id)
);
```

备注：

- 这张表专门记录 `attendance_detail` 的每一次状态变更。
- `attendance_detail` 只保留最新值，完整历史以这张日志表为准。
- 查课学生查课过程中的修改和管理员后续修正，都写入这张表。

### 12. 管理员修改日志表

```sql
CREATE TABLE admin_operation_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,              -- 主键，自增 ID
    operator_user_id BIGINT NOT NULL,                  -- 操作人内部用户 ID，通常为管理员
    target_table VARCHAR(50) NOT NULL,                 -- 被修改的表名，如 "attendance_detail"
    target_id BIGINT NOT NULL,                         -- 被修改记录的主键值
    action_type VARCHAR(50) NOT NULL,                  -- 操作类型，如 "update"
    old_value TEXT DEFAULT NULL,                       -- 修改前内容，可存 JSON 字符串
    new_value TEXT DEFAULT NULL,                       -- 修改后内容，可存 JSON 字符串
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,     -- 操作时间
    KEY idx_operator_time (operator_user_id, created_at), -- 常用索引，便于查询某管理员的操作历史
    KEY idx_target (target_table, target_id),          -- 常用索引，便于查询某条业务数据的修改历史
    CONSTRAINT fk_admin_operation_log_operator FOREIGN KEY (operator_user_id) REFERENCES user(id)
);
```

备注：

- 管理员的修改记录单独存日志表，不混在业务表里。
- 这张表继续用于记录管理员对班级、课程、用户等后台数据的修改历史。
- 查课明细的状态变更历史，优先写入 `attendance_detail_log`，不要混进这张表。
- 删除策略采用物理删除。

## 目前还没考虑完整的点

### 1. 外键删除顺序

因为采用物理删除，并且表之间有外键约束，所以删除数据时要注意顺序。

建议顺序：

- 先删 `attendance_detail_log`
- 再删 `attendance_detail`
- 再删 `attendance_check`
- 再删 `course_session`
- 然后删 `course_student`、`course_class`
- 最后删 `course`

### 2. 日志表设计

- `attendance_detail_log` 采用结构化字段存储状态变更，便于直接做轨迹查询和审计。
- `admin_operation_log.old_value` 和 `admin_operation_log.new_value` 建议统一存 JSON 字符串，便于记录后台管理数据的修改前后内容。

### 3. 约束与业务代码一致性

现在文档已经约定：

- 请假只是状态，不做审批
- 请假状态不要求备注
- 修改某一次课的信息时，相关记录跟随变化
- 正在查课时通常不会改课程信息，先忽略这类并发问题
- 同一节课只允许存在一条 `attendance_check`
- 未完成的查课记录允许重新进入，且允许其他查课学生补查
- 当前课程下课 30 分钟后，查课学生端不再允许进入该门课界面
- `attendance_detail` 只保存最新状态，历史变更统一进入 `attendance_detail_log`

### 4. 账号权限约束

- 管理员账号和查课学生账号都不提供删除功能。
- 管理员可以将其他账号设置为管理员。
- 管理员可以冻结其他管理员账号。
- 管理员不能取消自己的管理员身份。
- 管理员不能冻结自己的账号。

后端实现时要和这里保持一致，不要再额外引入快照或审批流。

## 后端

目录：`wack-backend`

技术栈：

- 语言：Go 1.26（仓库已初始化）
- Web 框架：Gin（规划使用）
- 认证方式：JWT（规划使用）
- 数据库：SQLite（规划使用）
- 服务形态：单体后端服务

### 接口

统一约定：

- 基础路径前缀：`/api`
- 认证方式：登录成功后返回 JWT，后续通过 `Authorization: Bearer <token>` 传递
- 返回格式统一为：

```json
{
  "code": 0,
  "message": "ok",
  "data": {}
}
```

- `code=0` 表示成功，非 `0` 表示失败
- 时间字段统一使用 `YYYY-MM-DD HH:mm:ss`
- 除登录接口外，默认都需要鉴权

#### 1. 认证相关

`POST /api/auth/login`

- 说明：账号密码登录
- 角色：公开
- 请求体：

```json
{
  "student_id": "202231104021059",
  "password": "123456"
}
```

- 返回数据：

```json
{
  "token": "jwt-token",
  "user": {
    "id": 1,
    "student_id": "202231104021059",
    "real_name": "张三",
    "role": 2,
    "status": 1
  }
}
```

`GET /api/auth/me`

- 说明：获取当前登录用户信息
- 角色：管理员、查课学生

`POST /api/auth/change-password`

- 说明：修改当前用户密码
- 角色：管理员、查课学生
- 请求体：

```json
{
  "old_password": "123456",
  "new_password": "654321"
}
```

#### 2. 用户管理

`GET /api/users`

- 说明：分页查询用户列表，可按角色、状态、姓名、学号筛选
- 角色：管理员

查询参数：

- `page`
- `page_size`
- `role`
- `status`
- `keyword`

`POST /api/users`

- 说明：创建用户账号
- 角色：管理员
- 请求体：

```json
{
  "student_id": "202231104021059",
  "real_name": "张三",
  "password": "123456",
  "role": 2,
  "status": 1,
  "class_ids": [1, 2]
}
```

`GET /api/users/{student_id}`

- 说明：获取单个用户详情
- 角色：管理员

`PUT /api/users/{student_id}`

- 说明：更新用户基本信息、角色、状态、所属班级
- 角色：管理员

`PATCH /api/users/{student_id}/status`

- 说明：冻结或解冻账号
- 角色：管理员
- 请求体：

```json
{
  "status": 2
}
```

`PATCH /api/users/{student_id}/role`

- 说明：设置账号角色
- 角色：管理员
- 请求体：

```json
{
  "role": 1
}
```

接口约束：

- 管理员不能冻结自己
- 管理员不能取消自己的管理员身份

#### 3. 班级管理

`GET /api/classes`

- 说明：分页查询班级列表
- 角色：管理员

`POST /api/classes`

- 说明：创建班级
- 角色：管理员

`GET /api/classes/{id}`

- 说明：获取班级详情
- 角色：管理员

`PUT /api/classes/{id}`

- 说明：更新班级信息
- 角色：管理员

`DELETE /api/classes/{id}`

- 说明：删除班级
- 角色：管理员

`GET /api/classes/{id}/students`

- 说明：查看班级下学生列表
- 角色：管理员

#### 4. 查课学生空闲时间

`GET /api/free-times`

- 说明：查询空闲时间，可按学期、学生筛选
- 角色：管理员、查课学生

查询参数：

- `term`
- `student_id`，查课学生查询自己时可不传

`POST /api/free-times`

- 说明：新增空闲时间
- 角色：管理员、查课学生

`PUT /api/free-times/{id}`

- 说明：更新空闲时间
- 角色：管理员、查课学生

`DELETE /api/free-times/{id}`

- 说明：删除空闲时间
- 角色：管理员、查课学生

接口约束：

- 查课学生只能管理自己的空闲时间

#### 5. 课程管理

`GET /api/courses`

- 说明：分页查询课程列表，可按学期、课程名、教师筛选
- 角色：管理员

`POST /api/courses`

- 说明：创建课程主信息
- 角色：管理员

`GET /api/courses/{id}`

- 说明：获取课程详情，包含课程主信息、上课班级、上课学生、上课次数
- 角色：管理员

`PUT /api/courses/{id}`

- 说明：更新课程主信息
- 角色：管理员

`DELETE /api/courses/{id}`

- 说明：删除课程
- 角色：管理员

`PUT /api/courses/{id}/students`

- 说明：整体更新课程上课学生列表
- 角色：管理员
- 请求体：

```json
{
  "student_ids": ["202231104021059", "202231104021060"]
}
```

`PUT /api/courses/{id}/classes`

- 说明：整体更新课程关联班级列表
- 角色：管理员

`PUT /api/courses/{id}/sessions`

- 说明：整体更新课程上课次数列表
- 角色：管理员

#### 6. 管理员课程看板

`GET /api/admin/course-calendar`

- 说明：按周返回课程表数据，用于管理员端全院课程表
- 角色：管理员

查询参数：

- `term`
- `week_no`

`GET /api/admin/attendance-dashboard`

- 说明：返回查课结果看板统计数据
- 角色：管理员

查询参数：

- `term`
- `week_no`
- `course_id`

`GET /api/admin/attendance-results`

- 说明：分页查询查课结果列表，可按课程、教师、周次、状态筛选
- 角色：管理员

`GET /api/admin/free-time-calendar`

- 说明：按周返回查课学生空余时间日历数据
- 角色：管理员

查询参数：

- `term`
- `week_no`

#### 7. 查课学生端课程与查课流程

`GET /api/student/courses/available`

- 说明：返回当天可进入的课程列表
- 角色：查课学生

返回字段建议包含：

- `course_session_id`
- `course_id`
- `course_name`
- `teacher_name`
- `week_no`
- `weekday`
- `section`
- `building_name`
- `room_name`
- `started_at`
- `can_enter`
- `enter_deadline`

接口约束：

- 只返回当天课程
- 当前课程下课 30 分钟后，不再返回为可进入状态
- 如果该 `course_session` 已存在 `attendance_check`，需要返回当前查课进度摘要，便于前端显示“继续查课”

`POST /api/student/attendance-checks`

- 说明：进入某门课查课界面；若记录已存在则直接复用，不重复创建
- 角色：查课学生
- 请求体：

```json
{
  "course_session_id": 1001
}
```

返回字段建议包含：

- `attendance_check_id`
- `course_session`
- `course`
- `students`

接口约束：

- 若当前时间已超过该课程下课后 30 分钟，返回禁止进入
- 首次进入时创建 `attendance_check`
- 首次进入时按应到学生批量初始化 `attendance_detail`
- 非首次进入时直接返回已有查课记录与明细

`GET /api/student/attendance-checks/{id}`

- 说明：获取查课详情，用于恢复进度
- 角色：查课学生

接口约束：

- 若已超过进入截止时间，接口可返回只读信息，或直接返回禁止进入；后端实现时二选一，但需全局保持一致

`PATCH /api/student/attendance-details/{id}/status`

- 说明：更新单个学生的查课状态
- 角色：查课学生
- 请求体：

```json
{
  "status": 1
}
```

接口约束：

- 仅允许在进入截止时间前修改
- 每次修改都同步更新 `attendance_detail.status`、`status_set_by_user_id`、`status_set_at`
- 每次修改都写入 `attendance_detail_log`

`POST /api/student/attendance-checks/{id}/complete`

- 说明：查课学生主动完成查课
- 角色：查课学生

接口约束：

- 该接口只表示“前端结束本次查课操作”，不再要求在 `attendance_check` 表记录完成人、完成时间、完成状态
- 若当前时间已超过进入截止时间，可直接返回禁止操作
- 后续是否允许再次进入，统一仍按“当前课程下课 30 分钟后不允许进入”判断

#### 8. 管理员修改查课结果

`GET /api/admin/attendance-checks/{id}`

- 说明：管理员查看某次查课详情
- 角色：管理员

`PATCH /api/admin/attendance-details/{id}/status`

- 说明：管理员修改单个学生查课状态
- 角色：管理员
- 请求体：

```json
{
  "status": 4
}
```

接口约束：

- 修改 `attendance_detail` 当前值
- 写入 `attendance_detail_log`
- 同时写入 `admin_operation_log`

`GET /api/admin/attendance-details/{id}/logs`

- 说明：查看某条查课明细的状态变更日志
- 角色：管理员

#### 9. 日志与审计

`GET /api/admin/operation-logs`

- 说明：分页查询管理员操作日志
- 角色：管理员

`GET /api/admin/attendance-detail-logs`

- 说明：分页查询查课明细状态变更日志
- 角色：管理员

#### 10. 后端实现约束

- `attendance_check` 不再通过 `finished_by`、`finished_at`、`status` 判断流程状态
- 查课是否可进入，统一通过课程实际下课时间加 30 分钟判断
- `attendance_detail` 和 `attendance_detail_log` 不再处理备注字段
- “请假”只是状态枚举之一，不要求备注
- 管理员修改查课结果时，学生端相关限制不适用

## 前端

目录：`wack-frontend`

技术栈：

- Vue 3
- TypeScript
- Vite

### 界面

#### 通用界面

- 登录界面
  - 字段：学号、密码
- 修改密码界面
  - 字段：旧密码、新密码、确认新密码

#### 管理员端（PC）

- 全院课程表
  - 日历视图
  - 顶部显示周数切换
  - 表格首行显示周一到周日，首列显示上午 1-2 节到晚上 9-10 节
  - 每个格子内展示课程标签，支持点击查看课程详情
- 课程管理
  - 列表视图
  - 支持查看、创建、修改、删除课程
  - 支持维护课程上课学生
- 班级管理
  - 列表视图
  - 支持创建、编辑、查看班级信息
- 查课结果看板
  - 可视化界面
  - 当前阶段先预留空白页，后续补图表与统计卡片
- 查课结果列表
  - 列表视图
  - 支持查看应到、已到、未到和请假明细
  - 支持修改某个学生的查课状态为签到、迟到、缺勤、请假
- 查课学生空余时间日历
  - 日历视图
  - 按时段查看哪些查课学生空闲
- 查课学生空余时间管理
  - 列表视图
  - 支持新增、修改、删除查课学生空余时间
- 用户管理
  - 列表视图
  - 支持创建账号、冻结账号、解冻账号
  - 支持将其他账号设置为管理员
  - 支持冻结或解冻其他管理员账号

#### 查课学生端（移动端）

- 可查课程
  - 列表视图
  - 展示当天待查课程的时间、地点、教师、课程名称等信息
  - 点击课程后进入查课流程；若该课程已有查课记录且仍在允许进入时间内，则直接恢复到上次进度
- 查课流程
  - 顶部提供学生姓名搜索框
  - 中间为学生列表，展示姓名和当前状态
  - 支持通过下拉框直接修改状态
  - 底部提供签到、迟到、缺勤、请假快捷按钮
  - 点击快捷按钮后，将当前焦点学生更新为对应状态，并自动切换到下一个学生
  - 支持临时退出后重新进入，继续未完成的查课记录
  - 若原查课人未完成，其他查课学生可进入同一记录继续补查
  - 点击“完成查课”后，结束本次前端查课操作；不再单独落库记录完成状态
  - 课程结束 30 分钟后，不再允许查课学生进入该门课界面
  - 抽查模式后续单独设计，不复用当前逐个学生落库规则
- 空闲时间修改
  - 日历视图
  - 顶部不显示周数
  - 支持左右滑动，每次聚焦一列时间片
  - 点击单元格后可编辑该时间片对应的空闲周数
