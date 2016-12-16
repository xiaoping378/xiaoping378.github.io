# github开源项目的正确贡献姿势

常见的开源项目贡献指导里都是差不多的样子：

    * 要先fork
    * 然后change something
    * 再然后fetch，rebase
    * push origin， 最后发起pull request
 
具体到不同的项目，可能会要求更多的细节步骤,但大体如上。

这些都没错，但实际操作起来，和习惯不符。因为我一般是先clone一个项目，然后使用中发现有问题，会尝试去修改，fix OK的话，才会想着去贡献代码
可事情到了这一步，再按照一开始的方式操作，会平白无辜耗费很多时间，还涉及到已经修改完代码如何同步过去的问题。
以下是我个人总结的一套方法，屡试不爽乎。

这里我以k8s项目的贡献经历来举例，以备不时之需。git这个东西，不常用，会忘记的，即使你已经理解原理了。。。

* 首先clone K8s的项目代码。
    ```
    git clone https://github.com/kubernetes/kubernetes.git
    ```

* 然后自行编译 **make**, 使用中发现一些问题

    就会去github的issue里找找看。。。竟然没人提这个问题，问问同事或同行，人家表示没碰到过你的问题，
    好吧，自己尝试去修改... 不断编译 ... 测试... 最终OK了，我要贡献代码！！！
    完啦，没有按照最佳习惯来，改动前忘记新建分支了。。。（这个习惯很重要，可以省掉很多麻烦）
    只能如此操作了，可以来个大挪移到新建分支上

  ```
  git stash
  git checkout -b fix_something
  git stash apply
  ```

* 此时去github上fork下原项目，拿到fork后的项目地址，再来个偷天换日。
  ```
  git remote rename origin upstream
  git remote add origin git@github.com:xiaoping378/kubernetes.git
  ```

* 再然后就可以按一开始介绍的，fetch, rebase, push origin, 发起PR了。
  ```
  git fetch upstream
  git rebase upstream/master
  # 有冲突就git mergetool
  git push origin fix_something 
  # 然后去github页面发起pull request即可。
  ```

* 注意事项
    值的注意的是，以后在master分支上**git pull**，就是从upstream/master那里拉取的，和一般情况不一样的地方。
    这样会少了烦人的merge msg（-ff可以解决）， 还可以用简单的pull来同步上游代码。
    更可以意淫自己是原项目的核心开发人员了。。。
    以后本地同步fork的项目到上游的最新状态，这样操作：
  ```
  git checkout master
  git pull
  git push origin master
  ```
    其实github上那个fork的项目，只是用来提PR的，这样可以在原项目的分支上任意玩耍了。当然你也可以用来备份一些比较大的feature

* 最后记录下个人的git global配置
  ```
  # cat ~/.gitconfig
  [user]
    email = xiaoping378@163.com
    name = xiaoping378
  [merge]
    tool = meld
  [push]
    default = simple
  [core]
    quotepath = false
  ```
