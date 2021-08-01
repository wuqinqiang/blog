---
title: "推荐两款go开发中提高效率工具"
date: 2021-04-25T22:25:52+08:00 
toc: true 
tags:

- go
- enum


---



### 介绍

推荐两款 `go` 开发中用的还行的工具。

为什么推荐工具？是为了让评论区的大佬介绍其他更好用的工具，解放我的双手。

顺便问问，有没有关说话就能自动打完代码的工具？



### JSON-To-Stuct



这个工具可以把 `json` 格式的数据转换成 `go` 的 `struct`。这样的话比如你在对接第三方的时候，就不需要根据对方的接口一个个定义 `struct` 字段，就像下面示例一样,直接复制微信小商店订单 `json` 数据到网站的左框即可,当然自己还是需要做一些局部的调整。

![image](https://image.syst.top/image/go-tool/1.png)



其实这个功能21年新版的 `goland` 也支持了。在 `goland` 中你只需要这样,

![image](https://image.syst.top/image/go-tool/2.gif)



### Table-To-Stuct

被业务缠身的同学每天免不了 `CURD`。`CURD` 之前总得建表吧。建表之后总得在代码中定义模型吧。总不能又一个个字段定义过去吧，那么下面这个工具可能管用。

假设你有一个库 `dream`，有一个表 `category`，结构如下，

```sql
CREATE TABLE `category` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(20) NOT NULL DEFAULT '',
  `parent_id` int(11) unsigned NOT NULL DEFAULT '0',
  `created_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=23 DEFAULT CHARSET=utf8mb4;
```



你只需引入包 `github.com/gohouse/converter` ,然后写这样的代码，就可以实现 `table-to-go` 功能。

```go
package main

import (
	"fmt"
	"github.com/gohouse/converter"
)

func main() {
	// 初始化
	t2t := converter.NewTable2Struct()
	// 个性化配置
	t2t.Config(&converter.T2tConfig{
		// 如果字段首字母本来就是大写, 就不添加tag, 默认false添加, true不添加
		RmTagIfUcFirsted: false,
		// tag的字段名字是否转换为小写, 如果本身有大写字母的话, 默认false不转
		TagToLower: false,
		// 字段首字母大写的同时, 是否要把其他字母转换为小写,默认false不转换
		UcFirstOnly: false,
		//// 每个struct放入单独的文件,默认false,放入同一个文件(暂未提供)
		//SeperatFile: false,
	})
	// 开始迁移转换
	err := t2t.
		// 指定某个表,如果不指定,则默认全部表都迁移
		Table("category").
		//// 表前缀
		//Prefix("prefix_").
		// 是否添加json tag
		EnableJsonTag(true).
		// 生成struct的包名(默认为空的话, 则取名为: package model)
		PackageName("model").
		// tag字段的key值,默认是orm
		TagKey("orm").
		// 是否添加结构体方法获取表名
		RealNameMethod("TableName").
		// 生成的结构体保存路径
		SavePath("model/category.go").
		// 数据库dsn,这里可以使用 t2t.DB() 代替,参数为 *sql.DB 对象
		Dsn("root:Passw0rd@tcp(localhost:3306)/dream?charset=utf8").
		// 执行
		Run()

	fmt.Println(err)
}
```

运行这段代码，最后会根据设置的 `SavePath` 里的地址(尚未存在的目录需要先自行创建)，生成 `category.go` 文件，内容如下，

```go
package model

type Category struct {
	Id        int    `orm:"id" json:"id"`
	Name      string `orm:"name" json:"name"`
	ParentId  int    `orm:"parent_id" json:"parent_id"`
	CreatedAt string `orm:"created_at" json:"created_at"`
	UpdatedAt string `orm:"updated_at" json:"updated_at"`
}

func (*Category) TableName() string {
	return "category"
}
```

相应的再进行调整即可。



### 总结



今天主要分享的是 `json-to-stuct`、`table-to-stuct` 这两款日常可能会用上的工具。

好了，现在开始你们给我介绍趁手的工具了。











