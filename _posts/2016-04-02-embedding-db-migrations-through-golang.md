---
layout: post
title: "Embedding DB migrations through Golang build tags"
date: 2016-04-02 18:25:00 +0530
tags: [golang, database]
---

In this post I am going to explain how to embed database migration sqls within application binary and how can we utilize the Golang build tags to maintain both embedded and non-embedded versions of database migrations.

I am developing a system application in Golang that uses SQLite as database. It is destined to run on user's machine. The application runs database migrations at every startup. In a typical web application the migration sql files are usually deployed along with the web application so that it can migrate the database next time when it is launched. But in my case as it is an application distributed to the user, I don't want to keep the migrations as separate sql files. So I decided to embed the migration sqls with the application binary. I also took advantage of the Golang build tags to maintain both embedded and non-embedded versions.

## Embedding Migration SQLs

I've used [goose](https://bitbucket.org/liamstask/goose/) and [migrate](https://github.com/mattes/migrate) libraries in an earlier projects. But both libraries didn't support embedding the migration sqls within application binary. Quick search revealed that [sql-migrate](https://github.com/rubenv/sql-migrate) library can do both non-embedded and embedded migrations. We need to convert the migration sqls into Go source files using [go-bindata](https://github.com/jteeuwen/go-bindata) and instruct sql-migrate to use embedded sqls.

The following command converts all migration sqls from "db/migrations" directory into Go source file named "bindata.go" with package name of "myapp".

```bash
$ go-bindata -pkg myapp -o bindata.go db/migrations/
```

The "bindata.go" exports two functions named "Asset" and "AssetDir". These functions are used to retrieved the embedded file and file list of a embedded directory.

The following code snippet wires up these functions with sql-migrate for it retrieve the embedded migration sqls.

```go
...
migrations := &migrate.AssetMigrationSource{
    Asset:    Asset,
    AssetDir: AssetDir,
    Dir:      "db/migrations",
}
...
```

## Embedding with Build Tags

I want to embed the migrations only for production build. I want to use non-embedded migrations for development builds to avoid the additional step of running go-bindata every time the migration sql changes. I achieved this workflow with help of Go build tags.

Go build tags are originally created to select platform specific source files while building multi-platform applications/libraries. The mechanism can be used for other purposes too, like my usage of embedding migration sqls. All build related go commands(e.g. go build, go test) supports specifying build tags.

So to achieve the above workflow I've created two migration source files, one for non-embedded version and another one for embedded version.

migrations_non_embedded.go

```go
// +build !embedded_migrations
package db 
import (
        "github.com/rubenv/sql-migrate"
)

func migrations() migrate.MigrationSource {
        return &migrate.FileMigrationSource{
                Dir: "db/migrations",
        }
}
```

migrations_embedded.go

```go
// +build embedded_migrations
package db 
import (
        "github.com/rubenv/sql-migrate"
)

func migrations() migrate.MigrationSource {
        return &migrate.AssetMigrationSource{
                Asset:    myapp.Asset,
                AssetDir: myapp.AssetDir,
                Dir:      "db/migrations",
        }
}
```

In the migration source files, the build tag "+build !embedded_migrations" instructs the build tool to use the source if the build tag "embedded_migrations" is not specified. Similarly the build tag "+build embedded_migrations" instructs the build tool to use the source if the build tag "embedded_migrations" is specified. Basically only one of the source file is used by the build tool based on the presence/absence of the build tag.

Regardless whether the migration sqls are embedded or not, we can retrieve the migration sqls by simply calling "db.migrations()" function and upgrade the database like below.

main.go
```go
...
n, err := migrate.Exec(db, "sqlite3", db.migrations(), migrate.Up)
if err != nil {
    log.Fatal("db migrations failed: ", err)
}
...
```

Now to build the application with non-embedded migrations we can run the following command

```bash
$ go build
```

and to build with embedded migrations for production/deployment we can run the following command.

```bash
$ go-bindata -pkg myapp -o bindata.go db/migrations/
$ go build -tags embedded_migrations
```