# 一个简单的Web服务器

我们要构建的这个Web服务器包含三个功能：在浏览器输入/时会显示index.html内容，访问服务器的/hello时，会执行服务器上定义的hello函数。访问/form时则调用form函数，显示form.html内容。

安装完golang之后，在用户主目录下会自动创建一个叫go的目录。在Windows，Linux或者Mac上都一样。里面包含bin和src子目录。可以在src目录下创建一个新目录go-server，并在其中创建一个名为static的子目录和一个名为main.go的文件。static子目录下创建两个文件，分别命名为index.html和form.html。然后用你喜欢的编辑器打开main.go，开始编辑。

## 编写html文件
可以在index.html和form.html文件中编写最简单的html信息。

## 编写main.go代码
go代码需要一个名为main的package。需要在文件的第一行写下package的定义，如下：
```
package main
```
接着引入这段代码依赖的一些package，首先是格式化打印，其次是日志记录，再一个是web服务需要的http包，如下：

```
package main

import (
    "fmt"
    "log"
    "net/http"
)
```
package在定义的时候不需要用双引号括住，但是在import的时候需要。尽管import的时候包括/分隔符，但在使用的时候可以只使用最后一个分隔符后的标识来表示。这可能会引起名称冲突，可以在import时给引入的包指定一个别名来解决冲突。

go代码会从一个名为main的函数开始执行，因此我们需要定义它：
```
func main() {
    server := http.FileServer(http.Dir("./static"))  // 从static目录查找静态文件
    http.Handle("/", server)  // 处理到/的请求
    http.HandleFunc("/form", formHandler)  // 处理到/form的请求
    http.HandleFunc("/hello", helloHandler)  // 处理到/hello的请求
    
    fmt.Printf("Starting server at port 8080\n")
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}
```
这里我们调用了两个请求处理函数，接下来就开始实现它们：
```
func helloHandler(w http.ResponseWriter, r *http.Request) {  // 这里的*代表是指针
    if r.URL.Path != "/hello" {
        http.Error(w, "404 not found", http.StatusNotFound)
        return
    }
    if r.Method != "GET" {
        http.Error(w, "method is not suppported", http.StatusNotFound)
        return
    }
    fmt.Fprintf(w, "Hello")
}

func formHandler(w http.ResponseWriter, r *http.Request) {
    if err := r.ParseForm(); err != nil {
        fmt.Fprintf(w, "ParseForm error: %v", err)   // 输出变量err的信息
        return
    }
    fmt.Fprintf(w, "POST request successful")
    name := r.FormValue("name")
    address := r.FormValue("address")
    fmt.Fprintf(w, "Name = %s\n", name)
    fmt.Fprintf(w, "Address = %s\n", address)
}
```

# 运行
回到命令行，执行以下命令：
```
go run main.go
```
再打开浏览器，依次访问/,/hello,/form.html。页面可以正常打开。

