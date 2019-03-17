---
title: MongoDB基本使用
date: 2017-11-20 10:06:10
tags:
- MongoDB基本使用
- MongoDB
categories:
- 数据库
---

# 基本使用

## 基本概念

* **数据库** MongoDB的一个实例可以拥有一个或多个相互独立的数据库，每个数据库都有自己的集合（表）。
* **集合 **可以看作是拥有动态模式的表
* **方档 **是MongoDB中基本的数据单元，类似于RDB的行。文档是键值对的一个有序集合。在JS中，文档被表示成对象。
* **_id **每个文档都有个特殊的 <code>_id</code>，在文档所属集合中是唯一的，默认由mongodb自己维护，当然你也可以自己指定。这和关系型数据库中的自动增长主键是类似的。
* **JavaScript shell** MongoDB自带了一个功能强大的JavaScript Shell，可以用于管理或操作MongoDB。

### MongoDB和RDB的概念对比

1. 都有数据库的概念
1. 集合 --> RDB的表
1. 文档 --> RDB表中的一条记录
1. 文档对象里面的 key --> RDB表中的字段
1. 文档对象里面的 value --> RDB表中字段的值
1. MongoDB中没有外键的概念
<!-- more -->
### 数据库名称定义规则

1. 不能是空串
1. 不得含有`/`、`\`、`？`、`$`、空格、空字符等等，基本只能使用ASCII中的字母和数字
1. 区分大小写，建议全部小写
1. 最多为**64**字节
1. 不得使用保留的数据库名，比如：`admin`，`local`，`config`

注意：数据库最终会成为文件，数据库名就是文件的名称

### 集合名称定义规则

1. 不能是空串
1. 不能包含 `\0字符` 空字符），这个字符表示集合名的结束，也不能包含 `$`
1. 不能以 `system.` 开头，这是为系统集合保留的前缀。

### 文档的键的定义规则

1. 不能包含 `\0` 字符（空字符），这个字符表示键的结束
1. `.` 和 `$` 是被保留的，只能在特定环境下用
1. 区分类型，同时也区分大小写
1. 键不能重复

注意：文档的键值对是有顺序的，相同的键值对如果有不同顺序的话，也是不同的文档。

### MongoDB基本的数据类型

|数据类型 |描述 |举例 |
|---|---|---|
|null |表示空值或者未定义的对象 |`{""x"":null}`|
|布尔值 |真或者假：true或者false |`{""x"":true} `|
|32位整数 |shell不支持该类型，默认会转换成64位浮点数，也可以使用NumberInt类 |`{“x”:NumberInt(“3”)}`|
|64位整数 |shell不支持该类型，默认会转换成64位浮点数,也可以使用NumberLong类 |`{“x”:NumberLong(“3”)}`|
|64位浮点数 |shell中的数字就是这一种类型 |`{""x""：3.14，""y""：3}`|
|字符串 |UTF-8字符串 |`{""foo"":""bar""}`|
|符号 |shell不支持，shell会将数据库中的符号类型的数据自动转换成字符串 | |
|对象id |文档的12字节的唯一id |`{""id"": ObjectId()}`|
|日期 |从标准纪元开始的毫秒数 |`{""date"":new Date()}`|
|正则表达式 |文档中可以包含正则表达式，遵循JavaScript的语法 |`{""foo"":/foobar/i}`|
|代码 |文档中可以包含JavaScript代码 |`{""x""：function() {}}`|
|未定义 |undefined |`{""x""：undefined}`|
|数组 |值的集合或者列表 |`{""arr"": [""a"",""b""]}`|
|内嵌文档 |文档可以作为文档中某个key的value |`{""x"":{""foo"":""bar""}}`|

## 增删改（CUD）操作

接下来我在命令行客户端中操作数据，这里假设你已经进入命令行。如果你没有与我保持同步，请再看一看上面的教程。

* 运行shell，命令：mongo ip:port，在这里我使用的默认的端口所以就没有指定
    ```shell
    [root@bogon mongodb-linux-x86_64-rhel70-3.4.1]# ./bin/mongo
    ```
* 显示现有的数据库，命令：`show dbs;`
    ```shell
    > show dbs
    admin  0.000GB
    local  0.000GB
    ```
* 显示当前使用的数据库，命令：`db`
    ```shell
    > db
    test
    ```
* 切换当前使用的数据库，命令：`use 数据库名称`
    ```shell
    > use local
    switched to db local
    ```
* 创建数据库：MongoDB没有专门创建数据库的语句，可以使用 `use` 来使用某个数据库，如果要使用的数据库不存在，那么将会创建一个，会在真正向该库加入文档后，保存成为文件。
    ```shell
    > use mall
    switched to db mall
    ```
* 删除数据库，命令：`db.dropDatabase()`，在使用这个命令前要记得切换到将被删除的数据，可以用上面学到的命令确认一下。
    ```shell
    > db
    mall
    > db.dropDatabase()
    { ""dropped"" : ""mall"", ""ok"" : 1 }
    ```
* 显示现有的集合，命令：`show collections`，我们切换到一个新的数据库，到里面插入一个集合并保存一条数据，这样做只是为了造些语句便于演示这个命令。
    ```shell
    > use mall
    switched to db mall
    > db.user.insert({name:'张三'})
    WriteResult({ ""nInserted"" : 1 })
    > show collections
    user
    ```
* 创建集合：在MongoDB中不用创建集合，因为没有固定的结构，直接使用db.集合名称.命令 来操作就可以了。如果非要显示创建集合的话，用：`db.createCollecion('集合名称');`
    ```shell
    > db.createCollection('system')
    { ""ok"" : 1 }
    ```
* 插入并保存文档
  1. insert方法，可以单独插入一个文档，也可以插入多个，用 `[ ]` 即可。
  1. MongoDB会为每个没有 `_id` 字段的文档自动添加一个 `_id` 字段
  1. 每个Doc必须小于16MB
* 删除文档，命令：`remove`，可以按条件来删除只是删除文档，集合还在，如果使用 `drop` 命令，会连带集合和索引都删掉。
* 查看集合中所有的文档，命令：`db.集合名称.find();`
    ```shell
    > db.user.find()
    { ""_id"" : ObjectId(""5871e5f99423674edcea4eec""), ""name"" : ""张三"" }
    > db.user.remove({})
    WriteResult({ ""nRemoved"" : 1 })
    > db.user.find()
    >
    ```
* 查看集合的状态信息：`db.集合名.stats();`
    ```shell
    > db.user.stats()
    {
        ""ns"" : ""mall.user"",
        ""size"" : 0,
        ""count"" : 0,
        ""storageSize"" : 20480,
        ""capped"" : false,
        ""wiredTiger"" : {...},
        ...
    }
    ```
* 查看集合中第一个文档，命令：`db.集合名称.findOne({条件对象})`。下面伪造一些数据方便测试
    ```shell
    > db.user.insert([{name:'李四'},{name:'王五'}])
    BulkWriteResult({
        ""writeErrors"" : [ ],
        ""writeConcernErrors"" : [ ],
        ""nInserted"" : 2,
        ""nUpserted"" : 0,
        ""nMatched"" : 0,
        ""nModified"" : 0,
        ""nRemoved"" : 0,
        ""upserted"" : [ ]
    })
    > db.user.findOne()
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
    ```
* 文档替换，命令： `db.集合名称. update(条件，新的文档);`
    ```shell
    > db.user.update({name:'王五'}, {name:'王五',sex:'男'})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find()
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eee""), ""name"" : ""王五"", ""sex"" : ""男"" }
    ```
* 上面的操作说明：先找到名称是王五的这一条记录，然后用新文档替换数据原来的所有值，是一个整体替换的操作。
* save方法
  1. 如果文档存在就更新，不存在就新建，主要根据 `_id` 来判断。可以看到我们插入了一个名称相同的文档，结果是成功的。
  ```shell
  > db.user.save({name:'李四'})
  WriteResult({ ""nInserted"" : 1 })
  > db.user.find()
  { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
  { ""_id"" : ObjectId(""5871e8fe9423674edcea4eee""), ""name"" : ""王五"", ""sex"" : ""男"" }
  { ""_id"" : ObjectId(""5871eb8e9423674edcea4eef""), ""name"" : ""李四"" }
  ```
* upsert
  1. 找到了符合条件的文档就更新，否则会以这个条件和更新文档来创建一个新文档。指定update方法的第三个参数为true，可表示是upsert
    ```shell
    > db.user.update({name:""张三""}, {name:""张三"",sex:'女'})
    WriteResult({ ""nMatched"" : 0, ""nUpserted"" : 0, ""nModified"" : 0 })
    > db.user.find()
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eee""), ""name"" : ""王五"", ""sex"" : ""男"" }
    { ""_id"" : ObjectId(""5871eb8e9423674edcea4eef""), ""name"" : ""李四"" }
    ```
* 上面的修改没有成功，原因就在于根据条件查询不到张三这个文档，那下面我们给他换上第三个参数。我们希望找不到就增加，找到就修改。
    ```shell
    > db.user.update({name:""张三""}, {name:""张三"",sex:'女'}, true)
    WriteResult({
        ""nMatched"" : 0,
        ""nUpserted"" : 1,
        ""nModified"" : 0,
        ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489"")
    })
    > db.user.find()
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eee""), ""name"" : ""王五"", ""sex"" : ""男"" }
    { ""_id"" : ObjectId(""5871eb8e9423674edcea4eef""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""女"" }
    ```
* 从查询出来的结果中我们看到，要修改的记录没有找到就变成新增操作了。
  1. 查询更新了多少个文档
  1. 使用命令：`getLastError`，返回最后一次操作的相关信息，里面的 `n` 就是更新的文档的数量。
    ```shell
    > db.runCommand({""getLastError"":1})
    {
        ""connectionId"" : 3,
        ""n"" : 0,
        ""syncMillis"" : 0,
        ""writtenTo"" : null,
        ""err"" : null,
        ""ok"" : 1
    }
    ```

### 更新修改器，用来做复杂的更新操作

上面介绍的修改操作是整体替换的，如果想局部修改怎么办呢？

* 我准备了测试数据如下
    ```shell
    > db.user.find()
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eed""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871e8fe9423674edcea4eee""), ""name"" : ""王五"", ""sex"" : ""男"" }
    { ""_id"" : ObjectId(""5871eb8e9423674edcea4eef""), ""name"" : ""李四"" }
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""女"" }
    ```
* `$set` 指定一个字段的值，如果字段不存在，会创建一个
    ```shell
    > db.user.update({name:'张三'}, {$set:{sex:'保密'}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"" }
    ```
* 如果指定修改的字段不存在则新增加
    ```shell
    > db.user.update({name:'张三'}, {$set:{nick:'小三'}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""nick"" : ""小三"" }
    ```
* `$unset` 删掉某个字段
    ```shell
    > db.user.update({name:'张三'}, {$unset:{nick:1}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"" }
    ```
* `$inc` 用来增加已有键的值，如果字段不存在，会创建一个。只能用于整型、长整型、或双精度浮点型的值。
    ```shell
    > db.user.update({name:'张三'}, {$inc:{age:12}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12 }
    ```
* `$push` 向已有数组的末尾加入一个元素，要是没有就新建一个数组
    ```shell
    > db.user.update({name:'张三'}, {$set:{group:[4,2]}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 4, 2 ] }
    > db.user.update({name:'张三'}, {$push:{group:5}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 4, 2, 5 ] }
    ```
* `$each` 通过一次 `$push` 来操作多个值，同时push多个值
    ```shell
    > db.user.update({name:'张三'}, {$push:{group:{$each:[4,2,6,5]}}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 4, 2, 5, 4, 2, 6, 5 ] }
    ```
* `$slice` 限制数组只包含最后加入的n个元素，其值必须是负整数
    ```shell
    > db.user.update({name:'张三'}, {$push:{group:{$each:[4,2,6,5],$slice:-3}}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 2, 6, 5 ] }
    ```
* `$sort` 对数组中的元素，按照指定的字段来对数据进行排序（1为升序，-1为降序），然后再按照slice删除。注意：不能只将$slice或者$sort与$push配合使用，且必须使用 `$each`
    ```shell
    > db.user.update({name:'张三'}, {$push:{group:{$each:[4,2,6,5],$slice:-3, $sort:1}}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 6 ] }
    ```
* `$ne` 判断一个值是否在数组中，如果不在则添加进去
    ```shell
    > db.user.update({name:'张三', group:{$ne:7}}, {$push:{group:7}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 6, 7 ] }
    > db.user.update({name:'张三', group:{$ne:7}}, {$push:{group:7}})
    WriteResult({ ""nMatched"" : 0, ""nUpserted"" : 0, ""nModified"" : 0 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 6, 7 ] }
    ```
* `$addToSet` 将数组作为数据集使用，以保证数组内的元素不会重复
    ```shell
    <pre class=""line-numbers prism-highlight"" data-start=""1""><code class=""language-bash"">> db.user.update({name:'张三'}, {$addToSet:{group:7}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 0 })
    ```
* `$pop` 从数组一端删除元素，`{$pop:{key:1}}`，从末尾删掉一个，-1则从头部删除
    ```shell
    > db.user.update({name:'张三'}, {$pop:{group:1}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    ```
* `$pull` 按照条件来删除所有匹配的元素
    ```shell
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 6, 7, 7 ] }
    > db.user.update({name:'张三'}, {$pull:{group:7}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 6 ] }
    ```
* `$` 用来修改第一个匹配的元素
    ```shell
    > db.user.update({name:'张三'}, {$set:{'group.2':8}})
        WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
        > db.user.find({name:'张三'})
        { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 6, 8 ] }
    ```
* 像下面这种就是根据数组的索引来替换值
    ```shell
    > db.user.update({name:'张三', 'group.1':6}, {$set:{'group.$':4}})
    WriteResult({ ""nMatched"" : 1, ""nUpserted"" : 0, ""nModified"" : 1 })
    > db.user.find({name:'张三'})
    { ""_id"" : ObjectId(""5871ecba7f264dfcaeec2489""), ""name"" : ""张三"", ""sex"" : ""保密"", ""age"" : 12, ""group"" : [ 5, 4, 8 ] }
    ```

以上的命令一定要自己都跟着练习几次，如此这些东西才能算是你学到的。学了和学到是两个完全不同的状态。
