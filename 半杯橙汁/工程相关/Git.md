# Git

## 基础配置config

```bash
$git --version # 查看版本信息
$git config --global user.name "Orange Han" 	# 设置仓库的全局用户名
$git config --global user.email 3213213213		# 设置邮箱
$git config --global credential.helper store	#用户名和邮箱自动存储
$git config --global --list					   #显示用户信息
```

## 新建仓库init&clone

```bash
git init repositoryName # 创建本地仓库，即创建一个repositoryName目录，里面有.git目录表示是一个仓库
git clone url	# 从url克隆仓库到本地
```

## 工作区域和文件状态

**工作区域：**

+ 工作区（.git）:就是我们使用的区域，即本地目录
+ 暂存区（.git/index）：存储仓库的变化信息
+ 本地仓库（.git/objects）:存储代码的主要位置

> 一般流程：修改代码后通过git add将修改代码添加至暂存区，再通过git commit提交至仓库

**文件状态：**

+ 未跟踪(Untrack)：新建文件，还未被git管理
+ 未修改(Unmodified)：已经被git管理，文件内容没有变化
+ 已修改(Modified)：已经被git管理，存在修改，但是还未添加至暂存区
+ 已暂存(Staged)：更新内容已经放在暂存区

相关命令：

+ 未跟踪文件可以通过git add将文件直接加入暂存区，变为已暂存状态
+ 已修改文件也可以通过git add将文件放入暂存区
+ 已暂存文件通过git commit提交至仓库后，意味着文件状态统一，所以变为未修改转态
+ 已暂存文件也可以通过git reset命令退回至已修改的状态
+ 已修改文件可以使用git checkout命令退回至未修改状态
+ 未修改文件可以通过git rm从git管理中删除，从而变成未跟踪状态

## 添加和提交文件add&commit

```bash
git status 			# 查看仓库状态
git add	filename	# 将文件添加到暂存区,文件名支持通配符，也支持目录名称
git commit -m "log info"		# 将暂存区中的文件提交至仓库
git log				# 查看提交日志,--oneline将每条日志以一行的形式展示
```

## 回退版本reset

```bash
# 回退至某个版本，并保存工作区和暂存区的所有修改内容,即仅将仓库的最近一次更新取消
git reset --soft versionID		
# 回退至某个版本，并丢弃工作区和暂存区的所有修改内容，即将上个版本之后的所有操作取消
git reset --hard versionID		
# 回退至某个版本，保存工作区修改内容，丢弃暂存区修改内容，即仓库和暂存区的最近一次更新取消
git reset --mixed versionID		
# 版本ID可以使用git log查看
git ls-files	# 查看暂存区内容

# 如果误操作，同样可以回退
git reflog	# 查看操作历史记录
git reset --hard versionID
```

## 查看差异diff

```bash
# 查看工作区域之间的差异，版本之间的差异，分支之间的差异
git diff			# 默认是工作区和暂存区之间的差异
git diff HEAD		# 工作区和版本库之间的差异
git diff --cache	# 暂存区和版本库之间的差异
git diff ID1 ID2	# 两个版本之间的差异
git diff HEAD~ HEAD # 比较上个版本和当前版本的差异
git diff HEAD~2 HEAD# 比较前两个版本和当前版本的差异
git diff ID1 ID2 filename # 仅查看两个版本中特定文件的差异
git diff branchName1 branchName2 # 查看两个分支之间的差异
```

## 删除文件rm

```bash
# 如果在本地使用了rm操作，也需要使用git add更新暂存区的状态
rm file
git add file
git commit -m "delete file"
# 或者直接使用git rm
git rm file 
git commit -m "delete file"
# 其他形式
git rm --cached file	# 删除暂存区文件/删除版本库中的文件
git rm -r * 			# 递归删除目录下的所有内容
```

## .gitignore文件

可以在.gitignore中写入文件名，git就会忽略管理这些文件，文件名也支持通配符。

需要注意的是，已经被管理的文件不会被影响。 

需要git忽略的文件：

+ 系统或者软件自动生成的文件
+ 编译产生的中间文件和结果文件
+ 运行时生成的日志文件、缓存文件、临时文件
+ 敏感信息文件

> 常见语言的忽略文件模板可以在官方文档中查询

## SSH配置和克隆仓库

```bash
cd ~/.ssh
ssh-kengen -t rsa -b 4096
# 将公钥复制到github即可
```

注意，克隆后的本地仓库和远程仓库是独立的仓库。他们之间的同步通过git push和git pull来操作

```bash
git push <remote><branch>	# 推送更新内容
git pull <remote>			# 拉取更新内容
```

## 关联本地和远程仓库

首先需要创建一个空的远程仓库，github会提供下述语句：

> 需要注意的是，本地必须有一次提交，这样才会创建默认分支

```bash
git remote add <shortname><url>	# git remote add origin git@github.com:OrangeHan422/responseName.git
git branch -M main		# 将本地分支命名为main
git push -u origin main:main	#将本地的main和远程的origin关联起来
```

从远程仓库中获取更新

```bash
# 会自动执行一次合并，需要手动处理冲突，如果分支名相同，可以忽略：以及：后面的内容
git pull <remote-respo><remote_branch>:<local_branch>	
git fetch	# 仅会将远程的修改拿回来，需要手动合并
```