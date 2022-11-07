## git常用命令
### 常用
- git add .把所有更改的文件添加到暂存区
- git restore . 把修改的文件恢复
- git commit -m '你的提交信息'把暂存区的文件提交到本地仓库
- git push origin 当前分支把本地仓库推送到远程仓库
- git pull 远程分支名称 把远程仓库的数据拉下来，并合并（也可以忽略分支名称，那就会把远程仓库的所有拉到本地仓库）
- git fetch 远程分支名称 把远程仓库的数据拉下来，但是不合并
- git merge 分支 把当前分支和指定分支进行合并
- git status 查看状态

### 提交
- git rebase -i 提交名称 将除了指定提交前的所有提交进行处理，可以进行合并
- git show 提交id 查看某次提交的修改内容

### 分支
- git checkout 某分支从当前分支切换到某分支
- git checkout -b 新分支 origin/分支基于已有分支代码创建新的分支
- git push origin 新分支把远程分支的代码推送到新分支上
- git branch 查看本地分支
- git branch -r 查看远程分支
- git branch 新分支创建一个分支
- git checkout -b 新分支切换到新分支上
- git branch -d 本地分支删除本地分支
- git push origin --delete 远程分支删除远程分支
### 回滚
- git reset --hard 提交回滚至此提交

### 解决冲突
- git log 分支 分支 查看分支之间的区别
- git pull origin 分支将冲突的分支拉下来
此时就可以看到冲突的地方,在冲突的地方进行合并之后
- git add .将修改的代码提交到暂存区
- git commit -m ""将暂存区的代码提交到本地仓库
当推到远程仓库的时候，若此提交低于仓库提交，可以强制性推上去，此时会删除掉远程仓库的最新提交
- git push origin 分支 --force将该分支强制性推上去

### 用户相关的
- git config --list 查看配置
- git config --global user.name(email) 查看全局的用户名或者邮箱
- git config --global --replace-all user.name "xxx" 修改全局的用户名或者邮箱

### git stash
**使用场景**

a.发现一个类是多余的，删掉之后又怕以后需要查看它的代码，想保存它但有不想增加一个脏的提交。这个时候，就可以考虑git stash

b.一般使用分支解决任务切换问题时，会建一个新的分支去调试代码，如果发现自己原有分支上有不得不修改的bug，就必须把新分支上修改的代码commit到本地仓库，然后切换到原分支上去修改，改完之后，再切换回来，那这样log上会有大量不必要的记录。因此，就可以使用git stash将当前未提交到本地的代码推入Git的栈中，这时候工作区间和上一次提交的内容是完全一样的，现在就可以去修复bug，等修完之后，再切换回来到这个分支，使用git apply 将以前一半的工作应用回来。

**git stash用法**

1.stash当前修改

git stash会把当前未提交的修改都保存起来，用于后续恢复当前工作目录。
实际应用中推荐给每个stash加一个message，用于记录版本，使用git stash save取代git stash命令

2.重新应用缓存的stash

可以通过git stash pop命令恢复之前缓存的工作目录，这个指令将缓存栈中的第一个stash删除，并将对应修改应用到当前的工作目录下。
也可以使用git apply命令，将缓存栈中的stash多次应用到工作目录中，但并不删除stash拷贝

3.查看现有stash

可以使用git stash list命令
在使用git stash apply命令可以通过名字指定使用哪个stash，默认使用最近的stash(即stash@{0})

4.移除stash

可以使用git stash drop命令，后面可以跟着stash名字。
或者git stash clear命令，删除所有缓存的stash

5.查看指定stash的diff

可以使用git stash show命令，后面可以跟着stash命令
在该命令命令添加-p或--patch可以查看特定stash的全部diff

6.暂存范围

默认情况下，git stash会缓存下列文件
- 添加到暂存区的修改
- Git跟踪的但并未添加到暂存区的修改
