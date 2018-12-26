title: Go get through socks5 proxy
date: 2018-12-26 10:07:49
tags: [Go]
---
### set git through socks5 proxy:

```go
git config --global http.proxy socks5://127.0.0.1:1080
```

### go get via socks5 proxy:

```go
http_proxy=socks5://127.0.0.1:1080 go get github.com/mattn/go-sqlite3
```

### Recover git global config:

```go
git config --global --unset http.proxy
```