# Batch 04.5 — Migration auto-apply on boot

状态：**契约草稿** ⏳（待用户 approve）
更新时间：2026-06-06
范围：技术债清理，无新业务接口、无前端改动。

---

## 业务背景

Batch 4 的 migration 000004（`brands.description`）在本地从未应用，直到业务验收阶段才发现。Makefile 硬编码 `postgres:postgres` 凭据 vs 开发机实际 `liushan` superuser 导致 `make migrate-up` 静默 no-op。**Batch 5+ migration 频次只会增加**（Staff/Instructor/Course 等多个域可能需要补字段），所以起跑前必须把 schema drift 风险消掉。

---

## Grill 阶段决定（详见会话）

1. **三个 cmd 都跑 migration**，靠 golang-migrate 的 advisory lock 互斥；保留部署顺序自由
2. **`//go:embed migrations/*.sql`** 打包到二进制，运行时不依赖 cwd
3. **环境差异化默认**：dev yaml `auto_migrate_on_boot: true`；生产 yaml `false`，需要 Railway 显式 env 才开
4. **Makefile DSN** 改为读 `DATABASE_URL`，fallback 用当前 OS user
5. **失败硬阻塞**：parse 错 / dirty / DB 不可达 → `log.Fatal`；up-to-date → log + 继续

---

## 改动清单

### 1. config 扩展

`backend/internal/infrastructure/config/config.go`：
- 在 `DatabaseConfig` 加 `AutoMigrateOnBoot bool` 字段
- `keysToBind` 加 `"database.auto_migrate_on_boot"`

`backend/configs/config-brand.yaml` / `config-admin.yaml` / `config.yaml` / `config-app.yaml`：
- 默认 `database.auto_migrate_on_boot: true`（dev / mock 模式）

生产环境（Railway）通过 env `MINI_SCHEDULE_DATABASE_AUTO_MIGRATE_ON_BOOT=true` 显式开启；不设就是 false（更安全）。

### 2. migrations embed + runner

新文件 `backend/internal/infrastructure/database/migrate.go`：

```go
package database

import (
    "embed"
    "errors"
    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    "github.com/golang-migrate/migrate/v4/source/iofs"
)

//go:embed all:migrations/*.sql 不行，embed 必须在引用位置——放 cmd/ 不合适。
// 实际：把 migrations/ 复制 / 软链或者在 backend/ 根加一个 embed package。

func RunMigrationsUp(dsn string, fs embed.FS) error {
    src, err := iofs.New(fs, "migrations")
    if err != nil { return err }
    m, err := migrate.NewWithSourceInstance("iofs", src, dsn)
    if err != nil { return err }
    if err := m.Up(); err != nil && !errors.Is(err, migrate.ErrNoChange) {
        return err
    }
    return nil
}
```

新文件 `backend/migrations/embed.go`：

```go
package migrations
import "embed"
//go:embed *.sql
var FS embed.FS
```

### 3. 三个 cmd main.go 集成

`cmd/api-{brand,admin,app}/main.go`：

```go
if cfg.Database.AutoMigrateOnBoot {
    dsn := buildDSN(cfg.Database)
    if err := database.RunMigrationsUp(dsn, migrations.FS); err != nil {
        log.Fatal("migration failed: ", err)
    }
    log.Info("migrations applied or already up to date")
}
```

集成位置：DB 连接之后、Gin 启动之前。

### 4. Makefile DSN 改造

```makefile
DATABASE_URL ?= postgres://$(or $(PG_USER),$(USER))@127.0.0.1:5432/mini_schedule?sslmode=disable

migrate-up:
	migrate -path migrations -database "$(DATABASE_URL)" up

migrate-down:
	migrate -path migrations -database "$(DATABASE_URL)" down 1
```

用户可 `DATABASE_URL=... make migrate-up` 自定义；默认尝试当前 OS user（即 `liushan`），匹配开发机。

### 5. 依赖

`go.mod` 新增：
- `github.com/golang-migrate/migrate/v4`（含 `database/postgres` + `source/iofs` 子包）

---

## 验收

- [ ] `go build ./...` + `go vet ./...` 通过
- [ ] 拉 dev DB 到 version 3，启动 api-brand，日志看 "migrations applied"，schema_migrations.version 变为 4
- [ ] 再次启动 api-brand，日志看 "schema up to date"，version 不变
- [ ] `make migrate-up`（用默认 `${USER}`）应能直接跑通，无需指定 postgres user
- [ ] 生产 yaml 默认 `auto_migrate_on_boot: false`；通过 env 覆盖能开启
- [ ] 故意制造 dirty（手动 `UPDATE schema_migrations SET dirty=true;`），启动应 `log.Fatal` 拒绝运行

---

## 不做

- 不做 down/rollback 自动化（破坏性，留人工）
- 不接 Railway 部署侧的 migration job（用 boot-time + env 已够）
- 不改 migration 文件格式 / 工具 / 版本号体系

---

## 等你 approve

逐条回复 OK / 修改：

1. 三个 cmd 各自跑 migration（靠 advisory lock 互斥）OK 吗？还是想拆出独立 migrator binary？
2. dev 默认 `true` / 生产默认 `false`，生产靠 env 显式开 OK 吗？
3. Makefile DSN 用 `$(USER)` 兜底 OK 吗？还是想强制传 `DATABASE_URL`？
4. dirty 状态 → log.Fatal 拒绝启动，OK？（更安全，但首次开发可能踩坑）

逐条 OK 后我会：
- 实现 + 逐 task commit
- 跑静态验证（build/vet）
- 手测 happy path（DB version 3→4→重启不变）
- 发邮件请你做业务验收（清单上面的 6 条）
