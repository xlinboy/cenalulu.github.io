## 主要功能

#### Organization
> 个人使用时只要使用个人账户就足够了，但如果是公司，建议使用 Organization 账户。可以统一管理账户和权 限，还能统一支付一些费用。

#### Issue 功能

> 是将一个任务或问题分配给一个 Issue 进行追踪和管理的功能。每一个功能更改或修正都对应一个Issue，讨论或修正都以这个Issue 为中心进行。只要查看 Issue，就能知道和这个更改相关的一切信息，并以此进行管理。

#### Wiki
> 通过 Wiki 功能，任何人都能随时对一篇文章进行更改并保存，因 此可以多人共同完成一篇文章。

#### Pull Request
> 开发者向 GitHub 的仓库推送更改或功能添加后，可以通过 Pull
Request 功能向别人的仓库提出申请，请求对方合并。


# Git

#### GIT是啥
- Git是仓库管理工具是github的核心。   
- Git 属于分散型版本管理系统，是为版本管理而设计的软件。

#### 什么是版本管理
> 版本管理就是管理更新的历史记录。它为我们提供了一些在软件开发过程中必不可少的功能，例如记录一款软件添加或更改源代码的过程，回滚到特定阶段，恢复误删除的文件等。

#### 集中型和分散型

##### 集中型

![image](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/%E5%88%86%E6%95%A3%E5%9E%8B.PNG)

> 将仓库集中存放在服务器之中，所以只存在一个仓库。

优点
- 易于管理

缺点
- 服务器凉了，就无法继续开发
- 数据易丢失

#### 分散型

> 分散型拥有多个仓库，相对而言稍显复杂。不过，由于
本地的开发环境中就有仓库，所以开发者不必连接远程仓库就可以进行开发。

优点
- 仓库之间都可以进行 push 和 pull。
- 不通过 GitHub，开发者 A 也可以直接向开发者 B 的
仓库进行 push 或 pull

缺点：
- 在使用前如果不事先制定规范，初学者往
往会搞不清最新的源代码保存在哪里，导致开发失去控制

![image](https://xlactive-1258062314.cos.ap-chengdu.myqcloud.com/Subversion.png)


### 初始设置

- ~/.gitconfig

```
用英文
$ git config --global user.name "Firstname Lastname"
$ git config --global user.email "your_email@example.com"

# 提高可读性 
$ git config --global color.ui auto
```

### 使用前的准备

#### 设置SSH key

```
 ssh-keygen -t rsa -C "your_email@example.com"
 
 # 这里的密码是免密登陆github时，验证使用
 Enter passphrase (empty for no passphrase): 请输入密码
 
```

上个步骤成功后会生成
```
Your identification has been saved in /Users/your_user_directory/.ssh/id_rsa.
Your public key has been saved in /Users/your_user_directory/.ssh/id_rsa.pub.

```
id_rsa 文件是私有密钥，id_rsa.pub 是公开密钥。

#### 添加公开密钥
> 在 GitHub 中添加公开密钥，今后就可以用私有密钥进行认证了。 Key 部分请粘贴 id_rsa.pub 文件里的内容。id_rsa.pub
的内容可以用如下方法查看

```
cat ~/.ssh/id_rsa.pub
ssh-rsa 公开密钥的内容 your_email@example.com

```
添加成功之后，创建账户时所用的邮箱会接到一封提示“公共密钥添加完成”的邮件。   


接下来进行免密登陆
```
ssh -T git@github.com
```

### 创建仓库
- Repository name 仓库名称。
- Description 栏中可以设置仓库的说明。选填
- Public、Private。
- Initialize this repository with a README：GitHub 会自动初始化仓库并设置 README 文件，让用户可以立刻clone仓库。
- Add .gitignore：通过它可以在初始化时自动生
成.gitignore文件。 .gitignore文件该文件用来描述 Git 仓库中不需管理的文件与目录。
- Add a license：选择要添加的许可协议文件。将自动
生成包含许可协议内容的 LICENSE 文件，用来表明该仓库内容的许可协议。


### 连接仓库
```
https://github.com/%E7%94%A8%E6%88%B7%E5%90%8D/Hello-Word
```
- README.md：文件的内容
会自动显示在仓库的首页当中。人们一般会在这个文件中标明本
仓库所包含的软件的概要、使用流程、许可协议等信息。
- GitHub Flavored Markdown


### 公开代码

- clone 已有仓库
```
git clone git@github.com:hirocastest/Hello-World.git
```
这里会要求输入 GitHub 上设置的公开密钥的密码。认证成功后，
仓库便会被 clone 至仓库名后的目录中。将想要公开的代码提交至这个
仓库再 push 到 GitHub 的仓库中，代码便会被公开。

- 编写代码
```
查看git 状态
git status
```

- 提交
```
git add hello_world.php

git commit -m "Add hello world script by php"
```
通过 git add命令将文件加入暂存区 ，再通过 git commit命令提交。

```
日志
git log
```
- 进行 push
 push，GitHub 上的仓库就会被更新。
```
git push [url]
```

### 通过实际操作学习Git

#### 基础操作

- `git init`——初始化仓库

  ```shell
  # .git 目录里存储着管理当前目录内容所需的仓库数据。
  # git-tutorial 里面的内容一般叫做“附属于该仓库的工作树”
  $ mkdir git-tutorial
  $ cd git-tutorial
  $ git init
  ```

- `git status`——查看仓库的状态

- `git add`——向暂存区中添加文件

  ```shell
  $ git add README.md
  $ git status
  ```

-  `git commit`——保存仓库的历史记录

  ```shell
  # 记述一行提交信息
  $ git commit -m "First commit"
  ```

- `git log`——查看提交日志

  ```shell
  $ git log
  # 只显示提交信息的第一行
  $ git log --pretty=short
  # 只显示指定目录、文件的日志
  $ git log README.md
  ```

- `git diff`——查看更改前后的差别

#### 分支的操作

- git branch——显示分支一览表

- git checkout  - b——创建、切换分支

  ```shell
  # 切换到 feature - A 分支并进行提交
  $ git checkout -b feature-A
  $ git branch
  $ git add README.md
  $ git commit -m "Add feature-A"
  #  切换到 master 分支
  $ git checkout master
  # 切换回上一个分支
  $ git checkout -
  ```

- git merge——合并分支

  ```shell
  # 切换到merge分支
  $ git checkout master
  $ git merge --no-ff feature-A
  ```
