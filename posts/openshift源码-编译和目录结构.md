# openshift源码 - 编译和目录结构篇

 介绍openshift的源码编译和目录结构组织，为了方便代码调试和了解大型Golang项目的构建方式

### 编译

无论是openshift还是Kubernetes等大型Golang项目都用到了`Makefile`, 所以有必要从此开始说起，这里只说项目里用到的makefile特性，想了解更多的可以参考[跟我一起写Makefile](http://scc.qibebt.cas.cn/docs/linux/base/%B8%FA%CE%D2%D2%BB%C6%F0%D0%B4Makefile-%B3%C2%F0%A9.pdf)

#### Makefile介绍

> makefile 关系到了整个工程的编译规则。一个工程中的源文件不计数，其按类型、功能、
模块分别放在若干个目录中，makefile 定义了一系列的规则来指定，哪些文件需要先编译，
哪些文件需要后编译，哪些文件需要重新编译，甚至于进行更复杂的功能操作，因为
makefile 就像一个 Shell 脚本一样，其中也可以执行操作系统的命令。 makefile 带来的好
处就是——“自动化编译”，一旦写好，只需要一个 make 命令，整个工程完全自动编译，
极大的提高了软件开发的效率。

Makefile里的规则，就在做两件事，一个是指明依赖关系，另一个是生成目标的方法

Golang项目里用到的Makefile规则比较简单，基本就是定义一个目标的生成方法，下面的示例是Openshift项目里makefile中定义的第一个目标。

```makefile
all build:
	hack/build-go.sh $(WHAT) $(GOFLAGS)
.PHONY: all build

```

- `all build`，是定义的目标，看到这个就知道可以在源码的根目录上执行`make all build`来编译了

- 第二行说明生成目标的方法，就是去hack目录下执行build-go.sh脚本，这里还支持传入一些参数

- 第三行 `.PHONY`，起到一个标识的作用，没什么实际意义，是用来告诉make命令，这里是个伪目标，不需要生成这个文件，因为默认被定义的目标是需要产出为各种.o目标文件的

在 Makefile 中，像这种目标规则的顺序也是很重要的，第一个目标会成为最终的目标，也可以说成是默认目标，所以在openshift的根目录上直接执行`make`, 等效于`make all build`

还可以自己决定是否编译出镜像或者rpm包（make release, make build-rpms）

#### 编译openshift

上边介绍了，直接敲`make`就可以自动编译出所有平台（linux, mac, windows）的二进制，编译前介绍两个hack方法，

- 在hack/build-go.sh的第二行加上`set -x`， 这样的话，shell脚本在运行时，里面的所有变量和执行路径会全部打印出来，一目了然，不用自己一行一行的加echo debug了
- 如下修改hack/build-cross.sh，不然会编译出多平台的二进制，花的时间略长啊。。。

  ```
  # by default, build for these platforms
  platforms=(
    linux/amd64
    # darwin/amd64
    # windows/amd64
    # linux/386
  )
  ```

下面简易说下执行make后，都发生了什么，只会捡关键点说。

```shell
➜  origin git:(xxpDev) ✗ make

hack/build-go.sh  

# 初始化一大堆变量，关键函数都在common.sh里实现的
source hack/common.sh hack/util.sh hack/lib目录下的所有脚本

# 还会改动GOPATH,然后会在$GOPATH/src/github.com/openshift下建个软连指向origin目录
export GOPATH=_output/local/go

# 最终组合成下面一条最原始的命令，来进行编译
go install \
  -pkgdir /home/xxp/Github/src/github.com/openshift/origin/_output/local/pkgdir/linux/amd64 \
  -tags ' ' \
  -ldflags '-X github.com/openshift/origin/pkg/bootstrap/docker.defaultImageStreams=centos7 \
    -X github.com/openshift/origin/pkg/cmd/util/variable.DefaultImagePrefix=openshift/origin \
    -X github.com/openshift/origin/pkg/version.majorFromGit=3 \
    -X github.com/openshift/origin/pkg/version.minorFromGit=6+ \
    -X github.com/openshift/origin/pkg/version.versionFromGit=v3.6.0-alpha.0+83e3250-176-dirty \
    -X github.com/openshift/origin/pkg/version.commitFromGit=83e3250 \
    -X github.com/openshift/origin/pkg/version.buildDate=2017-04-06T05:34:29Z \
    -X github.com/openshift/origin/vendor/k8s.io/kubernetes/pkg/version.gitCommit=43a9be4 \
    -X github.com/openshift/origin/vendor/k8s.io/kubernetes/pkg/version.gitVersion=v1.5.2+43a9be4 \
    -X github.com/openshift/origin/vendor/k8s.io/kubernetes/pkg/version.buildDate=2017-04-06T05:34:29Z \
    -X github.com/openshift/origin/vendor/k8s.io/kubernetes/pkg/version.gitTreeState=clean' \
  github.com/openshift/origin/cmd/openshift \
  github.com/openshift/origin/cmd/oc \
  github.com/openshift/origin/pkg/sdn/plugin/sdn-cni-plugin \
  github.com/openshift/origin/vendor/github.com/containernetworking/cni/plugins/ipam/host-local \
  github.com/openshift/origin/vendor/github.com/containernetworking/cni/plugins/main/loopback
```

可以看到openshift会编译出5个二进制来，其中3个和网络CNI接口有关，最后会放置到_output/local/bin/linux/amd64, 并作相关的软链接（oadm, kubelet）

所以以后分析程序的切入点就从cmd/openshift和 cmd/oc入手就行了

来看下编译成果

```
➜  origin git:(xxpDev) ✗ _output/local/bin/linux/amd64/oc version
oc v3.6.0-alpha.0+83e3250-176-dirty
kubernetes v1.5.2+43a9be4
features: Basic-Auth
```

看到输出`v3.6.0-alpha.0+83e3250-176-dirty`， 这就是上面编译时传进去的参数。

`-X github.com/openshift/origin/pkg/version.majorFromGit=3`,意思是说编译文件`github.com/openshift/origin/pkg/version.go`时，对常量majorFromGit赋值为3

### 项目目录结构

 -- 未完待续
