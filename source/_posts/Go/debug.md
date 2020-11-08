


一些常用的debug，性能测试工具记录

<!--more-->

## deubg

### build

一般我们想生成汇编代码，无非就是:

```shell
go build -gcflags -S main.go
```
或:

```shell
go tools compile -S main.go
```


但是通过以下命令，可以生成更加详细的汇编优化过程:
```shell
GOSSAFUNC=main go build main.go
```

### gdb

列举用到的文件,其中entry_files的地址就是函数开始运行的地址
```shell
info files
```
可以直接`b address`，然后运行`c`

### dlv

比较契合golang，

`dlv debug`： 在当前目录寻找main函数，并开始调试

具体进入dlv后输入help查看命令:

常用:
`print(p)`:打印local
`b xxx.go:123` break某一行
`c` :continue
`n`:下一行
`step`:下一步
`restart`:重启






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


