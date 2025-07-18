bpf2go的作用是将bpf的c代码转换为go代码吗？

SEC(".maps")表示将数据或结构体放到ELF的.maps段中。



**交叉编译**

你可能已经注意到bpf2go生成了两种类型的文件：

*_bpfel.o 和 *_bpfel.go 适用于小端架构，例如 amd64、arm64、riscv64 和 loong64

*_bpfeb.o 和 *_bpfeb.go 适用于 s390(x)、mips 和 sparc 等大端架构



两组.go文件均包含`//go:embed`指令，该指令会在编译阶段将对应的.o文件内容直接内嵌至字节切片中。最终生成的Go应用程序二进制文件可独立部署至目标机器，无需附带任何.o文件。若需进一步减少运行时依赖，只需在`go build`时添加`CGO_ENABLED=0`参数，即可消除对libc的依赖（前提是其他依赖项均无需cgo支持）。



由于生成的eBPF对象与Go脚手架代码均兼容大端序（big-endian）和小端序（little-endian）架构，您只需在编译时指定正确的GOARCH参数值，即可轻松完成Go应用程序的跨平台编译。



把所有这些放在一起，为运行 64 位 Linux 发行版的 Raspberry Pi 构建一个由 eBPF 驱动的 Go 应用程序：

```
CGO_ENABLED=0 GOARCH=arm64 go build
```





参考链接：

https://cloud.tencent.com/developer/article/2472587