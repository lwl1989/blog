CreateTime:2019-09-30 14:13:07.0

### 资料

[官方文档](https://docs.mongodb.com/ecosystem/drivers/go/ "官方文档")

[官方驱动github](https://github.com/mongodb/mongo-go-driver#installation "官方驱动github")

之前一直使用mgo，但是已经不维护了，(mgo：是MongoDB的Go语言驱动，它用基于Go语法的简单API实现了丰富的特性，并经过良好测试。使用起来很顺手，文档足够)，因此转到mongodb官方driver。mongo-go-driver：官方的驱动，设计的很底层，因此扩展性比较好，但是使用复杂度有一定提高，并且支持事务。


### 入门

##### 连接

```
import (
    "go.mongodb.org/mongo-driver/mongo"
    "go.mongodb.org/mongo-driver/mongo/options"
)

client, err := mongo.NewClient(options.Client().ApplyURI("mongodb://localhost:27017"))
```

go的连接驱动一般都支持连接池，果不其然，mongodb也是，我们查看源码发现可以设置

options对象里包含连接池大小配置，可以方便开发根据环境进行配置。
```
MaxPoolSize            *uint64
MinPoolSize            *uint64
```

##### 查询

查询相对复杂，会返回一个迭代器，这时候可以封装一个共用方法进行处理
```
ctx, _ = context.WithTimeout(context.Background(), 30*time.Second)
cur, err := collection.Find(ctx, bson.D{})
if err != nil { log.Fatal(err) }
defer cur.Close(ctx)
for cur.Next(ctx) {
   var result bson.M
   err := cur.Decode(&result)
   if err != nil { log.Fatal(err) }
   // do something with result....
}
if err := cur.Err(); err != nil {
  log.Fatal(err)
}
```

##### 插入

没啥难度，和其他db差不多
```
collection := client.Database("testing").Collection("pi_test")
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
 _, err := collection.InsertOne(ctx, bson.M{"name": "pi", "value": math.Round(10000)})
```

##### 更新

```

    c := Connect(db, collection)
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    opts := options.Update().SetUpsert(true)
    _, err := c.UpdateOne(ctx, query, update,opts)
```
mongo-go-driver需要设置opts := options.Update().SetUpsert(true),如果是想要进行upsert


##### 删除

```
    c := Connect(db, collection)
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    _, err :=collection.DeleteOne(ctx, bson.M{"name": "pi", "value": math.Round(10000)})
```


##### 事务的支持

MongoDB 4.0 引入的事务功能，支持多文档ACID特性，这实在是一件棒棒哒的事情(nosql事务哟)

只有官方驱动才支持事务，mgo已经年久失修了，别用了，放弃吧！

因为多处使用，所以封装了一个方法(此处封装来自于[mgo vs mongodb driver](https://segmentfault.com/a/1190000020362675 "mgo vs mongodb driver"))；
在这个方法中需要实现的方法是Exec的operator
```
type DBTransaction struct {
    Commit func(mongo.SessionContext) error
    Run    func(mongo.SessionContext, func(mongo.SessionContext, DBTransaction) error) error
    Logger *logging.Logger
}

func NewDBTransaction(logger *logging.Logger) *DBTransaction {
    var dbTransaction = &DBTransaction{}
    dbTransaction.SetLogger(logger)
    dbTransaction.SetRun()
    dbTransaction.SetCommit()
    return dbTransaction
}

func (d *DBTransaction) SetCommit() {
    d.Commit = func(sctx mongo.SessionContext) error {
        err := sctx.CommitTransaction(sctx)
        switch e := err.(type) {
        case nil:
            d.Logger.Info("Transaction committed.")
            return nil
        default:
            d.Logger.Error("Error during commit...")
            return e
        }
    }
}

func (d *DBTransaction) SetRun() {
    d.Run = func(sctx mongo.SessionContext, txnFn func(mongo.SessionContext, DBTransaction) error) error {
        err := txnFn(sctx, *d) // Performs transaction.
        if err == nil {
            return nil
        }
        d.Logger.Error("Transaction aborted. Caught exception during transaction.",
            zap.String("error", err.Error()))

        return err
    }
}

func (d *DBTransaction) SetLogger(logger *logging.Logger) {
    d.Logger = logger
}

func (d *DBTransaction) Exec(mongoClient *mongo.Client, operator func(mongo.SessionContext, DBTransaction) error) error {
    ctx, cancel := context.WithTimeout(context.Background(), 20*time.Minute)
    defer cancel()

    return mongoClient.UseSessionWithOptions(
        ctx, options.Session().SetDefaultReadPreference(readpref.Primary()),
        func(sctx mongo.SessionContext) error {
            return d.Run(sctx, operator)
        },
    )
}

//具体调用
func SyncBlockData(node models.DBNode) error {
    dbTransaction := db_session_service.NewDBTransaction(Logger)

    // Updates two collections in a transaction.
    updateEmployeeInfo := func(sctx mongo.SessionContext, d db_session_service.DBTransaction) error {
        err := sctx.StartTransaction(options.Transaction().
            SetReadConcern(readconcern.Snapshot()).
            SetWriteConcern(writeconcern.New(writeconcern.WMajority())),
        )
        if err != nil {
            return err
        }
        err = models.InsertNodeWithSession(sctx, node)
        if err != nil {
            _ = sctx.AbortTransaction(sctx)
            d.Logger.Info("caught exception during transaction, aborting.")
            return err
        }

        return d.Commit(sctx)
    }

    return dbTransaction.Exec(models.DB.Mongo, updateEmployeeInfo)
}
```


### 总结

就是一篇记录常用api的记录，顺便记录下遇到的坑。

ps:官方测试总是吧 context的 退出给关闭了。
![](https://oscimg.oschina.net/oscnet/0b266a801e168441e37f0f8dc38c0fed1b6.jpg)

实际上应该合理处理
```
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel() //一定记得加defer
```

