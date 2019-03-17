---
title: Gitflow工作流：一个健壮的用于管理大型项目的框架
date: 2017-12-23 0:12:32
tags:
- Gitflow
- Git
- Git Flow Workflow
categories:
- 源码管理
---

# Gitflow工作流介绍

> Gitflow工作流仍然用中央仓库作为所有开发者的交互中心，这和其它git的工作流一样，开发者在本地开发并push分支到要中央仓库中。Gitflow工作流定义了一个围绕项目发布的严格分支模型，它提供了用于一个健壮的用于管理大型项目的框架。

![](/images/git-workflow-gitflow.png)
<!-- more -->
Gitflow工作流没有用超出**功能分支工作流**的概念和命令，它为不同的分支分配一个很明确的角色，并定义分支之间在什么时候怎样进行交互。下面对各个分支的作用进行说明，也有一个例子用来介绍分支之间具体如何交互。

## 历史分支

Gitflow工作流使用2个分支来记录项目的历史。master分支存储了正式发布的历史，而develop分支作为功能的集成分支。这样也方便master分支上的所有提交分配一个版本号。

![](/images/git-workflow-release-cycle-1historical.png)

## 功能分支

每个新功能都有一个自己的专属分支，这样可以push到中央仓库备份和协作。但功能分支并不是基于master而创建的分支，而是使用develop分支作为父分支。当新功能完成时需要合并回develop分支，它不应该直接与master交互。

![](/images/git-workflow-release-cycle-2feature.png)

最佳实践：

* 基于 `develop` 发展
* 必须合并回到 `develop`
* 分支命名规范 `master`, `develop`, `release-*`, or `hotfix-*`

## 发布分支

当develop分支上有做了一次合并或者是积攒了足够功能，就从develop分支上fork一个发布分支。新建的这个分支用于发布代码，所以从这个时间点开始之后新的功能不能再加到这个分支上，这个分支只能修改Bug和其它面向发布的任务。一旦对外发布的工作都完成了，发布分支合并到master分支并分配一个版本号打好Tag。另外，这些从新建发布分支以来的做的修改要合并回develop分支。

![](/images/git-workflow-release-cycle-3release.png)

使用一个用于发布准备的专门分支，使得一个团队可以在完善当前的发布版本的同时，另一个团队可以继续开发下个版本的功能。

这也制定了一个良好的开发阶段（比如，可以很轻松地说『这周我们要准备发布版本4.0』），并且在仓库的目录结构中可以实际看到。

## 维护分支

维护分支或说是热修复（hotfix）分支用于生成快速给产品发布版本（production releases）打补丁，这是唯一可以直接从master分支fork出来的分支。修复完成，修改应该马上合并回master分支和develop分支（当前的发布分支），master分支应该用新的版本号打好Tag。

![](/images/git-workflow-release-cycle-4maintenance.png)

为Bug修复使用专门分支，让团队可以处理掉问题而不用打断其它工作或是等待下一个发布循环。你可以把维护分支想成是一个直接在master分支上处理的临时发布。

# 例子

> 下面的示例演示本工作流如何用于管理单个发布循环。假设你已经创建了一个中央仓库，我们模拟几位同事运用gitflow工作流协作。

## 创建开发分支

![](/images/git-workflow-release-cycle-5createdev.png)

基于master分支创建一个develop分支。可以先在本地创建一个空的develop分支，然后push到服务器上：

```bash
git branch develop
git push -u origin develop
```

以后这个分支将会包含了项目的全部历史，而master分支将只包含了部分历史。其它开发者这时应该克隆中央仓库，建好develop分支的跟踪分支（即本地develop与远程develop进行关联）：

```bash
git clone ssh://user@host/path/to/repo.git
git checkout -b develop origin/develop
```

上面由项目管理者操作即可，而每个开发人员应该从下面的流程开始，我们假充有两位同事：李伟和刘明。

## 李伟和刘明开发各自的新功能

![](/images/git-workflow-release-cycle-6maryjohnbeginnew.png)

这两位同事负责开发独立的功能，首先他们需要为各自的功能创建相应的分支。新分支不是基于master分支，而应该基于develop分支：

```bash
# 李伟
git checkout -b 0.1.0 develop

# 刘明
git checkout -b 0.1.1 develop
```

他们在各自创建的分支上编写代码，在开发过程中陆续有代码提交到本地仓库的功能分支。

```bash
git status
git add --all
git commit -m 'something...'
```

## 李伟完成功能开发

![](/images/git-workflow-release-cycle-7maryfinishes.png)

李伟在开发过程中会有多次提交，当他确认功能开发完毕后可以直接合并到本地的develop分支，并且push到中央仓库。如果团队使用Pull Requests，这时候也可以发起一个用于合并变更到develop分支申请。

```bash
git pull origin develop
git checkout develop
git merge 0.1.0
git push
git branch -d 0.1.0
```

第一条命令确保在合并前develop分支是最新的。要特别注意的是功能分支上的代码决不应该直接合并到master分支。在合并过程中可能会有冲突，解决方法的思路和SVN是类似的。

## 李伟开始准备发布

![](/images/git-workflow-release-cycle-8maryprepsrelease.png)

这个时候刘明还在实现他的功能，李伟已经准备他第一个项目正式发布。像功能开发一样，他用一个新的分支来做发布准备。这一步也确定了发布的版本号：

```bash
git checkout -b release-0.1.0 develop
```

这个分支是清理发布、执行所有测试、更新文档和其它为下个发布做准备操作的地方，像是一个专门用于改善发布的功能分支。只要李伟创建这个分支并push到中央仓库，这个发布就是功能冻结的。任何不在develop分支中的新功能都推到下个发布循环中。

## 李伟完成发布

![](/images/git-workflow-release-cycle-9maryfinishes.png)

一旦准备好了对外发布，李伟合并修改到master分支和develop分支上，并且删除发布分支。合并回develop分支很重要，因为在发布分支中已经提交的更新需要在后面的新功能中也是必须的。另外，如果李伟的团队要求Code Review，这是一个发起Pull Request的理想时机。

```bash
git checkout master
git merge release-0.1.0
git push
git checkout develop
git merge release-0.1.0
git push
git branch -d release-0.1.0
```

发布分支是作为功能开发（develop分支）和对外发布（master分支）间的缓冲。只要有合并到master分支，就应该打好Tag以方便跟踪。

```bash
git tag -a v0.1.0 -m "Initial public release" master
git push --tags
```

Git有提供各种勾子（hook），即仓库有事件发生时触发执行的脚本。可以配置一个勾子，在你push中央仓库的master分支时，自动构建好对外发布。

## 最终用户发现Bug

![](/images/git-workflow-gitflow-enduserbug.png)

版本对外发布后，李伟回去和刘明一起做下个版本的新功能开发，直到有用户开了一个Ticket抱怨当前版本的一个Bug。为了处理Bug，李伟（或刘明）从master分支上拉出了一个维护分支，提交修改以解决问题，然后直接合并回master分支：

```bash
git checkout -b issue-#001 master
# Fix the bug
git checkout master
git merge issue-#001
git push
```

就像发布分支，维护分支中新加这些重要修改需要包含到develop分支中，所以李伟要执行一个合并操作。然后就可以安全地删除这个分支了：

```bash
git checkout develop
git merge issue-#001
git push
git branch -d issue-#001
```

# 实际项目中应用


> 如果你认真阅读完上面的内容，想必对gitflow工作流的实施有疑问，毕竟上面的操作过于复杂，需要开发者和项目管理者主观控制的地方特别多。既然是人为主观控制的流程，那也就表示有不按流程执行的风险，这个流程很可能会流于形式。

为了解决上面的问题，我们尝试着找到一款软件能够帮助开发团队准确、完整的实施gitflow，幸运的是我们找到<a href="https://git.cloud.tencent.com" title="TGit">TGit</a>。这也是我们团队实际在使用的一款软件，它完美的实现了上面的理论。现在内测中，我们用的还不错。

仍然以两位同事李伟和刘明来举例说明整个协作过程：

1. 李伟需要在**项目A**上开发新功能，项目经理为这一期迭代分配的版本是 `0.1.0`，他接到任务后克隆 `git clone git@git.qcloud.com:example/test.git` 出项目代码准备开发。
1. 在本地创建功能分支，也就是要开发的分支： `git checkout -b 0.1.0`
1. 按照开发计划陆续完成各项任务，在这个过程中会有代码提交到本地仓库 `git commit -am '完成xx任务'`。
1. 一段时间之后李伟的开发任务全部完成，此时要将本地仓库中的变更全部摄推送到中央仓库 `git push 0.1.0`。
1. 在TGit系统上创建一条合并请求，将0.1.0分支上的变更合并到develop。
1. 对于开发人员（李伟）提交的合并，会进入代码评审阶段，评审人员由项目初始化时管理员配置。
1. 由指定的评审人员评审通过后，项目维护者可以同意本次合并请求，此致代码合并的操作就完成了。

上面只是简单的演示运用支持Gitflow的工具该如何操作，介绍这个流程的大致情况，并不是TGit实际操作的帮助文档。另外除了TGit之外，其实完整支持工作流的还有gitlab等工具，大家可以去实践。
