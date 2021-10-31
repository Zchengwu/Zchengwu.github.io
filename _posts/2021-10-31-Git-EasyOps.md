# Git简单操作

## 常用命令

- `git init`
    
    > 在当前文件夹下建立git仓库
    > 
- `git add`
    
    > 开始跟踪新文件 / 把已跟踪的文件放到暂存 / 还能用于合并时把有冲突的文件标记为已解决状态
    > 
    - `git add`
    - `git add <file>`
- `git commit`
    
    > 提交暂存区内的修改
    > 
    - `git commit -m ""`， `m` 参数是本次提交的说明
    - `git commit —amend` 启动文本编辑器，修改之前的commit信息
- `git status`
    
    > 查看被追踪的文件的状态
    > 
    - `-s` 精简log
- `git diff`
    
    > 查看被追踪文件的具体修改
    > 
    - `git diff` 查看没有进入暂存的具体修改
    - `git diff --staged` 查看已暂存和最后一次commit之间的修改
- `git rm`
    
    > 移除文件
    > 
    - `git rm [-f] <file>`  从暂存区移除，并从工作目录中删除文件。在执行过add后想删除加`-f`
    - `git rm --cached <file>` 从暂存区移除，但并不从工作目录中删除文件。后续可以在`.gitignore`上补上
    - `git rm log/\*.log` 可以用`glob`模式，但是git有自己的文件名扩展匹配方式，与shell不是同一套。例如这里的`\*`
- `git mv`
    
    > 修改文件名 / 移动文件
    > 
    - `git mv <file_from> <file_to>`
    - 等价于
        
        ```bash
        $ mv README.md README
        $ git rm README.md
        $ git add README
        ```
        
- `git log`
    
    > 查看commit历史
    > 
    - `--pretty=oneline` 以一种好看的形式呈现
    - `-p` / `-patch` 显示每次提交的代码差异
    - `-2` 显示最近两次提交
- `git reset`
    
    > 将文件从暂存中撤销 / 版本回退
    > 
    - `git reset HEAD <file>` 将文件中暂存中撤销，但文件修改依然存在
    - `git reset —hard HEAD^` 版本回退到上一个commit
    - `git reset —hard 52cec` 版本回退到hash指定的特殊commit，例如这里52cec指定的commit
- `git checkout`
    
    > 撤销对文件的修改
    > 
    - `git checkout —- <file>` 不在暂存区的文件，可以通过这个指令撤销对文件的修改。
- `git reflog`
    
    > 显示出所有你之前git操作，记住想要恢复的commit hash，通过git reset恢复
    > 
- `git cherry-pick`
    
    > 如果在错误的版本回滚后，又新的commit，在恢复错误回滚后想同时恢复新的commit，可以用这个指令。
    > 

## Git设置

- 工具：`git config`
- `/etc/gitconfig` :  包含系统上每一个用户及他们仓库的通用配置。 如果在执行 `git config` 时带上 `-system` 选项，那么它就会读写该文件中的配置变量。
- `~/.gitconfig` 或 `~/.config/git/config` ：只针对当前用户。 你可以传递 `-global` 选项让 Git 读写此文件，这会对你系统上**所有**的仓库生效。
- `<repo>/.git/config` ：针对该仓库。 你可以传递 `-local` 选项让 Git 强制读写此文件，虽然默认情况下用的就是它（当然，你需要进入某个 Git 仓库中才能让该选项生效。）

```bash
$ git config --global user.name "Ricky Wu"
$ git config --global user.email 1234@xxx.com
```

## Git**远程仓库**

- `git remote`
    
    > 查看当前本地仓库是否存在对应的远程仓库
    > 
    - `git remote -v` 远程仓库对应的简写与其URL。
- `git remote rename <old_name> <new_name>`
    
    > 远程仓库简写重命名
    > 
- `git remote remove <shortname>`
    
    > 移除一个简写名对应的远程仓库
    > 

### git clone第一次提交代码到远程仓库

```bash
# git clone 命令会自动设置本地 master 分支跟踪克隆的远程仓库的 master 分支
$ git clone https://github.com/schacon/ticgit
# 查看本地库修改状态
$ git status                  
# 添加修改的文件进入版本库
$ git add.                    
# 提交 commit
$ git commit -m "xxx"         
# 将本地master分支的commit推送到远程仓库
$ git push origin master
```

### 非git clone第一次提交代码到远程仓库

```bash
# 初始化本地git仓库
$ git init                     
(# 查看远程仓库)
($ git remote -v)
(# 删除原有的仓库)
($ git remote rm <shortname>)
# 本地仓库连接到远程仓库，远程库名字就叫origin
$ git remote add <shortname> <remote url>

# 切换到master分支关联并拉取
$ git checkout master && git pull

# 修改并上传
$ git status                  
# 添加修改的文件进入版本库
$ git add.                    
# 提交 commit
$ git commit -m "xxx"         
# 将本地master分支的commit推送到远程仓库
$ git push origin master
```

### **从远程仓库中抓取的两种方式不同之处**

```bash
# 访问远程仓库，拉取本地仓库还没有的数据。只下载到本地仓库，不合并或修改本地仓库当前状态
$ git fetch <remote> 
## git clone后远程仓库的默认简写为"origin"，因此多用下面这个用法
$ git fetch origin

# 自动抓取后合并该远程分支到**当前分支**
$ git pull
```

## Git 分支

- `git branch`
    
    > 创建分支
    > 
    - `git branch <branchname>` 创建分支
    - `git branch -d <branchname>` 删除分支

### 切换分支

- `git checkout`
    
    > 切换分支
    > 
    - `git checkout <branchname>` 切换分支
    - `git checkout -b <branchname>` 创建并切换分支

### 分支合并

- `git merge`
    
    > 合并分支
    > 
    - `git merge <branch-A>` 合并`branch-A` 到本分支

### 关联本地分支和远程分支

- 远程分支不存在
    
    ```bash
    # 创建并切换到远程分支
    $ git checkout -b <new-branch>
    # 将分支推送到远程
    $ git push origin <new-branch>  
    # 查看一下分支有没有关联好
    $ git branch -a
    # 若果没有，本地分支new-branch关联到远程分支new-branch上
    $ git branch --set-upstream-to=origin/new-branch
    ```
    
- 远程分支存在
    
    ```bash
    $ git checkout -b <local branch name> origin/<remote branch name>
    ```
    

## .gitignore

- 所有空行或者以 `#` 开头的行都会被 Git 忽略。
- 可以使用标准的 glob 模式匹配，它会递归地应用在整个工作区中。
    
    > 用来匹配零个或多个字符，如`.[oa]`忽略所有以".o"或".a"结尾，`~`忽略所有以`~`结尾的文件（这种文件通常被许多编辑器标记为临时文件）；`[]`用来匹配括号内的任一字符，如`[abc]`，也可以在括号内加连接符，如`[0-9]`匹配0至9的数；`?`用来匹配单个字符。
    > 
- 匹配模式可以以（`/`）开头防止递归。
- 匹配模式可以以（`/`）结尾指定目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上叹号（`!`）取反。

## commit规范

> 在最近的项目中有看到学长使用，学习记录一下
> 
> 
> 模式：`type(scope) : subject`
> 
> 1.  type（必须） : commit 的类别，只允许使用下面几个标识：
>     - feat : 新功能
>     - fix : 修复bug
>     - docs : 文档改变
>     - style : 代码格式改变
>     - refactor : 某个已有功能重构
>     - perf : 性能优化
>     - test : 增加测试
>     - build : 改变了build工具 如 grunt换成了 npm
>     - revert : 撤销上一次的 commit
>     - chore : 构建过程或辅助工具的变动
> 2.  scope（可选） : 用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。
> 3.  subject（必须） : commit 的简短描述，不超过50个字符。commitizen 是一个撰写合格 Commit message 的工具，遵循 Angular 的提交规范。

---

## Ref

- [**Git Book**](https://git-scm.com/book/en/v2)
- **[Git中.gitignore的配置语法](https://www.jianshu.com/p/ea6341224e89)**
- **[git从已有分支拉新分支开发](https://blog.csdn.net/QH_JAVA/article/details/77760499)**