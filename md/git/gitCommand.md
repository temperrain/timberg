> Windows环境 右键 Git Bash Here就是我们的客户端

##### 初始化安装
设置用户名称与邮件地址。会写入到Git的提交中，不可更改：
> git config --global user.name "timberg"

> git config --global user.email "timberg@163.com"

*--global 全局安装  如果想在特定项目使用别的名称和邮箱 可在相应目录下直接运行没有此选项的命令*

##### 初始化仓库

在任意目录下，比如 D:\gitDemo  执行:
> git init


##### 添加文件到暂存区

> git add readme.txt 提交指定文件

> git add .    可提交当前目录下所有文件


###### 提交文件到仓库

> git commit -m "first commit"

*参数是用来注释你提交的信息的，这样以后才知道这次提交时用来干嘛*

##### 查看新增或修改的文件
> git status 

> git status -s  精简显示

##### 查看上次修改的内容
> git diff readme.txt

##### 查看历史提交记录
> git log   

> git log --pretty=oneline 或 git log --oneline  精简显示

> git log --decorate  查看各个分支当前所指的对象

> git reflog   查看各个版本切换的历史记录

##### 撤销操作
> git checkout --[file]      git checkout -- readme.txt   场景：文件在工作区修改，未提交到暂存区 意思：暂存区覆盖工作区

> git reset HEAD  场景：文件修改并commit提交到暂存区  意思：让HEAD覆盖暂存区 

> git checkout HEAD [file]  命令是：git checkout --[file] 和 git reset HEAD的合成体 意思：用HEAD 直接覆盖暂存区和工作区

##### 版本回退和切换
> git reset --hard HEAD^   参数：HEAD^ 指上一个版本  HEAD^^ 指上上个版本 HEAD~100 上100个版本

> git reset --hard 版本号  直接切换到对应版本 

##### 删除操作
> git rm   意思：同时删除工作区跟暂存区中的指定文件

##### 查看远程库的信息
> git remote        详细信息： git remote -v

##### 连接远程仓库
1.  > git init        初始化本地仓库


2.   > git add readme.txt        提交代码到本地仓库
     
     > git commit -m "first commit"  

3.   > git remote add origin http://localhost:3000/timberg/Git.git     连接上远程仓库
4.   > git push -u origin master             推送内容到远程仓库
     
     > git push -u origin dev     推送别的分支到远程仓库

```
注：git push 出现警告

The file will have its original line endings in your working directory.
warning: LF will be replaced by CRLF in web/resources/ueditor/ueditor.config.js.

CRLF, LF 都是用来表示文本换行的方式

#提交时转换为LF，检出时转换为CRLF
git config --global core.autocrlf true
#提交时转换为LF，检出时不转换
git config --global core.autocrlf input
#提交检出均不转换
git config --global core.autocrlf false
使用如下命令：
>  git rm -r --cached  .
>  git config core.autocrlf false
>  git add .
```



##### 从远程仓库下载项目
> git clone http://localhost:3000/timberg/Git.git 

##### 从远端仓库提取数据并尝试合并到当前分支
> git pull origin master

##### 删除跟远程仓库的连接
> git remote rm origin

##### 创建分支
> git branch test   

##### 切换分支
> git checkout test

> git checkout -b test    创建分支和切换到当前分支可以合并

##### 查看当前分支
> git branch       查看当前分支
> git branch -v    查看每一个分支的最后一次提交  
> git branch -a    查看本地和远程的分支情况
> git branch -merged   查看已经与当前分支合并的分支
> git branch -no-merged  查看已经与当前分支未合并的分支
> git branch -r     查看远程分支

##### 合并分支
> git merge  test      意思：合并指定分支到当前分支

> git merge --no-ff -m "描述" test    意思：因为本次合并要创建一个新的commit，所以加上-m参数，把commit描述写进去

##### 删除分支
> git branch -d test
