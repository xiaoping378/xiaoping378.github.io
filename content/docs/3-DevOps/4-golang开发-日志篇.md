---
tags: ["Dev"]
title: "Golang开发-glog日志库"
linkTitle: "Golang开发-glog日志库"
weight: 4
date: 2017-01-04
description: >
  介绍Golang开发中的glog日志库
---

基于Golang 1.7.5版本

软件项目里的日志输出是很重要的环节，可以用于日后BI分析，或者线上调试（万能调试大法printf）等等。

> 对于当年刚入软件行业时，自己的printf("11111\n")的做法，记忆犹新呀，调试完再删掉自己胡乱加的打印语句，偶尔还有漏删的情况，就commit，push上去了。

golang语言里有个`golang/glog`包，是类似google内部[glog](https://github.com/google/glog)的开源实现，其可以做到无侵入式调试程序，主要是通过启动时命令行传参来控制打印级别。

有以下特性，

- 有四个级别的打印 Info, Warning, Error, Fatal，级别越来越高，分别都支持格式化输出Infof, Warningf, Errorf, Fatalf
- 支持 -v传参，指定打印级别
- 支持 -vmodule=file=2， 指定特殊文件开启打印，避免日志输出过多。
- 支持 -log_dir="", 指定日志输出目录， 默认会按级别输出/tmp目录下， 高级别的会记录到低级别里日志文件里

下面举个简单的例子

```go
//file name: glog.go
package main

import (
	"flag"
	"github.com/golang/glog"
)

func main() {
	flag.Parse()
	//flag.Set("logtostderr", "true")
	defer glog.Flush()

	glog.Info("这里是Info级别的日志")
	glog.Warning("这里是Warning级别的日志")
	glog.Errorf("这里是Error级别的日志: %s", "error")

	glog.V(3).Infoln("级别3的日志")
}
```

 代码如上运行，你需要执行`go get github.com/golang/glog`下载依赖包， 然后运行

 ```
 ➜  go run glog.go -v 3
E0408 09:35:38.703186    8663 glog.go:15] This is a Error log error
 ```

 - Error级别的会输出到标准输出，并记录到文件，
 - 日志默认输出到/tmp目录， 每次执行都会记录新的文件，日志文件如下样式命名， glog是文件名，air13是主机名，xxp是用户名

	```
	 ➜  ls /tmp/glog.* -l
	-rw-rw-r-- 1 xxp xxp 260 4月   8 10:11 /tmp/glog.air13.xxp.log.ERROR.20170408-101153.12771
	-rw-rw-r-- 1 xxp xxp 405 4月   8 10:11 /tmp/glog.air13.xxp.log.INFO.20170408-101153.12771
	-rw-rw-r-- 1 xxp xxp 334 4月   8 10:11 /tmp/glog.air13.xxp.log.WARNING.20170408-101153.12771
	lrwxrwxrwx 1 xxp xxp  46 4月   8 10:11 /tmp/glog.ERROR -> glog.air13.xxp.log.ERROR.20170408-101153.12771
	lrwxrwxrwx 1 xxp xxp  45 4月   8 10:11 /tmp/glog.INFO -> glog.air13.xxp.log.INFO.20170408-101153.12771
	lrwxrwxrwx 1 xxp xxp  48 4月   8 10:11 /tmp/glog.WARNING -> glog.air13.xxp.log.WARNING.20170408-101153.12771
	```
 - `-v 3` 指定运行时的记录的日志级别，因为3>=3, 这样`V(3).Infoln`会输出， 如果是传入 `-v 2`, 则不会
 - ` go run glog.go -v 2 -logtostderr=true`， 则会关闭记录文件，输出到标准输出。
 当然可以在程序里加上 `flag.Set("logtostderr", "true")`来关闭记录文件。

总之，glog包对日志的控制非常实用和灵活，大型Golang项目必备包之一。
