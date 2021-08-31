# 使用repo 管理一个工程的多个git仓库



## repo介绍

Repo 是建立在 Git 上的一个多仓库管理工具，可以组织多个仓库的上传和下载。Repo 是 Google 用 Python 脚本写的调用 Git 的脚本，主要帮助我们管理多个 Git 存储仓库，将其上传到我们的版本控制系统，并自动执行 Android 开发工作流程的某些部分。Repo 并不是要取代 Git，而是为了领用git管理包含众多git仓库的Android源码。



## 安装repo

```
mkdir ~/bin
PATH=~/bin:$PATH  # 本身加到 ~/.zshrc or ~/.bashrc or /etc/profile 里 

curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo

# 最后 source 下环境变量
```



## 创建工程Manifest文件



在gitlab上创建一个git仓库，然后clone到本地，创建工程对应的manifest.xml文件上传到仓库



```
<?xml version="1.0" encoding="UTF-8"?>
<manifest>

  <remote name="origin"
          fetch="ssh://github.com/group2/"/>

  <default revision="master"
           remote="origin"
           sync-j="4" />

  <project path="lazada_main" name="lazada_main.git" />

  <project path="lazandroid_homepage_aar" name="lazandroid_homepage_aar.git" />

</manifest>
```

解释下常用的各节点：

#### remote

远程仓库地址配置，可以多个。

- name: 名字，也用于子仓库的 git remote 名称（.git/config 里的 remote）
- alias: 别名，可省略，建议设为 `origin`， 设置了那么子仓库的 git remote 即为此名，方便不同的 name 下可以最终设置生成相同的 remote 名称
- fetch: 仓库地址`前缀`，即 project 的仓库地址为: `remote.fetch + project.name`
- pushurl: 一般可省略，省略了则直接用 fetch
- review: Gerrit code review 的地址，如果没有用 Gerrit 则不需要配置（也就不能用 repo upload 命令了）
- revision: 使用此 remote 的默认分支

**这里的 fetch 遇到个坑**，`git@github.com:group2/` 这样 git scheme 开头的地址会有问题（ssh/https 的正常），repo不支持，如果使用最终子仓库在本地的 remote 地址一直是 manifest 的地址拼上 fetch 再加 project 的 name (manifest.git_path + remote.fetch + project.name)。这算是repo的一个bug，这里我们需要把git scheme开头的地址转为ssh开头的地址。

​	例如：git@gitlab.xxx-inc.com:test/ 需要修改为 ssh://git@gitlab.xx-inc.com/test/ 

#### project

子项目仓库配置，可以多个。

- path: repo sync 同步时，相对于根目录的子仓库文件夹路径
- name: 子仓库的 git 仓库名称
- group: 分组
- revision: 使用的分支名
- clone-depth: 仓库同步 Git 的 depth

##### copyfile

project 的子节点属性.

- src: project 下的相对路径
- dest: 整个仓库根路径下的相对路径

##### linkfile

project 的子节点属性，类似 copyfile，只是把复制文件变为创建链接文件。

#### default

project 没有设置属性时会使用的默认配置，常用的是 remote 和 revision。

#### 其他

其他现在暂时用的较少，详细完整的 manifest 格式说明请看[官方文档](https://gerrit.googlesource.com/git-repo/+/HEAD/docs/manifest-format.md)。
了解了 Repo 下的 Manifest 配置仓库的概念后就可以根据自己的项目来创建了。



## 初始化repo仓库

创建工程目录，初始化repo

```
mkdir projectA
cd projectA
```

#### repo 依赖本地git仓库init

```
repo init -u file:////Users/ian/manifest_src/project_manifest  -m manifest.xml -b master
```

注： 这里只会clone已经提交到git仓库的manifest.xml文件到当前目录，本地修改但未提交到git仓库的改动不会带入。



### repo依赖远程git仓库的manifest文件init

```
repo init -u git@gitlab.xxx-inc.com:group1/project_manifest.git  -m manifest.xml -b master
```



## 下载代码/同步代码

```
repo sync
```

代码下载完成后，就可以愉快得开始开发了。

如果出现Broken pipe的报错，则添加下面的内容到~/.ssh/config后重新执行

```
Host *
   ServerAliveInterval 600
   TCPKeepAlive yes
   IPQoS=throughput
```



## 创建主题分支

当您开始进行更改（例如当您开始处理 bug 或实现新功能）时，请在本地工作环境中新建一个主题分支。主题分支**不是**原始文件的副本；它一个指针，指向某一项提交内容，可以简化创建本地分支以及在本地分支之间进行切换的操作。通过使用分支，您可以将工作的某个方面与其他方面分隔开来。请参阅[分隔主题分支](http://www.kernel.org/pub/software/scm/git/docs/howto/separating-topic-branches.txt)（一篇有关使用主题分支的有趣文章）。

如需使用 Repo 新建一个主题分支，请转到相应项目并运行以下命令：

```
repo start BRANCH_NAME .
```

尾随句点 (` .`) 代表当前工作目录中的项目。

如需验证新分支是否已创建，请运行以下命令：

```
repo status .
```

## 使用主题分支

如需将分支分配给特定项目，请运行以下命令：

```
repo start BRANCH_NAME PROJECT_NAME
```

如需查看所有项目的列表，请参阅 [android.googlesource.com](https://android.googlesource.com/)。如果您已导航到相应的项目目录，则只需使用一个句点来表示当前项目即可。

如需切换到本地工作环境中的另一个分支，请运行以下命令：

```
git checkout BRANCH_NAME
```

如需查看现有分支的列表，请运行以下命令：

```
git branch
```

或

```
repo branches
```

这两个命令均可返回现有分支的列表，并会在当前分支的名称前面标注星号 (*)。

**注意**：如果存在错误，可能会导致 `repo sync` 重置本地主题分支。如果在您运行 `repo sync` 之后，`git branch` 显示 *（无分支），请再次运行 `git checkout`。

## 暂存文件

默认情况下，Git 会检测到您在项目中所做的更改，但不会跟踪这些更改。如需让 Git 保存您的更改，您必须标记或暂存这些更改，以将其纳入到提交中。

如需暂存更改，请运行以下命令：

```
git add
```

此命令接受将项目目录中的文件或目录作为参数。`git add` 并不像其名称表示的这样只是简单地将文件添加到 Git 代码库，它还可以用于暂存文件的修改和删除的内容。

## 查看客户端状态

如需列出文件状态，请运行以下命令：

```
repo status
```

如需查看未提交的修改（**未**标记为需要提交的本地修改），请运行以下命令：

```
repo diff
```

如需查看已提交的修改（**已标记**为需要提交的本地修改），请确保您已转到相应的项目目录，然后运行包含 `cached` 参数的 `git diff`：

```
cd ~/WORKING_DIRECTORY/PROJECT
git diff --cached
```

## 提交更改

在 Git 中，提交是修订版本控制的基本单位，包含目录结构的快照以及整个项目的文件内容。 您可以使用以下命令在 Git 中创建提交：

```
git commit
```

当系统提示您输入提交消息时，请针对要提交至 AOSP 的更改提供一条简短（但有帮助）的消息。如果您不添加提交消息，提交将会失败。

