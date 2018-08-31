title: Postgres dump table into file
date: 2018-04-19 12:36:39
tags: [Postgres]
toc: true
---

- Using [go-pg](https://github.com/go-pg/pg). To avoid out of memory issue, instead of writing the result to buffer, the result should be written directly to a file. The snippets will be:

<!-- more -->
```go
//open database connection first
db := pg.Connect(&pg.Options{
    User:     "username",
    Password: "password",
    Database: "database",
    Addr:     "192.168.1.2:5432",   //database address
})
defer db.Close()

//open output file
out, err := os.Create("/tmp/bar.json")
if err != nil {
    panic(err)
}
defer out.Close()

//execute opy
copy := `COPY (SELECT row_to_json(foo) FROM (SELECT * FROM bar) foo ) TO STDOUT`
_, err = db.CopyTo(out, copy)
if err != nil {
    panic(err)
}
```

- Using [psql](https://www.postgresql.org/docs/current/static/app-psql.html) and [exec](https://golang.org/pkg/os/exec/) package. Basically, this approach execute a psql command in the form of: psql -c query1 -c query2 ... args. Please note, this approach requires that psql is already installed. The snippets:

```go
queries := []string{
    `SET client_encoding='UTF8'`,
    `\COPY (SELECT row_to_json(foo) FROM (SELECT * FROM bar) foo ) TO '/tmp/bar.json'`,
}
dsn := "postgresql://username:password@192.168.1.2/database"

//construct arguments
args := []string{}
for _, q := range queries {
    args = append(args, "-c", q)
}
args = append(args, dsn)

//Execute psql command
cmd := exec.Command("psql", args...)
stdoutStderr, err := cmd.CombinedOutput()
if err != nil {
    panic(err)
}
fmt.Printf("%s\n", stdoutStderr)
```

