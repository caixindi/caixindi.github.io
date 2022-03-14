### Git 简介

Git is a [free and open source](https://git-scm.com/about/free-and-open-source) distributed version control system designed to handle everything from small to very large projects with speed and efficiency.

Git is [easy to learn](https://git-scm.com/doc) and has a [tiny footprint with lightning fast performance](https://git-scm.com/about/small-and-fast). It outclasses SCM tools like Subversion, CVS, Perforce, and ClearCase with features like [cheap local branching](https://git-scm.com/about/branching-and-merging), convenient [staging areas](https://git-scm.com/about/staging-area), and [multiple workflows](https://git-scm.com/about/distributed).

### Git 安装

进入Git官网下载适合系统平台的稳定版本的git，无脑下一步即可。Git 各平台安装包下载地址为：http://git-scm.com/downloads

下载完成后，需要配置一下用户信息：

Git提供了`git config`命令，可以用来配置相关的工作环境变量。这些环境变量决定了Git在各个环节的具体工作方式和行为。这些变量存放在一下三个不同的地方：

- `/etc/gitconfig`文件：系统配置，对所有用户都适用，使用`git config --system --list`可以查看相关的系统配置信息。

  ![image-20211119203542178](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119203542178.png)

- `~/.gitconfig`：用户配置，只适用于该用户，使用`git config --global --list`可以查看用户配置信息。

  ![image-20211119203932537](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119203932537.png)

- 工作目录中的 `.git/config` 文件：只对当前项目生效，并且配置是向上层覆盖的。

使用`git config --global`配置用户信息

```shell
git config --global user.name "xiaochenxin"
git config --global user.email "caixindi98@qq.com"
```

### Git 图解

工作区：存放项目代码的地方

暂存区(stage/index)：存放在 **.git** 目录下的 index 文件（.git/index）中，使用`git ls-files --stage`可以查看存放在暂存区的文件。

本地仓库：存放数据的地方，包含所有版本数据。也就是**.git**目录下的内容，其中的HEAD可以理解为一个指针，指向当前所在的分支指针，而这个分支指针又会指向当前分支的最新提交。

远程仓库：托管代码的服务器。

![image-20211119213426653](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119213426653.png)

![](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119212828639.png)

掌握图中的几个git命令基本上已经可以初步使用git了。

### Git 基础技能

#### Git 将更新应用到仓库

##### 查看状态

首先，我们必要要清楚，在工作目录下的每一个文件就只有两种状态：分别为已跟踪状态(tracked)和未跟踪状态(untracked)。已跟踪的文件会被纳入Git版本控制（就是Git已经知道的文件），在上一次的快照中会有它们的记录，它们的状态可能是未修改，已修改或者已放入暂存区。

而未跟踪文件不会在上次的快照记录中，并且也不会在暂存区中，在初次克隆某仓库的时候，你克隆下来的所有文件都属于已跟踪文件，并且是属于未修改状态。

如图是文件状态的变换过程：

![image-20211119225236484](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119225236484.png)

我们可以使用`git status`命令来查看文件处于什么状态，对一个刚刚克隆的项目使用该命令，会有如下的输出：

```shell
$ git status
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working directory clean
```

这就表示当前文件在上次提交后都未更改过，并且当前目录下不存在未跟踪的文件，并且还告知我们当前所在分支与远程服务器上对应的分支没有发生偏离。

接下来尝试跟踪一个新的文件：

执行`git add test.txt`并且执行`git status`命令，显示如下：

![image-20211119230910750](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119230910750.png)

此时发现文件已经被跟踪并且处于暂存状态。接下来我们修改一下这个`test.txt`文件（添加点内容），然后执行`git status`命令，显示如下：

![image-20211119231423870](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119231423870.png)

这表示文件的内容已经更新但还没有放到暂存区里面，在暂存区里的那个文件并不是最新的，也就是这两个文件的版本并不是一致的，此时需要执行`git add test.txt`命令。再次查看status，显示如下：

![image-20211119231719771](C:/Users/admin/AppData/Roaming/Typora/typora-user-images/image-20211119231719771.png)

所以`git add`不能仅仅只理解为将一个文件添加到项目中，而是将内容准确地添加到下一次的提交中去。当然，如果需要查看尚未暂存的文件哪些地方改变了，可以执行`git diff`命令，如图所示：

![image-20211120201214865](../img/Git%20%E7%AE%80%E4%BB%8B/image-20211120201214865.png)

这个命令可以比较工作区中的当前文件和暂存区域文件之间快照的差异。当然如果需要知道暂存区域的文件和最后一次文件提交之间的差异，可以执行`git diff --staged`命令。

##### 忽略文件

在工作目录中，可以创建一个`.gitignore`文件用来告诉Git这些文件不需要被纳入管理，也不需要被跟踪。

文件 `.gitignore` 的格式规范如下：

- 所有空行或者以 `#` 开头的行都会被 Git 忽略。
- 可以使用标准的 glob 模式匹配，它会递归地应用在整个工作区中。
- 匹配模式可以以（`/`）开头防止递归。
- 匹配模式可以以（`/`）结尾指定目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上叹号（`!`）取反。

看一个简单的例子：

```
# 忽略所有.txt文件
*.txt
# 跟踪test.txt文件，虽然你忽略了所有的txt文件
！test.txt
# 忽略该目录下的文件
pkg/opcua/opcua
pkg/modbus/modbus
pkg/bluetooth/bluetooth
.idea
vendor/
```

##### 提交更新

`git commit`命令将提交所有放在暂存区域的快照，如果文件没有通过`git add`添加进暂存区，将保持已修改的状态，可以在下次提交的时候纳入版本管理。可以使用`git commit -m`来增加提交信息，也可以使用`git commit -a -m `来跳过使用暂存区域，也就是不需要执行`git add`，这将会将所有已跟踪的文件暂存起来一并提交。

> 注意：每一次的提交操作都会对项目做一次快照，也就是以后可以回到这个状态，或者与其他的状态进行比较。

##### 删除和移动文件

在Git中删除文件只需执行`git rm`命令，分为两种情况：

```shell
# 删除工作区文件并且删除缓存区文件
$ git rm [file1] [file2] ...
# 停止追踪指定文件，但该文件会保留在工作区
$ git rm --cached [file]
```

在Git中移动或者重命名一个文件需要执行`git mv file_from file_to`。事实上，这相当于执行了三条指令，分别是：

```shell
$ mv file_from file_to
$ git rm file_from
$ git add file_to
```

#### Git 查看提交历史

使用`git log`命令可以查看Git的提交历史信息，比如：

![image-20211120222647039](../img/Git%20%E7%AE%80%E4%BB%8B/image-20211120222647039.png)

直接使用该命令并且不添加任何参数，则会按时间的先后顺序一次输出该项目所有的提交历史，时间越新的排在越上面，包括每个提交的SHA-A校验和，作者的名字和电子邮件地址。当然，`git log`命令提供了许多的选项，使用`git log --pretty=format`可以定制记录的显示格式，format常用的选项如下表格所示：

| 选项  | 说明                                          |
| :---- | :-------------------------------------------- |
| `%H`  | 提交的完整哈希值                              |
| `%h`  | 提交的简写哈希值                              |
| `%T`  | 树的完整哈希值                                |
| `%t`  | 树的简写哈希值                                |
| `%P`  | 父提交的完整哈希值                            |
| `%p`  | 父提交的简写哈希值                            |
| `%an` | 作者名字                                      |
| `%ae` | 作者的电子邮件地址                            |
| `%ad` | 作者修订日期（可以用 --date=选项 来定制格式） |
| `%ar` | 作者修订日期，按多久以前的方式显示            |
| `%cn` | 提交者的名字                                  |
| `%ce` | 提交者的电子邮件地址                          |
| `%cd` | 提交日期                                      |
| `%cr` | 提交日期（距今多长时间）                      |
| `%s`  | 提交说明                                      |

`git log`的常用选项：

| 选项              | 说明                                                         |
| :---------------- | :----------------------------------------------------------- |
| `-p`              | 按补丁格式显示每个提交引入的差异。                           |
| `--stat`          | 显示每次提交的文件修改统计信息。                             |
| `--shortstat`     | 只显示 --stat 中最后的行数修改添加移除统计。                 |
| `--name-only`     | 仅在提交信息后显示已修改的文件清单。                         |
| `--name-status`   | 显示新增、修改、删除的文件清单。                             |
| `--abbrev-commit` | 仅显示 SHA-1 校验和所有 40 个字符中的前几个字符。            |
| `--relative-date` | 使用较短的相对时间而不是完整格式显示日期（比如“2 weeks ago”）。 |
| `--graph`         | 在日志旁以 ASCII 图形显示分支与合并历史。                    |
| `--pretty`        | 使用其他格式显示历史提交信息。可用的选项包括 oneline、short、full、fuller 和 format（用来定义自己的格式）。 |
| `--oneline`       | `--pretty=oneline --abbrev-commit` 合用的简写。              |

限制`git log`输出的选项：

| 选项                  | 说明                                       |
| :-------------------- | :----------------------------------------- |
| `-<n>`                | 仅显示最近的 n 条提交。                    |
| `--since`, `--after`  | 仅显示指定时间之后的提交。                 |
| `--until`, `--before` | 仅显示指定时间之前的提交。                 |
| `--author`            | 仅显示作者匹配指定字符串的提交。           |
| `--committer`         | 仅显示提交者匹配指定字符串的提交。         |
| `--grep`              | 仅显示提交说明中包含指定字符串的提交。     |
| `-S`                  | 仅显示添加或删除内容匹配指定字符串的提交。 |

#### Git 撤销操作

##### 撤销提交

使用`git commit --amend`来重新提交，这个操作将会覆盖掉原来的提交信息。例如：

```shell
$ git commit -m "initial commit"
$ git add forgotten_file
$ git commit --amend
```

这个操作怎么理解：与其说是修改原来的提交，不如理解为用新的提交去覆盖原来的提交，旧的提交将不会出现在git的版本历史中。

##### 撤销暂存的文件

使用`git reset HEAD <file> ...`命令可以取消文件的暂存，注意，这是个危险的命令，如果加上了`--hard`则会更危险，如果工作区的文件已经修改，执行`git reset --hard`命令将会重置暂存区与工作区，而保持与上一次commit一致。

##### 撤销对文件的修改

使用`git checkout <file>`将会恢复暂存区的指定文件到工作区，这同样也是一个危险的命令，因为你的本地修改都会被暂存区的文件所覆盖。

> 这边需要注意的是，需要注意保护本地工作区的文件，如果自己不清楚修改了哪些文件，不用轻易使用`git reset`或者`git checkout`命令，因为这将可能造成工作区文件修改的丢失，而向上提交的操作可以随时进行，因为在Git中任何已经提交的东西都是可以恢复的，甚至是那些被删除的分支中的提交或者使用了前面所提到的`--amend`选项覆盖的提交。

#### Git 远程仓库的使用

| 功能                                                         | 命令                               |
| ------------------------------------------------------------ | ---------------------------------- |
| 查看远程仓库[显示对应的URL]                                  | `git remote [-v]`                  |
| 添加远程仓库                                                 | `git remote add <shortname> <url>` |
| 从远程仓库拉取数据(`git fentch`只会将数据下载到你的本地仓库——它并不会自动合并或修改你当前的工作) | `git fentch <remote>`              |
| 推送到远程仓库                                               | `git push <remote> <branch>`       |
| 查看某个远程仓库                                             | `git remote show <remote>`         |
| 远程仓库的重命名                                             | `git remote rename <from> <to>`    |
| 远程仓库的移除                                               | `git remote remove <remote>`       |

#### Git 标签

##### 查看以及创建标签

使用`git tag `可以查看已有的标签，如果加上`-l`或者`--list`选项，那么就可以按照特定的模式查找标签。

Git支持两种标签，分别是轻量标签(lightweight)和附注标签(annotated)，轻量标签是某个特定提交的引用，而附注标签是存储在Git数据库中的一个完整对象，它可以被校验，其中包含打标签者的名字、电子邮件地址、日期时间以及一个可以使用GNU签名并验证的标签信息。

创建附注标签：例如`git tag -a v1.0 -m "my version 1.0"`，其中`-m`指定了一条将会存储在标签中的信息。通过`git show v1.0`命令可以看到标签信息和与之对应的提交信息。

创建轻量标签：例如`git tag v1.0-dev`，之后使用`git tag`命令就可以查询到这个标签，但是使用`git show v1.0-dev`将不会显示额外的标签信息，只会显示提交信息。

当前你也可以为过去的提交打标签，只需要知道过去提交的校验和(可以使用`git log --pretty=oneline`查询得到)，如图：

![image-20211121214819967](../img/Git%20%E7%AE%80%E4%BB%8B/image-20211121214819967.png)

通过`git tag -a v1.0-dev fe4ee36`，这样就可以为之前的提交打上标签了。

##### 共享标签

默认情况下，Git将不会在使用`git push`命令时将标签信息推送到远程服务器上，必须显示推送，使用`git push origin v1.0`推送某个标签或者`git push origin --tags`推送所有标签，这样当其他人拉取仓库的时候，也可以得到这些标签。

##### 删除标签

使用`git tag -d v1.0-dev`可以删除你本地仓库上的标签，如果需要移除远程仓库的这个标签，有两种方式：

第一种：`git push <remote> :refs/tags/<tagname>`，这就表示将冒号前面的空值推送到远程标签名，从而达到删除的目的。

第二种：比较直接，使用`git push origin --delete <tagname>`。

##### 检出标签

在标签创建完成后，你在上线之前又做过多次修改。这时你可以检出你之前那个准备发布的版本(打标签的版本)进行部署。
Git中不能真的检出一个标签，因为他们并不能像分支一样来回移动。如果想要工作目录与仓库中特定地标签版本完全一致，可以使用`git checkout -b [分支名] [标签名]`在特定地标签上创建一个新分支。例如：

```shell
$ git checkout -b version1 v1.0-dev
Switched to a new branch 'version1'
```

### Git 高阶技能

#### Git 分支



### 附录：Git 常用命令

#### 仓库

```shell
# 在当前目录新建一个Git代码库
$ git init

# 新建一个目录，将其初始化为Git代码库
$ git init [project-name]

# 下载一个项目和它的整个代码历史
$ git clone [url]
```

#### 配置

```shell
# 显示当前的Git配置
$ git config --list

# 编辑Git配置文件
$ git config -e [--global]

# 设置提交代码时的用户信息
$ git config [--global] user.name "[name]"
$ git config [--global] user.email "[email address]"
```

#### 增加/删除文件

```shell
# 添加指定文件到暂存区
$ git add [file1] [file2] ...

# 添加指定目录到暂存区，包括子目录
$ git add [dir]

# 添加当前目录的所有文件到暂存区
$ git add .

# 添加每个变化前，都会要求确认
# 对于同一个文件的多处变化，可以实现分次提交
$ git add -p

# 删除工作区文件，并且将这次删除放入暂存区
$ git rm [file1] [file2] ...

# 停止追踪指定文件，但该文件会保留在工作区
$ git rm --cached [file]

# 改名文件，并且将这个改名放入暂存区
$ git mv [file-original] [file-renamed]
```

#### 代码提交

```shell
# 提交暂存区到仓库区
$ git commit -m [message]

# 提交暂存区的指定文件到仓库区
$ git commit [file1] [file2] ... -m [message]

# 提交工作区自上次commit之后的变化，直接到仓库区
$ git commit -a

# 提交时显示所有diff信息
$ git commit -v

# 使用一次新的commit，替代上一次提交
# 如果代码没有任何新变化，则用来改写上一次commit的提交信息
$ git commit --amend -m [message]

# 重做上一次commit，并包括指定文件的新变化
$ git commit --amend [file1] [file2] ...
```

#### 分支

```shell
# 列出所有本地分支
$ git branch

# 列出所有远程分支
$ git branch -r

# 列出所有本地分支和远程分支
$ git branch -a

# 新建一个分支，但依然停留在当前分支
$ git branch [branch-name]

# 新建一个分支，并切换到该分支
$ git checkout -b [branch]

# 新建一个分支，指向指定commit
$ git branch [branch] [commit]

# 新建一个分支，与指定的远程分支建立追踪关系
$ git branch --track [branch] [remote-branch]

# 切换到指定分支，并更新工作区
$ git checkout [branch-name]

# 切换到上一个分支
$ git checkout -

# 建立追踪关系，在现有分支与指定的远程分支之间
$ git branch --set-upstream [branch] [remote-branch]

# 合并指定分支到当前分支
$ git merge [branch]

# 选择一个commit，合并进当前分支
$ git cherry-pick [commit]

# 删除分支
$ git branch -d [branch-name]

# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]
```

#### 标签

```shell
# 列出所有tag
$ git tag

# 新建一个tag在当前commit
$ git tag [tag]

# 新建一个tag在指定commit
$ git tag [tag] [commit]

# 删除本地tag
$ git tag -d [tag]

# 删除远程tag
$ git push origin :refs/tags/[tagName]

# 查看tag信息
$ git show [tag]

# 提交指定tag
$ git push [remote] [tag]

# 提交所有tag
$ git push [remote] --tags

# 新建一个分支，指向某个tag
$ git checkout -b [branch] [tag]
```

#### 查看信息

```shell
# 显示有变更的文件
$ git status

# 显示当前分支的版本历史
$ git log

# 显示commit历史，以及每次commit发生变更的文件
$ git log --stat

# 搜索提交历史，根据关键词
$ git log -S [keyword]

# 显示某个commit之后的所有变动，每个commit占据一行
$ git log [tag] HEAD --pretty=format:%s

# 显示某个commit之后的所有变动，其"提交说明"必须符合搜索条件
$ git log [tag] HEAD --grep feature

# 显示某个文件的版本历史，包括文件改名
$ git log --follow [file]
$ git whatchanged [file]

# 显示指定文件相关的每一次diff
$ git log -p [file]

# 显示过去5次提交
$ git log -5 --pretty --oneline

# 显示所有提交过的用户，按提交次数排序
$ git shortlog -sn

# 显示指定文件是什么人在什么时间修改过
$ git blame [file]

# 显示暂存区和工作区的差异
$ git diff

# 显示暂存区和上一个commit的差异
$ git diff --cached [file]

# 显示工作区与当前分支最新commit之间的差异
$ git diff HEAD

# 显示两次提交之间的差异
$ git diff [first-branch]...[second-branch]

# 显示今天你写了多少行代码
$ git diff --shortstat "@{0 day ago}"

# 显示某次提交的元数据和内容变化
$ git show [commit]

# 显示某次提交发生变化的文件
$ git show --name-only [commit]

# 显示某次提交时，某个文件的内容
$ git show [commit]:[filename]

# 显示当前分支的最近几次提交
$ git reflog
```

#### 远程同步

```shell
# 下载远程仓库的所有变动
$ git fetch [remote]

# 显示所有远程仓库
$ git remote -v

# 显示某个远程仓库的信息
$ git remote show [remote]

# 增加一个新的远程仓库，并命名
$ git remote add [shortname] [url]

# 取回远程仓库的变化，并与本地分支合并
$ git pull [remote] [branch]

# 上传本地指定分支到远程仓库
$ git push [remote] [branch]

# 强行推送当前分支到远程仓库，即使有冲突
$ git push [remote] --force

# 推送所有分支到远程仓库
$ git push [remote] --all
```

#### 撤销

```shell
# 恢复暂存区的指定文件到工作区
$ git checkout [file]

# 恢复某个commit的指定文件到暂存区和工作区
$ git checkout [commit] [file]

# 恢复暂存区的所有文件到工作区
$ git checkout .

# 重置暂存区的指定文件，与上一次commit保持一致，但工作区不变
$ git reset [file]

# 重置暂存区与工作区，与上一次commit保持一致
$ git reset --hard

# 重置当前分支的指针为指定commit，同时重置暂存区，但工作区不变
$ git reset [commit]

# 重置当前分支的HEAD为指定commit，同时重置暂存区和工作区，与指定commit一致
$ git reset --hard [commit]

# 重置当前HEAD为指定commit，但保持暂存区和工作区不变
$ git reset --keep [commit]

# 新建一个commit，用来撤销指定commit
# 后者的所有变化都将被前者抵消，并且应用到当前分支
$ git revert [commit]

暂时将未提交的变化移除，稍后再移入
$ git stash
$ git stash pop
```

#### 其他

```shell
# 生成一个可供发布的压缩包
$ git archive
```

### 参考资料

官方：https://www.git-scm.com/book/zh/v2

菜鸟教程：https://www.runoob.com/git/git-tutorial.html