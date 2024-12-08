# 什么是 go:embed

`//go:embed` 在 Go 1.16 版本中被加入，这也是我接触 Go 语言的第一个版本。

`//go:embed` 是一个 **编译器指令** ，能够在程序编译时期在 Go 的二进制文件中嵌入 **任意文件和目录** （除了少数 Go 官方限制不允许嵌入的指定类型文件或目录，后文会讲）。

`//go:embed` 用法非常简单，示例如下：

```
import "embed"

//将文件嵌入到 `string` 中，适合嵌入单个文件（如配置数据、模板文件或一段文本）
//go:embed hello.txt
var content string

//将文件嵌入到 `[]byte` 中，适合嵌入单个文件（如二进制文件：图片、字体或其他非文本数据）
//go:embed hello.txt
var contentBytes []byte

//将文件嵌入到 `embed.FS` 中，适合嵌入多个文件或整个目录（`embed.FS` 是一个只读的虚拟文件系统）
//go:embed hello.txt
var fileFS embed.FS
var data, _ = fileFS.ReadFile("hello.txt")
```

我们**有且仅有 3 种方式**可以将一个文件内容嵌入到 Go 变量中。

`//go:embed` 指令语法格式为：`//go:embed patterns`，其中 `patterns` 可以是文件名、目录名或 `path.Match` 所支持的路径通配符。

值得注意的是：`//go:embed` 指令仅接受相对于包含 Go 源文件的目录的路径，即当前程序源码所在目录，不能直接嵌入父目录文件内容（后文会讲解解决方案）。

并且，`//go:embed` 指令要紧挨着写在被嵌入文件的变量上面，类似注释，`//go:embed` 是固定写法，字符中间不能含有任何空格。比如 `// go:embed` 这种写法是不能被解析的。

此外，`//go:embed` 指令需要配合 `embed` 包一起使用。

# 快速开始

下面我们来通过一个示例程序，演示下 `//go:embed` 的使用。

准备如下项目目录结构：

```
$ tree -a getting-started
getting-started
├── file
│   ├── hello1.txt
│   ├── hello2.txt
│   └── sub
│       └── sub.txt
├── go.mod
├── hello.txt
└── main.go

3 directories, 6 files
```

在 `main.go` 中编写示例代码如下：

```
package main

import (
    "embed"
    "fmt"
    "io"
    "io/fs"
)

//与源文件同级目录

//go:embed hello.txt
var content string


//与源文件不同级目录

//go:embed file
var fileFS embed.FS

//go:embed file/hello1.txt
//go:embed file/hello2.txt
var helloFS embed.FS

func main() {
    fmt.Printf("hello.txt content: %s\n", content)

    // embed.FS 提供了 ReadFile 功能，可以直接读取文件内容，文件路径需要指明父目录 `file`
    hello1Bytes, _ := fileFS.ReadFile("file/hello1.txt")
    fmt.Printf("file/hello1.txt content: %s\n", hello1Bytes)

    // embed.FS 提供了 ReadDir 功能，通过它可以遍历一个目录下的所有信息
    dir, _ := fs.ReadDir(fileFS, "file")
    for _, entry := range dir {
        info, _ := entry.Info()
        fmt.Printf("%+v\n", struct {
            Name  string
            IsDir bool
            Info  struct {
                Name string
                Size int64
                Mode fs.FileMode
            }
        }{
            Name:  entry.Name(),
            IsDir: entry.IsDir(),
            Info: struct {
                Name string
                Size int64
                Mode fs.FileMode
            }{Name: info.Name(), Size: info.Size(), Mode: info.Mode()},
        })
    }

    // embed.FS 实现了 io/fs.FS 接口，可以返回它的子文件夹作为新的 io/fs.FS 文件系统
    subFS, _ := fs.Sub(helloFS, "file")
    hello2F, _ := subFS.Open("hello2.txt")
    hello2Bytes, _ := io.ReadAll(hello2F)
    fmt.Printf("file/hello2.txt content: %s\n", hello2Bytes)
}
```

示例程序中，将 `hello.txt` 分别嵌入到 `content string`、`contentBytes []byte` 两个变量。

我们可以通过 `//go:embed 目录名` 的方式，将 `file` 目录嵌入到 `fileFS embed.FS` 文件系统。

对于 `embed.FS` 文件系统，我们可以连续写上多个 `//go:embed` 指令，来嵌入多个文件到 `helloFS embed.FS`。

并且，对于 `helloFS embed.FS` 这段代码：

```
//go:embed file/hello1.txt
//go:embed file/hello2.txt
var helloFS embed.FS
```

还有另一种写法：

```
//go:embed file/hello1.txt file/hello2.txt
var helloFS embed.FS
```

二者等价。

在 `main` 函数中，首先对 `content string` 和 `contentBytes []byte` 的字符串格式内容进行了打印。

接着，使用 `embed.FS` 提供的 `ReadFile` 方法读取 `file/hello1.txt` 文件内容（注意：**文件路径必须要指明父目录 `file`，否则会找不到 `hello1.txt` 文件**）并打印。

此外，`embed.FS` 还提供了 `fs.ReadDir` **方法可以读取指定目录下所有文件信息**。

最后，由于 `embed.FS` 实现了 `io/fs.FS` 接口，我们可以**使用 `fs.Sub` 获取指定目录的子文件系统，然后就可以用 **`subFS.Open("hello2.txt")` **方式直接读取** `hello2.txt` **内容（无需再指明父目录）并打印了**。
