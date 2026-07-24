---
title: "数据库结构版本管理"
date: 2026-07-24T10:00:00+08:00
draft: false
tags: ["数据库", "DB Migration", "工程规范", "dbmate"]
categories: ["技术文章"]
---

## 为什么数据库也需要版本管理？

假设你开发新功能时，在本地执行了：

```sql
ALTER TABLE users ADD COLUMN nickname VARCHAR(100);
```

代码提交并部署到测试环境后，接口却报错：

```text
column "nickname" does not exist
```

原因很简单：代码进入了 Git，修改数据库的 SQL 却只存在于你的电脑里。

如果数据库变更依靠聊天记录或人工执行，我们就很难回答：

- 哪些环境执行过这条 SQL？
- 新环境应该按什么顺序初始化？
- 测试库和生产库的结构是否一致？
- 三个月前为什么增加了这个字段？

解决办法是 **DB Migration（数据库迁移）**：把每次数据库结构变化写成带版本号的文件，提交到 Git，再由工具按顺序执行。

---

## Migration 是怎样工作的？

可以把 Migration 理解成“数据库的 Git Commit”：

```text
20260724100000  创建 users 表
20260724103000  增加 nickname 字段
20260724110000  创建邮箱索引
```

迁移工具会：

1. 读取 Migration 目录；
2. 查询数据库已经执行的版本；
3. 按顺序执行尚未执行的文件；
4. 记录成功执行的版本。

几个容易混淆的概念：

| 概念 | 用途 |
|---|---|
| Migration | 记录数据库如何一步步变化 |
| Schema 快照 | 展示数据库当前的完整结构 |
| 数据备份 | 在数据损坏或丢失后恢复 |

Migration 不能代替备份。执行 `DROP COLUMN` 后，Down Migration 也不一定能找回已经删除的数据。

ORM 自动建表也不完全等于版本管理。ORM 可以帮助生成 Migration，但共享环境最好执行已经生成、评审并提交到 Git 的确定性 SQL，而不是在应用启动时临时修改数据库。

---

## 推荐的目录结构

数据库迁移最好放在独立目录中，不依赖具体 Web 框架：

```text
db/
├── migrations/
│   ├── 20260724100000_create_users_table.sql
│   └── 20260724103000_add_bio_to_users.sql
├── schema.sql
├── seeds/
└── README.md
```

- `migrations/`：所有数据库变更，是最重要的历史记录；
- `schema.sql`：数据库当前结构的完整快照；
- `seeds/`：必要的初始数据或开发测试数据；
- `README.md`：团队的执行、发布和故障处理约定。

这样做以后，Java、Go、PHP、Python 或 Node.js 项目都可以使用同一套数据库规范。

---

## 使用 dbmate 完成第一次迁移

[dbmate](https://github.com/amacneil/dbmate) 是一个轻量、与框架无关的命令行工具，默认使用 `db/migrations/`，很适合入门。

### 1. 准备 PostgreSQL

如果本机没有 PostgreSQL，可以临时使用 Docker：

```bash
docker run --name migration-demo-postgres \
  -e POSTGRES_USER=app \
  -e POSTGRES_PASSWORD=app_password \
  -e POSTGRES_DB=app \
  -p 5432:5432 \
  -d postgres:17-alpine
```

示例密码只用于本地实验，不要复制到生产环境。

### 2. 安装并连接

macOS 可以使用：

```bash
brew install dbmate
```

也可以安装到 Node.js 项目：

```bash
npm install --save-dev dbmate
```

设置本地数据库连接：

```bash
export DATABASE_URL='postgres://app:app_password@127.0.0.1:5432/app?sslmode=disable'
```

真实项目应该通过本机环境、CI Secret 或密钥管理系统注入密码，不要把生产连接信息提交到 Git。

### 3. 创建 Migration

执行：

```bash
dbmate new create_users_table
```

dbmate 会生成类似下面的文件：

```text
db/migrations/20260724100000_create_users_table.sql
```

写入：

```sql
-- migrate:up
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrate:down
DROP TABLE users;
```

然后执行：

```bash
dbmate up
dbmate status
```

dbmate 会创建 `users` 表，并把版本号记录到 `schema_migrations`。如果本机安装了 `pg_dump`，还会更新 `db/schema.sql`。

### 4. 后续变更必须新增文件

现在需要增加个人简介字段。不要修改上一份 Migration，而是执行：

```bash
dbmate new add_bio_to_users
```

```sql
-- migrate:up
ALTER TABLE users ADD COLUMN bio TEXT;

-- migrate:down
ALTER TABLE users DROP COLUMN bio;
```

再次执行 `dbmate up`，所有环境就会按相同顺序得到一致的结构。

本地开发可以使用：

```bash
dbmate rollback
```

回滚最近一次变更。但生产环境中的 Down 可能删除数据，不能把它当成万能撤销按钮。

---

## 团队应该遵守的核心规范

### 1. 已执行的 Migration 永远不要修改

老环境不会重新执行已经成功的版本，新环境却会执行修改后的内容，最终产生两套不同结构。发现问题时应该新增 Migration，而不是改写历史。

dbmate 只记录版本号，不保存文件内容，因此这个约束需要通过代码评审和 CI 保证。Flyway 等工具还会保存 Checksum，帮助发现历史文件被修改。

### 2. 一项独立变更对应一个文件

推荐：

```text
create_users_table.sql
add_bio_to_users.sql
create_user_email_index.sql
```

避免使用含义模糊的 `update_database.sql`。文件越小，越容易评审和定位故障。

### 3. 使用全局唯一版本号

可以使用递增整数或时间戳。多人并行开发时，时间戳更不容易冲突。

### 4. Schema 和大规模数据迁移分开

增加字段与全表更新的风险不同。大批量 `UPDATE` 可能长时间占锁，应拆成可暂停、可重试、可限速的批处理任务。

### 5. Migration 必须进入代码评审

至少检查：

- 是否存在删表、删字段等破坏性操作；
- 是否可能长时间锁表；
- 字段类型、默认值和索引是否合理；
- 新旧版本应用能否同时运行；
- 失败后如何向前修复。

### 6. CI 必须从空库重放

CI 应启动空数据库，从第一份 Migration 执行到最新版本，并确认没有待执行版本。如果条件允许，再验证从上一个正式版本升级。

### 7. 生产环境只能有一个执行者

不要让多个 Web 实例在启动时同时迁移。推荐使用独立的 CI/CD Job 或 Kubernetes Job：

```text
执行 Migration → 确认成功 → 部署应用
```

### 8. 生产故障优先向前修复

生产 Down Migration 可能再次删除数据。更稳妥的做法通常是停止发布、确认数据库实际状态，再新增一份修复 Migration。

---

## 生产环境怎样安全改字段？

假设要把 `nickname` 改成 `display_name`，直接重命名会导致旧版本应用报错。

更安全的 **Expand and Contract（扩展与收缩）** 流程是：

1. 新增 `display_name`，暂时保留 `nickname`；
2. 应用同时兼容两个字段；
3. 分批把旧数据回填到新字段；
4. 所有应用切换到新字段；
5. 确认旧版本下线后，再删除 `nickname`。

```text
增加新结构 → 应用兼容 → 回填数据 → 应用切换 → 删除旧结构
```

这个过程可能跨越多次发布，但能避免应用和数据库在滚动发布期间互不兼容。

---

## 常见工具怎样选择？

| 工具 | 特点 | 适合场景 |
|---|---|---|
| dbmate | 轻量 CLI、纯 SQL、自动生成 Schema 快照 | 跨语言项目和小团队 |
| [golang-migrate](https://github.com/golang-migrate/migrate) | 简洁的 CLI 和 Go Library | 只需要迁移执行器 |
| [Goose](https://github.com/pressly/goose) | 支持 SQL 和 Go 函数 | Go 项目 |
| [Flyway Community](https://github.com/flyway/flyway) | Checksum、Repeatable Migration、数据库支持广 | 中大型团队 |
| [Liquibase Community](https://github.com/liquibase/liquibase) | Changeset、标签、上下文、多种文件格式 | 复杂发布和审计流程 |
| [Atlas](https://atlasgo.io/concepts/declarative-vs-versioned) | 声明式 Schema、版本化 Migration、自动 Diff | Schema-as-Code |

工具可以替换，但历史不可修改、变更必须评审、CI 完整重放和生产单执行器等规范应该保持不变。

---

## 总结

数据库版本管理最重要的不是选择哪个工具，而是建立一个共识：

> **数据库结构也是代码，必须进入版本管理、代码评审和自动化发布流程。**

刚开始可以先做好四件事：

1. 建立独立的 `db/migrations/`；
2. 所有数据库变更都新增 Migration；
3. 使用 CI 从空库重放完整历史；
4. 使用独立、单一的任务执行生产迁移。

做到这些，就能告别“上线时记得手动执行一下 SQL”。
