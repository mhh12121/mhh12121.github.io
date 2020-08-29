


一些常用的debug，性能测试工具记录

<!--more-->

## deubg

gdb,dlv




## Performance


### Go pprof
有两个包

`runtime/pprof`
`net/pprof`用于网络型，可以实时监控


```go
go tool pprof  xxx.pprof
```
也可以展示http网页

```go
go tool pprof -http localhost:9190 xxx.pprof
```

### Go test

pprof还可以与go test结合

```go
go test -bench . -cpuprofile=cpu.prof

go test -bench . -memprofile=./mem.prof
```