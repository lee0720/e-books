---
html:
  embed_local_images: false
  embed_svg: true
  offline: false
  toc: true
print_background: false
export_on_save:
  html: true
  puppeteer: true
---
### 工具链

了解内置和扩展工具使用。

  * 开发环境安装及设置
  * 编译器（compile）和链接器（link）参数
  * 配套工具使用（get、vet）
  * 单元测试（testing）和性能工具（benchmark）
  * 调试器（dlv）及扩展工具（linter）使用

### 安装

    
    
    $ GOROOT_BOOTSTRAP=/usr/local/go1.4 ./make.bash
    

  * 使用 apt、apk 安装（版本老旧）
  * 下载预编译包（go1.11.linux-386.tar.gz）
  * 下载源码（go1.11.src.tar.gz）自举（bootstrap）编译
  * 将 go/bin 目录添加到搜索路径（PATH）

对于初学者建议使用 apt、apk 直接安装尝试，避免前期门槛。

#### **多版本安装**

    
    
    $ go get golang.org/x/build/version/go1.10
    $ go1.10 download
    $ go1.10 version
    

官方专门写了一个下载和执行包装器。可使用多版本安装模式尝试最新的 tip 版本。

提示：Go 1.10 是一个下载和间接调用 Go 命令的工具。

### 环境变量

    
    
    $ go env
    

  * GOARCH、GOOS：架构和操作系统
  * GOCACHE：编译缓存目录
  * GOPATH：工作空间目录列表
  * GOROOT：安装目录（工具链、标准库、运行时）
  * CGO_ENABLED：与 C 混合编程

### 编译

    
    
    $ go build
    

包括编译和链接过程，支持多种语言代码（.go、.s、.c、.cpp）

  * -o：目标文件名
  * -a：强制重新编译，包括标准库
  * -x：显示编译过程
  * -gcflags：编译器参数
  * -ldflags：链接器参数

#### **编译器**

    
    
    $ go tool compile -h
    

  * -N：禁用代码优化
  * -l：禁用内联
  * -u：禁用 unsafe 代码
  * -S：输出汇编
  * -m：输出优化信息（查看内存逃逸等）

完整的编译包括编译和链接两部分。编译会针对单个源码文件做出一些调整，生成符号表和二进制代码。

举个例子，一些自动化编译工具编译的可执行程序包含一些特殊的信息，比如版本号。编译是通过符号替换来实现。

    
    
    // main.go
    var version = "0.9"
    
    func test() {
        if runtime.GOOS == "darwin" {
            println("ping!")
        }
    }
    
    func main() {
        println("version:", version)
        test()
        info(platform())
    }
    
    
    
    $ go build -ldflags "-X main.version=1.0" #链接参数
    $ ./test
    

在不修改源码的情况下，通过命令行编译参数动态调整一些内容。

#### **交叉编译（cross compile）**

一份源码会对不同平台生成不同二进制包，编译的时候是否能实现增量编译。

编译缓存二进制的静态库信息，静态库和链接器可以直接链接。交叉编译会编译所有的标准库，我们可以考虑把这些需要的环境提前编译。

    
    
    $ GOOS=linux go install std
    $ GOOS=linux go build -o test
    

交叉编译不支持 CGO，CGO 要引用 SO。

#### **条件编译（Go、C）**

最常见的还有条件编译，C 语言通过一些宏指令来实现条件编译。

    
    
    #include <stdio.h>
    
    #define VER "0.9"
    
    int main()
    {
        printf("version: %s, ", VER);
    
        #ifdef __RELEASE__
        printf("release.\n");
        #else
        printf("debug.\n");
        #endif
    
        return 0;
    }
    
    
    
    $ gcc -g -o test #debug
    $ gcc -g -O3 -D__RELEASE__ -o test #release
    

定义 __RELEASE__ 条件，GCC 通过 `-D` 参数定义条件。

Go 语言有几种方式。第 1 种方式对同一个算法提供两个文件，文件名标注属于哪种条件。

我们提供同一个函数的两个版本。第一个版本是 Linux、另一个版本是苹果。

    
    
    // test_darwin.go
    func platform() string {
        return "hello, osx!"
    }
    
    
    
    // test_linux.go
    func platform() string {
        return "hello, linux!"
    }
    
    
    
    $ GOOS=darwin go build -x -o test #OSX
    $ GOOS=linux go build -x -o test #Linux
    

上面告诉编译器，文件里的代码只能在 Darwin 和 Linux 平台下执行。

比如实现一个算法，函数提供两个版本，一个版本是 386，一个版本是
amd64。编译可以选择两个版本。相对稳定的部分放在一个文件，需要动态的部分和平台有关系的算法放在不同的文件，这也是一种隔离。

另外一种条件方式是在文件里提供编译条件。

    
    
    // debug.go
    
    // +build !release
    func info(s string) {
        log.Println("debug:", s)
    }
    
    
    
    // release.go
    
    // +build release
    func info(s string) {
        log.Println("release:", s)
    }
    
    
    
    $ go build -x -o test
    $ go build -x -tags "release" -o test
    $ ./test
    

实现函数不能按照平台来划分的，因为不管哪个平台都需要，区分的不是平台而是调试版本和发行版本。通过编译伪指令实现。这不属于语言的功能，而是编译功能。通过编译伪指令方式在不同的状态进行切换。

最好的方式是通过把调试版本和正式版本的代码用不同的文件的隔离，编译选择装配的方式实现定制目标版本。条件编译对代码的隔离和冻结非常重要。

#### **链接器**

    
    
    $ go tool link -h
    

  * -s：禁止生成符号表
  * -w：禁止生成调试信息（DWARF）

### 预处理（go generate）

预处理操作就是 `//go:generate go build -o test -tags "release"`
专用的指令，编译器如果判断有这条指令的时候，编译器认为后面是一个标准的命令行命令，在单个文件里或者多个文件里写很多这样指令。用于完成一些相应的辅助工作。

    
    
    $ go generate #执行命令
    $ go generate -x #执行命令执行
    $ go generate -n #扫描命令不执行
    

跟语言无关的自动化的预处理工具有很多，更习惯用 Makefile 工具，因为这些东西写到源码文件是一种污染。

### 反汇编

    
    
    $ go tool objdump -S -s "main\.main" test
    
    
    
    TEXT main.main(SB)
     0x104e390 c3 RET
    

  * -S：输出源码
  * -s：仅针对特定目标

### 扩展包下载

    
    
    $ go get -x github.com/shirou/gopsutil/mem
    

将源码下载到第一工作空间并安装。

  * -d：仅下载不安装
  * -u：更新包括依赖包
  * -t：下载测试所需依赖包
  * -x：显示下载过程

### 垃圾清理

    
    
    $ go clean -i -cache
    

清理编译生成的临时文件等。

  * -i：清理安装文件
  * -cache：清理编译缓存文件
  * -testcache：清理测试缓存文件
  * -x：显示清理过程
  * -n：仅显示命令，不执行

### 扩展：代码检查（linter、deps）、格式化（fmt、tab/space）

最好的方式用多个工具构建一套完整的自动化工具，比如写源码对源码的检查，然后自动化构建，然后自动化部署。所有的语言都有代码检查。最常见的检查工具有：

> <https://github.com/golangci/golangci-lint>

对代码的一些本身的检查工具，它整合了很多检查工具，把它打包成一个来进行检查。

> <https://github.com/divan/depscheck>

我们开发软件都会依赖很多第三方的库，这个根据会扫描当前程序依赖了多少第三方库，然后进行统计。给一些代码的一些建议。

#### **代码检查**

    
    
    func main() {
        var m sync.Mutex
        m2 := m
        _ = m2
    }
    
    
    
    $ go vet
    ./main.go:7: assignment copies lock value to m2: sync.Mutex
    ./main.go:8: assignment copies lock value to _: sync.Mutex
    

检查编译器无法察觉的一些问题。

#### **代码检查增强版**

    
    
    func main() {
        f, _ := os.Open("./main.go")
        defer f.Close()
    }
    
    
    
    $ go get -u github.com/alecthomas/gometalinter
    $ gometalinter --install
    $ gometalinter
    main.go:7:15:warning: error return value not checked (defer f.Close()) (errcheck)
    main.go:6::warning: Errors unhandled.,LOW,HIGH (gosec)
    

### 调试器

    
    
    $ go get -u github.com/derekparker/delve/cmd/dlv
    

官方支持的调试器，支持服务器模式：

  * 调试模式（dlv debug）以 `"-N -l"` 方式编译
  * 已编译程序使用 `dlv exec`
  * 已运行程序使用 `dlv attach`

> **感谢各位的光临哟！！**
> **获取更多资源:掘金小册,gitchat专栏,极客时间等资源；**
> **请到闲鱼店：583128058yanghon**
> **啊呜呜~~~**