## git常用命令
###常用
- git add .把所有更改的文件添加到暂存区
- git commit -m '你的提交信息'把暂存区的文件提交到本地仓库
- git push origin 当前分支把本地仓库推送到远程仓库
- git pull把远程仓库的数据拉下来
- git status查看状态
### 分支
- git checkout 某分支从当前分支切换到某分支
- git checkout -b 新分支 origin/分支基于已有分支代码创建新的分支
- git push origin 新分支把远程分支的代码推送到新分支上
- git branch查看本地分支
- git branch -r查看远程分支
- git branch 新分支创建一个分支
- git checkout -b 新分支切换到新分支上
- git branch -d 本地分支删除本地分支
- git push origin --delete 远程分支删除远程分支
### 回滚
- git reset --hard 提交回滚至此提交

### 解决冲突
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