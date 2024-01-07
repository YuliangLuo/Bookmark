# 一、Git基础

## Git的特点
最优的存储能力、非凡的性能、开源的、很容易做备份、支持离线操作、很容易定制工作流程

## Git 使用前的最小配置
```
git config --global user.name "your_name"
git config --global user.email "your_email@domain.com"
```

缺省等同于local，其中local只对某个仓库有效，global对当前用户所有仓库有效，system对系统所有登录的用户有效
```
git config --local
git config --global
git config --system
```

显示config的配置，加--list
```
git config --list --local
git config --list --global
git config --list --system
```

## 建Git仓库
两种场景：
1.把已有的项目代码纳入Git管理
```
cd项目代码所在的文件夹
git init
```
2.新建的项目直接用Git管理
```
cd 某个文件夹
git init your_project #会在当前路径下创建和项目名称同名的文件夹
cd your_project
```

批量提交已提交但有更改的文件
```
git add -u
```

工作区、暂存区所有文件都会被清除
```
git reset --hard
```

回退单个版本及多个版本
```
git reset --hard HEAD^
git reset --hard HEAD~100
```

变更文件名
```
git mv readme readme.md
```

增加到版本库里面
```
git commit -am 'Add test'
git branch -av
```

## 通过git log查看版本演变历史

用户界面图形化
```
git log --all --graph
```

单行方式只看最近提交的n个记录
```
git log --oneline --all -n4
```

用户界面图形图形化与最近提交记录复合
```
git log --oneline --all -n4 --graph
```

浏览器查看提交的log
```
git help --web log
```

通过gitk图形界面查看版本历史
```
gitk
```

## 探秘git文件目录

切换到git文件目录
```
cd .git
```

查看仓库当前指向的分支
```
cat HEAD
```

创建分支
```
git branch name
```

创建并切换分支
```
git checkout -b dev
```

查看所有分支
```
git branch
```

查看分支详情
```
git branch -v
```

切换分支,创建分支
```
git checkout name
```

丢弃工作区的修改（add file之后没有commit file），切回原有分支。
```
git checkout -- file
```

没有commit之前，在版本库中恢复此文件
```
git checkout -- filename
```

查看git文件目录配置信息
```
cd .git/config
```

完整查看分区，最后市仓库存放的对象，为一串字符
```
cd .git
cd refs
cd heads/
cat master
```

查看仓库存放的对象及对象的类型
```
git cat-file -t master的ID
```

打包的文件会放在.git/objects里
```
cd .git/
cd objects
cd e8
git cat-file -t e8(e8的hash)
```

git cat-file -t 指的是哈希值的文件类型
git cat-file -p 指的是哈希值指向的文件内容

## commit、tree和blob三个对象之间的关系
commit->tree->tree->blob

查看文件目录下有没有对象
```
find .git/objects -type f
```

```
echo 'hello,world' > readme
git add doc
git commit -m'Add readme'
```
提交之后objects会生成四个tree，类型分别为tree(文件doc)、blob(文件内容)、tree(文件readme)、commit(提交文本Add raedme)

## 分离头指针情况下的注意事项

错误地指向特定hashnumber，修复
```
git checkout hashnumber
git branch fix_css hashnumber
```

基于特定的分支进行新建和切换
```
git checkout -b fix_readme fix_oldreadme
```

确认是否切换成功
```
cat .git/HEAD
```

HEAD的作用:
1. 指代新分支的最后一次提交
2. 不跟分支挂钩，处于分离头状态时，指代的某个具体的commit上面


# 二、独自使用Git时的常见场景
删除不需要的分支
```
git branch -d hashname
git branch -D hashname
```

修改最新commit的message
```
git commit --amend
```

修改老旧commit的message。变基操作会运用到分离头指针的原理，重新建立头。
```
git rebase -i commit的13位hash值
```

把连续的多个commit整理成1个。找到连续多个commit的父节点，
```
git rebase -i fatherhash
```
文档编辑时中间几个子节点的pick改为s

把间隔的几个commit整理成1个。找到要整理的commit，前缀改为s。
```
git rebase -i fatherhash
```

查看所有的提交记录
```
git log --all --graph
```

比较暂存区和HEAD所含文件的差异
```
git diff --cached
```

比较工作区和暂存区所含文件的差异
```
git diff -- readme.md(具体的文件)
```

让暂存区恢复成和HEAD的一样。暂存区的内容都不想要了。
```
git reset HEAD
```

让工作区的文件恢复为和暂存区一样
```
git checkout -- [filename]
```

取消暂存区部分文件的更改
```
git reset HEAD -- [filename]
```

清除最近的几次提交
```
git reset --hard hash
```

最近提交的commit
```
git log -n8 --all 
```

## 查看不同提交的指定文件的差异
```
git diff temp master
git diff temp master -- index.html(具体的文件)
```

正确删除文件的方法
```
git rm filename
```

## 开发中临时加塞了紧急任务怎么处理。

文件先修改了，stash存放到一个不影响
下一步的工作区中。此时工作区中的信息清空。
```
git stash
git stash list（查看stash中的文件）
git status
```

pop可以将最新提交的内容弹出
```
git stash apply(一是将之前存放的内容弹出来，二是堆栈里面这条信息还会存在)
git stash pop
```

改变文件头指针的指向，再使用pop丢掉提交信息
```
git reset --hard HEAD
```

指定不需要Git管理的文件，在.gitignore文件里面修改，格式为.xx文件后缀
```
.xx
```
## 如何将Git仓库备份到本地
常见的传输协议：
| 常用协议 | 语法格式 | 说明 |
| --- | --- | --- |
| 本地协议（1）| /path/to/repo.git | 哑协议 |
| 本地协议（2）| file///path/to/repo.git | 智能协议 |
| http/https协议 | http://git-server.com:port/path/to/repo.git | 平时接触到的都是智能协议 |
| ssh协议 | user@git-server.com:path/to/repo.git | 工作中最常用的智能协议 |

哑协议与智能协议
直观区别：哑协议传输进度不可见；智能协议传输可见。
传输速度：智能协议比哑协议传输速度快。

将Git仓库备份到本地
1. 远程库
bare不包含工作区，作为远端，以后备份方便一点
```
git clone --bare file:///XX/.git remoteRepoName.git
```
2. 本地仓库
跟远程库建立关联
```
git remote add remoteRepoName file:///XX/remoteRepoName.git
```
本地备份
```
git clone --bare [LocalPath]/.git Substitue.git
```
推送到远程服务器
```
git push --set-upstream remoteRepoName master（分支）
```

# 三、Git与GitHub的简单同步
检查本机存在的密钥
```
ls -al ~/.ssh
```

生成ssh密钥
```
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

查看远程仓库
```
git remote -v
```

添加目录远程仓库目录
```
git remote add [remoteRepoName] [remoteRepo url]
```

提交到远程仓库（空）
```
git remote add origin [remoteRepo url]
git push origin master
git push -u origin master
```

提交到远程仓库（已有内容）
```
git remote add origin [remoteRepo url]
git pull --rebase origin main
git push --set-upstream origin master
```

拉取后推送到远程库
```
git remote add github [url]
git fetch github [url]
git merge -h 
git merge --allow-unrelated-histories github/master
git push github
```

merge（保留旧的分支）还是rebase（合成一条分支）
1. merge
优点：
- 简单易上手
- 保留了提交历史和时间次序
- 保留了分支的结构
缺点：
- 提交历史被大量的merge提交污染了
- 使用git bisect调试变得更困难了
2. rebase
优点：
- 把复杂的历史变成优雅的提交线
- 操作单个提交变得很简单
- 避免了庞大的仓库、海量的分支以及烦人的merge提交
- 线性合并清楚了中间的无用提交
缺点：
- Rebase后feature分支间的上下文模糊了
- 团队里rebasing公共分支是高风险的行为
- 工作变多：feature分支需要经常更新
- Rebasing到远程分支需要force push。最大问题是已经force push，但忘记设置git
push默认值。结果本地远程所有同名的分支都进行了更新，清理起来耗费精力

# 四、Git多人单分支集成协作时的常见场景 

## 不同的人修改了不同文件处理
拷贝远端目录并建立新的目录
```
git clone git@github.com:xxx/xxx.git new
```

新的目录下添加用户名和账号
```
git config --add --local user.name '[username]'
git config --add --local user.email '[useremail]'
```

查询本地配置详情
```
git config --local -l
```

分支切换到对应的远端分支
```
git checkout -b [targetBranch] [originalBranch]
```

同步远端的目录
```
git fetch
```

## 不同的人修改同文件的不同区域处理
同步远端目录及文件，且要与本地文件Merge
```
git pull
```

## 不同的人修改同文件的相同区域处理
```
git pull
```

## 同时变更了文件名和文件内容处理


## 把同一文件改成不同的文件名如何处理

# 五、Git集成使用禁忌
## 禁止向集成分支执行push -f操作
会把之前的变更全删掉

## 禁止向集成分支执行变更历史的操作
公共分支严禁拉到本地做变基处理

# 六、初识GitHub
快速找到感兴趣的开源项目：关键字检索
jekyll-now创建Github Pages

# 七、使用GitHub进行团队协作

## 选择适合团队的工作流
需考虑的因素：
1. 团队人员的组成
2. 研发设计能力
3. 输出产品的特征
4. 项目难易程度

常见的开发流：
主干开发适用于：
1. 开发团队系统涉及和开发能力强。有一套有效的特性切换的实施机制，保证上线后无需修改代码
就能修改系统行为。需要快速迭代，想获得CI/CD所有好处。
2. 组件开发的团队，成员能力强，人员少，沟通顺畅。用户升级组件成本低的环境。

Git Flow适用于：
1. 不具备主干开发能力。有预定的发布周期。需要执行严格的发布流程。

Github Flow适用于:
1. 不具备主干开发能力。随时集成随时发布：分支集成时经过代码评审和自动化测试，就可以立即发布
的应用。

GitLab Flow (带生成分支)适用于：
1. 不具备主干开发能力。无法控制准确的发布时间，但又要求不停地集成。

GitLab Flow（带环境分支）适用于：
1. 不具备主干开发能力。需要逐个通过各个测试环境的验证才能发布。

GitLab Flow（带发布分支）使用于：
1. 不具备主干开发能力。需要对外发布和维护不同版本。

## 挑选合适的分支集成策略
Squash是多个commit合成为一个
rebase则是把多个commit按顺序在master分支上添加

## 启用issue跟踪需求和任务
设置创建提issue模板

## 用Project管理issue
看板

## 项目内部实施code review

## 团队协作时做多分支的集成

## 保证继承的质量
Github Marketplace安装Travis CI和CodeCov

## 把产品包发布到Github上

# 八、GitLab实践
GitLab核心功能：
[https://about.gitlab.com/devops-tools/]
[https://about.gitlab.com/stages-devops-lifecycle/]

Gitlab开源项目：
[https://gitlab.com/gitlab-org]


