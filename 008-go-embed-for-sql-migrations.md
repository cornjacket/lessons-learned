# Lesson 008: //go:embed for SQL Migrations in Containers

## The Problem

SQL migration files live on disk during development (`internal/services/foo/migrations/*.sql`). In production, the Go binary runs inside a minimal container (e.g., `gcr.io/distroless/static`) that has no filesystem, no shell, and no way to read external files.

Common workarounds — copying SQL files into the container, using Alpine for shell access, or running migrations from a bastion host — all add complexity, attack surface, or operational burden.

## The Pattern

Go's `//go:embed` directive compiles files into the binary at build time:

```go
package ingestion

import "embed"

//go:embed migrations/*.sql
var MigrationFS embed.FS
```

At runtime, `MigrationFS` is an `fs.FS` containing the SQL files — no disk reads, no external dependencies. Migration libraries like `golang-migrate/migrate` and `pressly/goose` both accept `io/fs` sources, so the embedded FS plugs in directly:

```go
source, _ := iofs.New(ingestion.MigrationFS, "migrations")
m, _ := migrate.NewWithSourceInstance("iofs", source, databaseURL)
m.Up()
```

## Why This Matters

- **Single artifact:** The binary contains everything — code, migrations, config defaults. Nothing else needed at runtime.
- **Distroless compatible:** No shell, no filesystem, no package manager. The binary is the only file in the container.
- **Zero-touch deployment:** Start the binary → migrations run → services start. No operator, no sidecar script, no init container.
- **Identical behavior everywhere:** Local dev, Docker, ECS, Lambda — the same binary applies the same migrations.

## The Key Insight

`//go:embed` turns deployment-time dependencies (files on disk) into compile-time dependencies (bytes in the binary). This eliminates an entire class of "file not found" production failures and removes the need for any migration infrastructure beyond the binary itself.

## When to Use

- Any Go service that owns database migrations
- Container deployments where filesystem access is restricted
- Serverless (Lambda) where there's no persistent filesystem
- Any context where you want a single self-contained binary

## When Not to Use

- Migrations that need to be modified after compilation (hotfixes to SQL)
- Non-Go services (the pattern is Go-specific)
- Development-only tooling where disk reads are fine
