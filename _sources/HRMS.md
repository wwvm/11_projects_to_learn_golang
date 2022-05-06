# HRMS
用mongo数据库做存储。还是使用fiber框架，这次用v2版本。但是代码放在main.go

## 项目准备
```go
go mod init hrms
```

## 代码

```go
package main

import (
    "context"
    "log"
    "time"
    "github.com/gofiber/fiber/v2"   // 这里最后一个标记是v2，为何后面可以引用fiber呢？
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
    "go.mongodb.org/mongo-driver/bson"
    "go.mongodb.org/mongo-driver/bson/primitive"
)

// 定义数据结构
// mongo数据库以bson存储
type Employee struct {
    ID      string   `json:"id,omitempty" bson:"_id,omitempty"` // :及,后面不能有空格，否则数据库中会多字段或者无法生成_id，并且返回的数据中对应不上_id
    Name    string   `json: "name"` // 但这里的:后面可以有空格，不影响
    Age     int      `json: "age"`
    Salary  float64  `json: "salary"`
}

type MongoInstance struct {
    Client *mongo.Client
    db     *mongo.Database  // 这里db用全小写是可以的
}

// 定义变量
var mg MongoInstance

// 定义常量
const dbName = "hrms"
const dbURI = "mongodb://localhost:27017/" + dbName  // 这里用户名口令可以放在localhost前面，但是dbName前直接跟端口号？

// 注册路由
func setupRoutes(app *fiber.App){
    app.Get("/employee", func(c *fiber.Ctx) error {
        // 构建查询
        query := bson.D{{}}
        cursor, err := mg.db.Collection("employees").Find(c.Context(), query)
        if err != nil {
            return c.Status(500).SendString(err.Error())  // 可以直接返回
        }
        // 创建并初始化列表
        var employees []Employee = make([]Employee, 0)
        if err := cursor.All(c.Context(), &employees); err != nil {
            return c.Status(500).SendString(err.Error())
        }
        return c.JSON(employees) //
    })
    
    app.Post("/employee", func(c *fiber.Ctx) error {
        employees := mg.db.Collection("employees")
        
        employee := new(Employee)
        if err := c.BodyParser(employee); err != nil {
            return c.Status(400).SendString(err.Error())
        }
        
        employee.ID = "" // 防止误传了ID
        res, err := employees.InsertOne(c.Context(), employee)
        
        if err != nil {
            return c.Status(500).SendString(err.Error())
        }
        
        filter := bson.D{{Key: "_id", Value: res.InsertedID}} // 过滤最近插入的记录
        record := employees.FindOne(c.Context(), filter)
        
        record.Decode(employee)
        return c.Status(201).JSON(employee)
    })
    
    app.Put("/employee/:id", func(c *fiber.Ctx) error {
        id, err := primitive.ObjectIDFromHex(c.Params("id"))
        
        if err != nil {
            return c.SendStatus(400)
        }
        
        employee := new(Employee)
        if err := c.BodyParser(employee); err != nil {
            return c.Status(400).SendString(err.Error())
        }
        
        filter := bson.D{{Key: "_id", Value: id}}
        // 这里有个问题，如果传入的信息不包括age或者其他任何一个，则对应的原有数据被全部更新成空的。
        update := bson.D{{
            Key: "$set",
            Value: bson.D{
                {Key: "name", Value: employee.Name},
                {Key: "age", Value: employee.Age},
                {Key: "salary", Value: employee.Salary},  // 为啥这里最后还要一个,
            }, // 这里也要
        }}
        err = mg.db.Collection("employees").FindOneAndUpdate(c.Context(), filter, update).Err()
        
        if err != nil {
            if err == mongo.ErrNoDocuments {
                return c.SendStatus(400)
            }
            return c.SendStatus(500)
        }
        
        employee.ID = c.Params("id")
        return c.Status(200).JSON(employee)
    })
    
    app.Delete("/employee/:id", func(c *fiber.Ctx) error {
        id, err := primitive.ObjectIDFromHex(c.Params("id"))
        if err != nil {
            return c.SendStatus(400)
        }
        filter := bson.D{{Key: "_id", Value: id}}
        res, err := mg.db.Collection("employees").DeleteOne(c.Context(), filter)
        
        if err != nil {
            return c.SendStatus(500)
        }
        
        if res.DeletedCount < 1 {
            return c.SendStatus(404)
        }
        return c.Status(200).JSON("Record deleted!")
    })
}

// 数据库连接
func connect() error {
    client, err := mongo.NewClient(options.Client().ApplyURI(dbURI))
    if err != nil {
        return err
    }
    
    ctx, cancel := context.WithTimeout(context.Background(), 30 * time.Second)  // mongo里部分函数是阻塞的，所以需要指定超时
    defer cancel()  // 这应该是超时后要执行的
    
    err = client.Connect(ctx)
    if err != nil {
        return err
    }
    
    mg = MongoInstance{
        Client: client,
        db: client.Database(dbName), // 这里也要一个,
    }
    
    return nil
}

// 函数入口
func main(){
    if err := connect(); err != nil {
        log.Fatal(err)   // 需要退出程序吗？
    }
    app := fiber.New()   // 为何这里可以直接引用fiber？
    
    setupRoutes(app)
    
    log.Fatal(app.Listen(":8080"))  // v2版本不能直接用端口号了
}
```

## 数据库准备
```shell
docker run --rm -d -p 27017:27017 mongo
```

## 编译运行

```go 
go mod tidy
go build
./hrms
```

## 问题解决
1. 在json和bson映射的时候，id字段需要严格不多空格。否则会有莫名其妙的问题。
2. mongo数据库只有在写入数据的时候才会创建数据库和collections，读取的时候如果不存在并不会创建。
3. postman测试发现，返回的字段，除id为全小写外，其他字段名都是首字母大写。原因是在json映射的时候，定义成这样：`json: "name"`。冒号后有空格导致。

## Mongo数据库操作

查看所有数据库：show dbs
查看当前数据库：db
切换数据库：use <dbname> # 如果dbname不存在，则会立即创建一个
查看当前库中集合：show collections
查看集合col下所有数据：db.col.find() 或者 db.col.find().pretty()

删除当前数据库：db.dropDatabase()
当前默认数据库：test
删除集合col: db.col.drop()
