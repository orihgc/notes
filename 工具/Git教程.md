# 本地仓库
## 创建版本库
在一个空目录或者已经有东西的目录里，使用  
==git init==  
把这个目录变为Git可以管理的仓库  
.git目录是隐藏的，用ls -ah命令可以看见
## 把文件添加到版本库
第一步，用命令git add告诉Git，把文件添加到仓库：  
==git add <file>==  
第二步，用命令git commit告诉Git，把文件提交到仓库：  
==git commit -m <message>==  
-m后面是本次提交的说明
## 版本回退
命令显示从最近到最远的提交日志，以查看提交历史，我们可以看到3次提交  
==git log==  
如果嫌输出信息太多，看得眼花缭乱的，可以试试加上--pretty=oneline参数  
==git log --pretty=oneline==  
在Git中，用HEAD表示当前版本，上一个版本就是HEAD^ ,上上个版本就是HEAD^^，  
往上100个版本就是HEAD^ 100  
==git reset --hard HEAD^==  
此时最新的版本不见了  
==git reset --hard <版本号>==  
此操作需要最新版本的commit id，Git提供了一个git reflog用来记录你的每一次命令  
==git reflog==  
查看命令历史
## 管理修改
==git diff HEAD -- <file>==  
命令可以查看工作区和版本库里面最新版本的区别
## 撤销修改
==git checkout -- <file>==  
此命令将file文件在工作区的修改全部撤销，回到最近一次git commit 或者git add时的状态  
==git reset HEAD <file>==  
不但乱了file文件，还添加到了暂存区，此命令就是将暂存区的修改撤销掉，回退到工作区
## 删除文件
在本地rm <file> 之后，现在有两个选择：  
一个是确实需要删除掉  
==git rm <file>   
git commit -m "remove file"==  
另一种就是删错了  
==git checkout -- <file>==
# 远程仓库
## 建立SSH连接
于你的本地Git仓库和GitHub仓库之间的传输是通过SSH加密的，所以，需要一点设置：  
第一步：创建SSH Key：  
==ssh-keygen -t rsa -C "youremail@example.com"==  
一路回车，如果一切顺利的话，可以在用户主目录里找到.ssh目录，里面有id_rsa和id_rsa.pub两个文件，这两个就是SSH Key的秘钥对，id_rsa是私钥，不能泄露出去，id_rsa.pub是公钥，可以放心地告诉任何人。  
第二步：在Git Hub上“Add SSH Key”
## 添加远程仓库
先在github上创建仓库，然后在本地仓库下运行命令：  
==git remote add origin <SSH address>==  
下一步，就可以把本地库的所有内容推送到远程库上：  
==git push -u origin master==  
git push 实际上是把当前分支master推送到远程  
由于远程库是空的，我们第一次推送master分支时，加上了-u参数，Git不但会把本地的master分支内容推送的远程新的master分支，还会把本地的master分支和远程的master分支关联起来，在以后的推送或者拉取时就可以简化命令。
从现在起，只要本地作了提交，就可以通过命令：  
==git push origin master==  
把本地master分支的最新修改推送至GitHub，现在，你就拥有了真正的分布式版本库
## SSH警告
当你第一次使用Git的clone或者push命令连接GitHub时，会得到一个警告：  
The authenticity of host 'github.com (xx.xx.xx.xx)' can't be established.
RSA key fingerprint is xx.xx.xx.xx.xx.
Are you sure you want to continue connecting (yes/no)?  
这是因为Git使用SSH连接，而SSH连接在第一次验证GitHub服务器的Key时，需要你确认GitHub的Key的指纹信息是否真的来自GitHub的服务器，输入yes回车即可。
## 从远程仓库克隆
==git clone <SSH adress>==  
假设我们从零开发，那么最好的方式是先创建远程库，然后，从远程库克隆。
# 分支管理
## 创建与合并分支
查看分支：  
==git branch==  
创建分支：  
==git branch <name>==  
切换分支：  
==git checkout <name>==  
创建+切换分支：
==git checkout -b dev==  
加上-b参数表示创建并切换，相当于以下两条命令：  
==git branch dev  
git checkout dev==  
先在dev上修改，切换回master，合并dev分支  
==git merge dev==  
合并完之后可以删除dev分支  
==git branch -d dev==
## 解决冲突
解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。  
==git log --graph==  
此命令可以看到分支合并图
## 分支管理策略
合并dev分支，请注意--no-ff参数，表示禁用Fast forward：  
==git merge --no-ff -m "merge with no-ff" dev==  
因为本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去。  
合并分支时，加上--no-ff参数就可以用普通模式合并，合并后的历史有分支，能看出来曾经做过合并，而fast forward合并就看不出来曾经做过合并。
## Bug分支
Git还提供了一个stash功能，可以把当前工作现场“储藏”起来，等以后恢复现场后继续工作：    
==git stash==  
在master分支上修改完以后，回到dev分支继续干活，此时工作区是干净的  
==git stash list==  
恢复工作现场
一种方法：  
==git stash apply==
此方法恢复后，stash不删除，需要以下命令删除  
==git stash drop==  
另一种是用
==git stash pop，恢复的同时删除stash
也可以多次stash，恢复的时候，先用git stash list查看，然后恢复指定的stash
git stash apply stash@{0}
## 强制删除未合并的分支
开发一个新feature，最好新建一个分支；  
如果要丢弃一个没有被合并过的分支，可以通过git branch -D <name>强行删除。
## 推送分支
把该分支上的所有本地提交推送到远程库：  
==git push origin master==
## 抓取分支
抓取远程分支：  
==git pull==  
在本地创建和远程分支对应的分支：  
==git checkout -b branch-name origin/branch-name==
## 多人协作
查看远程仓库的信息：  
==git remote==  
显示更详细的信息：  
==git remote -v==  
多人协作的工作模式通常是这样：  
1 首先，可以试图用==git push origin <branch-name>== 推送自己的修改；  
2 如果推送失败，则因为远程分支比你的本地更新，需要先用==git pull== 试图合并；  
3 如果合并有冲突，则解决冲突，并在本地提交；  
4 没有冲突或者解决掉冲突后，再用==git push origin <branch-name>== 推送就能成功！  
5 如果git pull提示no tracking information，则说明本地分支和远程分支的链接关系没有创建，用命令==git branch --set-upstream-to <branch-name> origin/<branch-name>== 。  
这就是多人协作的工作模式，一旦熟悉了，就非常简单。
## Rebase
==git rebase==  
rebase操作可以把本地未push的分叉提交历史整理成直线；
==git rebase -i <commit id>==  
合并此commit id之前的commit  
==git rebase -i HEAD～5==  
合并前5个分支
# 标签管理
## 创建标签
打一个标签：  
==git tag <name>==  
查看所有标签：  
==git tag==
默认是打在最新提交的commit上，那如何打在指定的历史commit上呢？  
==git log --pretty=oneline --abbrev-commit==  
然后  
==git tag <name> <commit id>==  
查看标签信息：  
==git show <tagname>==  
还可以创建带有说明的标签，用-a指定标签名，-m指定说明文字：  
==git tag -a <name> -m "<message>" <commit id>==  
## 操作标签
删除标签：  
==git tag -d <name>==  
推送某个标签到远程:  
==git push origin <tagname>==  
一次性推送所有标签：  
==git push origin --tags==  
如果标签已经推送到远程，要删除标签，先从本地删除：  
==git tag -d <tagname>==  
然后从远程删除，删除命令也是push：  
==git push origin :refs/tags/v1.0==
