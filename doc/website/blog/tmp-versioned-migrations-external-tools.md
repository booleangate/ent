---
title: Versioned Migrations Management and Migration Directory Integrity File
author: MasseElch
authorURL: "https://github.com/masseelch"
authorImageURL: "https://avatars.githubusercontent.com/u/12862103?v=4"
image: "TBD"
---

Five weeks ago we released a long awaited feature for managing database changes in Ent: **Versioned Migrations**. In
the [announcing blog post](2022-03-14-announcing-versioned-migrations.md) we gave a brief introduction into both the
declarative and change-based approach to keep database schemas in sync with the consuming applications, as well as their
drawbacks and why [Atlas'](https://atlasgo.io) (Ents underlying migration engine) attempt of bringing the best of both
worlds into one workflow is worth a try. We call it **Versioned Migration Authoring** and if you haven't read it, now is
a good time!

With versioned migration authoring, the resulting migration files are still "change-based", but have been safely planned
by the Atlas engine. This means, that you can still use your favorite and used-to migration management tool,
like [Flyway](https://flywaydb.org/), [Liquibase](https://liquibase.org/), 
[golang-migrate/migrate](https://github.com/golang-migrate/migrate), or 
[pressly/goose](https://github.com/pressly/goose) when developing services with Ent.

In this blog post I want to show you another new feature of the Atlas we call **Migration Directory Integrity File**,
which now also has support in Ent and how you can use it with any of the migration management tools you are already used
to and like. 

### The Problem

Using versioned migration has three major downsides a developer has to be aware of in order to not break the database.

1. History should not be changed
2. The order matters
3. SQL semantics are not checked automatically

While both of the above should be detected in basic code review, it can slip the human eye, as it is error-prone.
Therefore, an automated solution is a nice-to-have safety guard.

The first issue is addressed by most management tools by saving a hash of the applied migration file to the managed
database and comparing them with the files. If they don't match, the migration can be aborted. But this happens in a
very late stage in a features' development cycle, and it could save both time and resources if this can be detected
earlier.

For the second (and third) issue, have a look at the following image:

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/no-conflict.svg)

This diagram shows two possible errors, that go undetected. The first one being the order of the migration files. 

Team A and Team B both branch a feature roughly the same time. Team B generates a migration file with version
timestamp **x** and continues to work on the feature. Team A generates a migration file at a later point in time and
therefore has migration version timestamp **x+1**. But Team A is done with the feature and merges it into master,
possibly automatically deploying it in production with the migration version **x+1** applied. No problem so far.

Now Team B merges its feature with migration version **x**, which predates already applied version **x+1**. If the code
review process does not detect this, the migration file lands in production, and it now depends on the used migration
management tool what happens.

Most tools have their own solution to that problem, `pressly/goose` for example takes an approach they
call [hybrid versioning](https://github.com/pressly/goose/issues/63#issuecomment-428681694). Before I introduce you to
Atlas' (Ents) unique way of handling this problem, let's have a quick look at the third issue:

If both Team A and Team B develop a feature where they need new tables or columns, and they name them the same, (e.g.
`users`) they could both generate a statement to create that table. While the team that merges first will have a
successful migration, the second teams' migration will fail since the table or column already exists.

### The Solution

Atlas has a unique way of handling the above problems. The goal is to raise awareness about the issues as soon as
possible. In our opinion, the best place for this are the version control and continuous integration (CI) parts of a
product. Atlas' solution to this is the introduction of a new file we call the **Migration Directory Integrity File**.
It simply is another file named `atlas.sum` that is stored together with the migration files and contains some
meta-data about the migration directory. Its format is inspired by the `go.sum` file of a Go module, and it would look
similar to this: 

```text
h1:KRFsSi68ZOarsQAJZ1mfSiMSkIOZlMq4RzyF//Pwf8A=
20220318104614_team_A.sql h1:EGknG5Y6GQYrc4W8e/r3S61Aqx2p+NmQyVz/2m8ZNwA=

```

The `atlas.sum` file contains a sum of the whole directory as it's first entry, and a checksum for each of the migration
files (implemented by a reverse, one branch merkle hash tree). Let's see how we can use this file to detect the cases
above in version control and CI. Our goal is to raise awareness, that both teams added migrations and that they most
likely have to be checked before proceeding the merge.

![atlas-versioned-migrations-no-conflict](https://entgo.io/images/assets/migrate/conflict.svg)

:::note
To follow along, run the following commands to quickly have an example to work with:

```shell
mkdir ent-sum-file
cd ent-sum-file
go mod init ent-sum-file
go run -mod=mod entgo.io/ent/cmd/ent@master init User
go generate ./...
```
:::

The first step is to tell the migration engine to create and manage the `atlas.sum` by using the `schema.WithSumFile()`
option. The below example uses and [instantiated Ent client](/docs/versioned-migrations.md#from-client) to generate new
migration files:

```go
package main

import (
	"context"
	"log"
	"os"

	"<project>/ent"

	"ariga.io/atlas/sql/migrate"
	"entgo.io/ent/dialect/sql/schema"
	_ "github.com/go-sql-driver/mysql"
)

func main() {
	client, err := ent.Open("mysql", "root:pass@tcp(localhost:3306)/entdb")
	if err != nil {
		log.Fatalf("failed connecting to mysql: %v", err)
	}
	defer client.Close()
	ctx := context.Background()
	// Create a local migration directory.
	dir, err := migrate.NewLocalDir("migrations")
	if err != nil {
		log.Fatalf("failed creating atlas migration directory: %v", err)
	}
	// Write migration diff.
	// highlight-start
	err = client.Schema.NamedDiff(ctx, os.Args[1], schema.WithDir(dir), schema.WithSumFile())
	// highlight-end
	if err != nil {
		log.Fatalf("failed creating schema resources: %v", err)
	}
}
```

If you want to follow along, you can use the below very basic schema.

```go title="ent/schema/user.go"
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
)

// User holds the schema definition for the User entity.
type User struct {
	ent.Schema
}

// Fields of the User.
func (User) Fields() []ent.Field {
	return []ent.Field{
		field.Int("team_a_col"),
	}
}

```



In addition to the migration files generated, the `atlas.sum` file will be there as well. 

Now the above given examples will all raise a
merge-conflict in the version control, since there are no longer just new files added, but the same line in a file is
changed. Of course, this only works if the developer is keeping the `atlas.sum` file up-to-date. The [Atlas CLI](https://atlasgo.io/cli/getting-started/setting-up) 

Since
Team A generates and commits their migration before Team B does Team As migration file ends up on the master branch and
possibly in production. When Team B now commits their migration, which has a version-timestamp predating the 

[//]: # ()
[//]: # ()
[//]: # ()
[//]: # (Initially, Atlas supported a style of managing database schemas that we call "declarative migrations". With declarative)

[//]: # (migrations, the desired state of the database schema is given as input to the migration engine, which plans and executes)

[//]: # (a set of actions to change the database to its desired state. This approach has been popularized in the field of)

[//]: # (cloud native infrastructure by projects such as Kubernetes and Terraform. It works great in many cases, in)

[//]: # (fact it has served the Ent framework very well in the past few years. However, database migrations are a very sensitive)

[//]: # (topic, and many projects require a more controlled approach.)

[//]: # ()
[//]: # (For this reason, most industry standard solutions, like [Flyway]&#40;https://flywaydb.org/&#41;)

[//]: # (, [Liquibase]&#40;https://liquibase.org/&#41;, or [golang-migrate/migrate]&#40;https://github.com/golang-migrate/migrate&#41; &#40;which is)

[//]: # (common in the Go ecosystem&#41;, support a workflow that they call "versioned migrations".)

[//]: # ()
[//]: # (With versioned migrations &#40;sometimes called "change base migrations"&#41; instead of describing the desired state &#40;"what the)

[//]: # (database should look like"&#41;, you describe the changes itself &#40;"how to reach the state"&#41;. Most of the time this is done )

[//]: # (by creating a set of SQL files containing the statements needed. Each of the files is assigned a unique version and a)

[//]: # (description of the changes. Tools like the ones mentioned earlier are then able to interpret the migration files and to)

[//]: # (apply &#40;some of&#41; them in the correct order to transition to the desired database structure.)

[//]: # ()
[//]: # (In this post, I want to showcase a new kind of migration workflow that has recently been added to Atlas and Ent. We call)

[//]: # (it "versioned migration authoring" and it's an attempt to combine the simplicity and expressiveness of the declarative)

[//]: # (approach with the safety and explicitness of versioned migrations. With versioned migration authoring, users still)

[//]: # (declare their desired state and use the Atlas engine to plan a safe migration from the existing to the new state.)

[//]: # (However, instead of coupling the planning and execution, it is instead written into a file which can be checked into)

[//]: # (source control, fine-tuned manually and reviewed in normal code review processes.)

[//]: # ()
[//]: # (As an example, I will demonstrate the workflow with `golang-migrate/migrate`. )

[//]: # ()
[//]: # (### Getting Started)

[//]: # ()
[//]: # (The very first thing to do, is to make sure you have an up-to-date Ent version:)

[//]: # ()
[//]: # (```shell)

[//]: # (go get -u entgo.io/ent@master)

[//]: # (```)

[//]: # ()
[//]: # (There are two ways to have Ent generate migration files for schema changes. The first one is to use an instantiated Ent)

[//]: # (client and the second one to generate the changes from a parsed schema graph. This post will take the second approach,)

[//]: # (if you want to learn how to use the first one you can have a look at)

[//]: # (the [documentation]&#40;./docs/versioned-migrations#from-client&#41;.)

[//]: # ()
[//]: # (### Generating Versioned Migration Files)

[//]: # ()
[//]: # (Since we have enabled the versioned migrations feature now, let's create a small schema and generate the initial set of)

[//]: # (migration files. Consider the following schema for a fresh Ent project:)

[//]: # ()
[//]: # (```go title="ent/schema/user.go")

[//]: # (package schema)

[//]: # ()
[//]: # (import &#40;)

[//]: # (	"entgo.io/ent")

[//]: # (	"entgo.io/ent/schema/field")

[//]: # (	"entgo.io/ent/schema/index")

[//]: # (&#41;)

[//]: # ()
[//]: # (// User holds the schema definition for the User entity.)

[//]: # (type User struct {)

[//]: # (	ent.Schema)

[//]: # (})

[//]: # ()
[//]: # (// Fields of the User.)

[//]: # (func &#40;User&#41; Fields&#40;&#41; []ent.Field {)

[//]: # (	return []ent.Field{)

[//]: # (		field.String&#40;"username"&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (// Indexes of the User.)

[//]: # (func &#40;User&#41; Indexes&#40;&#41; []ent.Index {)

[//]: # (	return []ent.Index{)

[//]: # (		index.Fields&#40;"username"&#41;.Unique&#40;&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (```)

[//]: # ()
[//]: # (As I stated before, we want to use the parsed schema graph to compute the difference between our schema and the)

[//]: # (connected database. Here is an example of a &#40;semi-&#41;persistent MySQL docker container to use if you want to follow along:)

[//]: # ()
[//]: # (```shell)

[//]: # (docker run --rm --name ent-versioned-migrations --detach --env MYSQL_ROOT_PASSWORD=pass --env MYSQL_DATABASE=ent -p 3306:3306 mysql)

[//]: # (```)

[//]: # ()
[//]: # (Once you are done, you can shut down the container and remove all resources with `docker stop ent-versioned-migrations`.)

[//]: # ()
[//]: # (Now, let's create a small function that loads the schema graph and generates the migration files. Create a new Go file)

[//]: # (named `main.go` and copy the following contents:)

[//]: # ()
[//]: # (```go title="main.go")

[//]: # (package main)

[//]: # ()
[//]: # (import &#40;)

[//]: # (	"context")

[//]: # (	"log")

[//]: # (	"os")

[//]: # ()
[//]: # (	"ariga.io/atlas/sql/migrate")

[//]: # (	"entgo.io/ent/dialect/sql")

[//]: # (	"entgo.io/ent/dialect/sql/schema")

[//]: # (	"entgo.io/ent/entc")

[//]: # (	"entgo.io/ent/entc/gen")

[//]: # (	_ "github.com/go-sql-driver/mysql")

[//]: # (&#41;)

[//]: # ()
[//]: # (func main&#40;&#41; {)

[//]: # (	// We need a name for the new migration file.)

[//]: # (	if len&#40;os.Args&#41; < 2 {)

[//]: # (		log.Fatalln&#40;"no name given"&#41;)

[//]: # (	})

[//]: # (	// Create a local migration directory.)

[//]: # (	dir, err := migrate.NewLocalDir&#40;"migrations"&#41;)

[//]: # (	if err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (	// Load the graph.)

[//]: # (	graph, err := entc.LoadGraph&#40;"./ent/schema", &gen.Config{}&#41;)

[//]: # (	if err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (	tbls, err := graph.Tables&#40;&#41;)

[//]: # (	if err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (	// Open connection to the database.)

[//]: # (	drv, err := sql.Open&#40;"mysql", "root:pass@tcp&#40;localhost:3306&#41;/ent"&#41;)

[//]: # (	if err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (	// Inspect the current database state and compare it with the graph.)

[//]: # (	m, err := schema.NewMigrate&#40;drv, schema.WithDir&#40;dir&#41;&#41;)

[//]: # (	if err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (	if err := m.NamedDiff&#40;context.Background&#40;&#41;, os.Args[1], tbls...&#41;; err != nil {)

[//]: # (		log.Fatalln&#40;err&#41;)

[//]: # (	})

[//]: # (})

[//]: # (```)

[//]: # ()
[//]: # (All we have to do now is create the migration directory and execute the above Go file:)

[//]: # ()
[//]: # (```shell)

[//]: # (mkdir migrations)

[//]: # (go run -mod=mod main.go initial)

[//]: # (```)

[//]: # ()
[//]: # (You will now see two new files in the `migrations` directory: `<timestamp>_initial.down.sql`)

[//]: # (and `<timestamp>_initial.up.sql`. The `x.up.sql` files are used to create the database version `x` and `x.down.sql` to)

[//]: # (roll back to the previous version.)

[//]: # ()
[//]: # (```sql title="<timestamp>_initial.up.sql")

[//]: # (CREATE TABLE `users` &#40;`id` bigint NOT NULL AUTO_INCREMENT, `username` varchar&#40;191&#41; NOT NULL, PRIMARY KEY &#40;`id`&#41;, UNIQUE INDEX `user_username` &#40;`username`&#41;&#41; CHARSET utf8mb4 COLLATE utf8mb4_bin;)

[//]: # (```)

[//]: # ()
[//]: # (```sql title="<timestamp>_initial.down.sql")

[//]: # (DROP TABLE `users`;)

[//]: # (```)

[//]: # ()
[//]: # (### Applying Migrations)

[//]: # ()
[//]: # (To apply these migrations on your database, install the `golang-migrate/migrate` tool as described in)

[//]: # (their [README]&#40;https://github.com/golang-migrate/migrate/blob/master/cmd/migrate/README.md&#41;. Then run the following)

[//]: # (command to check if everything went as it should.)

[//]: # ()
[//]: # (```shell)

[//]: # (migrate -help)

[//]: # (```)

[//]: # (```text)

[//]: # (Usage: migrate OPTIONS COMMAND [arg...])

[//]: # (       migrate [ -version | -help ])

[//]: # ()
[//]: # (Options:)

[//]: # (  -source          Location of the migrations &#40;driver://url&#41;)

[//]: # (  -path            Shorthand for -source=file://path)

[//]: # (  -database        Run migrations against this database &#40;driver://url&#41;)

[//]: # (  -prefetch N      Number of migrations to load in advance before executing &#40;default 10&#41;)

[//]: # (  -lock-timeout N  Allow N seconds to acquire database lock &#40;default 15&#41;)

[//]: # (  -verbose         Print verbose logging)

[//]: # (  -version         Print version)

[//]: # (  -help            Print usage)

[//]: # ()
[//]: # (Commands:)

[//]: # (  create [-ext E] [-dir D] [-seq] [-digits N] [-format] NAME)

[//]: # (               Create a set of timestamped up/down migrations titled NAME, in directory D with extension E.)

[//]: # (               Use -seq option to generate sequential up/down migrations with N digits.)

[//]: # (               Use -format option to specify a Go time format string.)

[//]: # (  goto V       Migrate to version V)

[//]: # (  up [N]       Apply all or N up migrations)

[//]: # (  down [N]     Apply all or N down migrations)

[//]: # (  drop         Drop everything inside database)

[//]: # (  force V      Set version V but don't run migration &#40;ignores dirty state&#41;)

[//]: # (  version      Print current migration version)

[//]: # (```)

[//]: # ()
[//]: # (Now we can execute our initial migration and sync the database with our schema:)

[//]: # ()
[//]: # (```shell)

[//]: # (migrate -source 'file://migrations' -database 'mysql://root:pass@tcp&#40;localhost:3306&#41;/ent' up)

[//]: # (```)

[//]: # (```text)

[//]: # (<timestamp>/u initial &#40;349.256951ms&#41;)

[//]: # (```)

[//]: # ()
[//]: # (### Workflow)

[//]: # ()
[//]: # (To demonstrate the usual workflow when using versioned migrations we will both edit our schema graph and generate the)

[//]: # (migration changes for it, and manually create a set of migration files to seed the database with some data. First, we)

[//]: # (will add a Group schema and a many-to-many relation to the existing User schema, next create an admin Group with an)

[//]: # (admin User in it. Go ahead and make the following changes:)

[//]: # ()
[//]: # (```go title="ent/schema/user.go" {22-28})

[//]: # (package schema)

[//]: # ()
[//]: # (import &#40;)

[//]: # (	"entgo.io/ent")

[//]: # (	"entgo.io/ent/schema/edge")

[//]: # (	"entgo.io/ent/schema/field")

[//]: # (	"entgo.io/ent/schema/index")

[//]: # (&#41;)

[//]: # ()
[//]: # (// User holds the schema definition for the User entity.)

[//]: # (type User struct {)

[//]: # (	ent.Schema)

[//]: # (})

[//]: # ()
[//]: # (// Fields of the User.)

[//]: # (func &#40;User&#41; Fields&#40;&#41; []ent.Field {)

[//]: # (	return []ent.Field{)

[//]: # (		field.String&#40;"username"&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (// Edges of the User.)

[//]: # (func &#40;User&#41; Edges&#40;&#41; []ent.Edge {)

[//]: # (	return []ent.Edge{)

[//]: # (		edge.From&#40;"groups", Group.Type&#41;.)

[//]: # (			Ref&#40;"users"&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (// Indexes of the User.)

[//]: # (func &#40;User&#41; Indexes&#40;&#41; []ent.Index {)

[//]: # (	return []ent.Index{)

[//]: # (		index.Fields&#40;"username"&#41;.Unique&#40;&#41;,)

[//]: # (	})

[//]: # (})

[//]: # (```)

[//]: # ()
[//]: # (```go title="ent/schema/group.go")

[//]: # (package schema)

[//]: # ()
[//]: # (import &#40;)

[//]: # (	"entgo.io/ent")

[//]: # (	"entgo.io/ent/schema/edge")

[//]: # (	"entgo.io/ent/schema/field")

[//]: # (	"entgo.io/ent/schema/index")

[//]: # (&#41;)

[//]: # ()
[//]: # (// Group holds the schema definition for the Group entity.)

[//]: # (type Group struct {)

[//]: # (	ent.Schema)

[//]: # (})

[//]: # ()
[//]: # (// Fields of the Group.)

[//]: # (func &#40;Group&#41; Fields&#40;&#41; []ent.Field {)

[//]: # (	return []ent.Field{)

[//]: # (		field.String&#40;"name"&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (// Edges of the Group.)

[//]: # (func &#40;Group&#41; Edges&#40;&#41; []ent.Edge {)

[//]: # (	return []ent.Edge{)

[//]: # (		edge.To&#40;"users", User.Type&#41;,)

[//]: # (	})

[//]: # (})

[//]: # ()
[//]: # (// Indexes of the Group.)

[//]: # (func &#40;Group&#41; Indexes&#40;&#41; []ent.Index {)

[//]: # (	return []ent.Index{)

[//]: # (		index.Fields&#40;"name"&#41;.Unique&#40;&#41;,)

[//]: # (	})

[//]: # (})

[//]: # (```)

[//]: # (Once the schema is updated, create a new set of migration files.)

[//]: # ()
[//]: # (```shell)

[//]: # (go run -mod=mod main.go add_group_schema)

[//]: # (```)

[//]: # ()
[//]: # (Once again there will be two new files in the `migrations` directory: `<timestamp>_add_group_schema.down.sql`)

[//]: # (and `<timestamp>_add_group_schema.up.sql`.)

[//]: # ()
[//]: # (```sql title="<timestamp>_add_group_schema.up.sql")

[//]: # (CREATE TABLE `groups` &#40;`id` bigint NOT NULL AUTO_INCREMENT, `name` varchar&#40;191&#41; NOT NULL, PRIMARY KEY &#40;`id`&#41;, UNIQUE INDEX `group_name` &#40;`name`&#41;&#41; CHARSET utf8mb4 COLLATE utf8mb4_bin;)

[//]: # (CREATE TABLE `group_users` &#40;`group_id` bigint NOT NULL, `user_id` bigint NOT NULL, PRIMARY KEY &#40;`group_id`, `user_id`&#41;, CONSTRAINT `group_users_group_id` FOREIGN KEY &#40;`group_id`&#41; REFERENCES `groups` &#40;`id`&#41; ON DELETE CASCADE, CONSTRAINT `group_users_user_id` FOREIGN KEY &#40;`user_id`&#41; REFERENCES `users` &#40;`id`&#41; ON DELETE CASCADE&#41; CHARSET utf8mb4 COLLATE utf8mb4_bin;)

[//]: # (```)

[//]: # ()
[//]: # (```sql title="<timestamp>_add_group_schema.down.sql")

[//]: # (DROP TABLE `group_users`;)

[//]: # (DROP TABLE `groups`;)

[//]: # (```)

[//]: # ()
[//]: # (Now you can either edit the generated files to add the seed data or create new files for it. I chose the latter:)

[//]: # ()
[//]: # (```shell)

[//]: # (migrate create -format unix -ext sql -dir migrations seed_admin)

[//]: # (```)

[//]: # (```text)

[//]: # ([...]/ent-versioned-migrations/migrations/<timestamp>_seed_admin.up.sql)

[//]: # ([...]/ent-versioned-migrations/migrations/<timestamp>_seed_admin.down.sql)

[//]: # (```)

[//]: # ()
[//]: # (You can now edit those files and add statements to create an admin Group and User.)

[//]: # ()
[//]: # (```sql title="migrations/<timestamp>_seed_admin.up.sql")

[//]: # (INSERT INTO `groups` &#40;`id`, `name`&#41; VALUES &#40;1, 'Admins'&#41;;)

[//]: # (INSERT INTO `users` &#40;`id`, `username`&#41; VALUES &#40;1, 'admin'&#41;;)

[//]: # (INSERT INTO `group_users` &#40;`group_id`, `user_id`&#41; VALUES &#40;1, 1&#41;;)

[//]: # (```)

[//]: # ()
[//]: # (```sql title="migrations/<timestamp>_seed_admin.down.sql")

[//]: # (DELETE FROM `group_users` where `group_id` = 1 and `user_id` = 1;)

[//]: # (DELETE FROM `groups` where id = 1;)

[//]: # (DELETE FROM `users` where id = 1;)

[//]: # (```)

[//]: # ()
[//]: # (Apply the migrations once more, and you are done:)

[//]: # ()
[//]: # (```shell)

[//]: # (migrate -source file://migrations -database 'mysql://root:pass@tcp&#40;localhost:3306&#41;/ent' up)

[//]: # (```)

[//]: # ()
[//]: # (```text)

[//]: # (<timestamp>/u add_group_schema &#40;417.434415ms&#41;)

[//]: # (<timestamp>/u seed_admin &#40;674.189872ms&#41;)

[//]: # (```)

### Wrapping Up

In this post, we demonstrated the general workflow when using Ent Versioned Migrations with `golang-migate/migrate`. We
created a small example schema, generated the migration files for it and learned how to apply them. We now know the
workflow and how to add custom migration files. 

Have questions? Need help with getting started? Feel free to [join our Slack channel](https://entgo.io/docs/slack/).

:::note For more Ent news and updates:

- Subscribe to our [Newsletter](https://www.getrevue.co/profile/ent)
- Follow us on [Twitter](https://twitter.com/entgo_io)
- Join us on #ent on the [Gophers Slack](https://entgo.io/docs/slack)
- Join us on the [Ent Discord Server](https://discord.gg/qZmPgTE6RX)

:::
