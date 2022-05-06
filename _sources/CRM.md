# CRM系统

## 项目准备
创建项目目录，并初始化模块。
```shell
mkdir go-crm && cd go-crm
go mod init crm
mkdir db lead
touch db/db.go lead/lead.go main.go
```

## 代码实现
### main

注册路由。

```go 
package main

import (
    "fmt"
    "github.com/gofiber/fiber"
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/sqlite"
    "crm/db"
    "crm/lead"
)

func setupRoutes(app *fiber.App){
    app.Get("/api/v1/lead", lead.GetLeads)
    app.Get("/api/v1/lead/:id", lead.GetLead)
    app.Post("/api/v1/lead", lead.NewLead)
    app.Delete("/api/v1/lead/:id", lead.DeleteLead)
}

func initDB(){
    var err error
    db.Session, err = gorm.Open("sqlite3", "leads.db")
    
    if err != nil {
        panic("Failed to open database!")
    }
    fmt.Println("Database connected.")
    db.Session.AutoMigrate(&lead.Lead{})
    fmt.Println("Database migrated.")
}

func main(){
    app := fiber.New()
    initDB()
    setupRoutes(app)
    app.Listen(8080)  // 这里有很多种写法，如：- app.Listen("8080") - app.Listen(":8080") - app.Listen("127.0.0.1:8080")
    defer db.Session.Close()
}
```

### DB

```go
package db

import (
    "github.com/jinzhu/gorm"
//    _ "github.com/jinzhu/gorm/dialects/sqlite" // 这一行可以不要
)

var (
    Session *gorm.DB
)
```

### Lead
```go
package lead

import (
    "github.com/jinzhu/gorm"
    "github.com/gofiber/fiber"
    "crm/db"
)

type Lead struct{
    gorm.Model
    Name string      `json:"name"`
    Company string   `json:"company"`
    Email string     `json:"email"`
    Phone int        `json:"phone"`
}

func GetLeads(c *fiber.Ctx){
    var leads []Lead  // slice of Lead
    db.Session.Find(&leads)
    c.JSON(leads)
}

func GetLead(c *fiber.Ctx){
    id := c.Params("id")
    var lead Lead
    db.Session.Find(&lead, id)
    c.JSON(lead)   // 这里直接用的对象不是么
}

func NewLead(c *fiber.Ctx){
    lead := new(Lead) // 分配内存，后面可以不用&，如果用Lead{}初始化，后面就需要用&
    
    if err := c.BodyParser(lead); err != nil {
        c.Status(503).Send(err)
        return
    }
    db.Session.Create(lead)  // 这里还需要用&吗？是不是需要返回创建的ID？*答案是不需要用&了。*
    c.JSON(lead) // 居然也可以直接用指针？是的，确实可以。
}

func DeleteLead(c *fiber.Ctx){
    id := c.Params("id")
    var lead Lead
    db.Session.First(&lead, id)
    if lead.Name == "" {
        c.Status(500).Send("Not found.")
        return
    }
    db.Session.Delete(&lead)
    c.Send("Delete successful.")
}
```

## 编译运行

```shell 
go mod tidy  # 安装各种依赖
go build     # 会生成一个与go mod init中定义的名称（可以在go.mod）相同的可执行文件，这里是crm
./crm

# 或者直接运行
go run main.go
```

## 测试
用postman调用各个接口。都正常。

## 技巧
可以用`go mod tidy`检查是否正确安装了需要的包，并自动下载需要的包，把不需要的清理掉。

## 排错
1. package的名字必须与文件夹的名称相同。
2. 如果8080端口意外被占用，程序会退出，并没有别的提示。可以加上log模块，并用如下代码替换对应的行。
> log.Fatal(app.Listen(8080))
