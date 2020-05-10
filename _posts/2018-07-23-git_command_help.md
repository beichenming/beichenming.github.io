---
layout: post
title: 那些年冷门但很有用的Git命令
subtitle:   "Git"
date:       2018-07-23 17:00:00
author:     "BCM"
header-img: "../../img/post-bg-unix-linux.jpg"
tags:
    - 工具
---

`原创文章转载请注明出处，谢谢`

---

## 前沿

突发奇想写这篇文章的主要目的是温故知新，我们几乎每天的工作都在和Git打交道（使用SVN的同学请出门左转），其中不乏多数同学像我一样热衷于命令行工具，所以熟练的使用命令行对我们平时的工作效率帮助是巨大的；不过对于到底选用命令行还是GUI工具我的观点一直都是适合自己就好；我只是建议应该或多或少补充一些命令行知识有利于对Git的理解；

接下来我就以平时工作中遇到的一些情况为例，分享一下我对应的处理方法；


#### 临时忽略指定文件

**关于如何用Git忽略一些特定的文件我想大家最常用到的应该就是在.gitignore文件进行配置了，不过一般.gitignore针对的都是一些大家都需要忽略的文件，比如我们常用到的build生成目录，或者类似iOS中Cocospod生成的pods目录等等；它们的特点就是远程分支不需要依赖这些东西，它们属于临时产物可再生成；**

**不过平时开发过程中针对于每个开发者来说，每个人可能都需要修改自己本地的特殊配置环境，比如数据库配置，正式/测试环境的切换等等，这个时候我们往往需要忽略这些改动，以防止不小心将这些临时的改动push到远程；**

```
方法一：

// 忽略指定文件
git update-index --assume-unchanged path
// 恢复忽略的指定文件
git update-index --no-assume-unchanged path

git update-index --assume-unchanged这个命令可以使我们的文件处于暂时被忽略的状态，这个时候即使执行git status，git依然会告诉你没有任何文件有改动；但是该命令其实只是假设文件没有改动，如果你执行reset命令的话文件依然会被重置，它并不是一个真正意义上的ignore；

方法二：

// 忽略指定文件
git update-index --skip-worktree path
// 恢复忽略的指定文件
git update-index --no-skip-worktree path

git update-index --skip-worktree这个命令也能使我们的文件处于忽略状态，即使执行git status依然会告诉你没有文件的改动；但是它与--assume-unchanged的区别在于，即使这个时候执行reset操作，文件内容依然不会被重置，它属于真正的ignore操作；

总结一下--assume-unchanged和--skip-worktree两者的适用场景：

--assume-unchanged比较适用于那些你想修改一个内容比较多的文件，然后中途伴随有其他文件的提交，但是此时不想针对这个文件有多次的提交记录，想直到全部修改完成以后一次性提交，这个时候你就可以用--assume-unchanged暂时忽略它，直到这个文件修改完成以后再进行提交；

--skip-worktree比较适用于那些长期在你的开发环境中不会修改的文件配置；

最后如果被忽略的文件太多想要统一进行查看的话，可以使用以下的命令：

--assume-unchanged ===> git ls-files -v | grep '^h\ '

--skip-worktree ===> git ls-files -v | grep -i ^S

```

#### 导出历史提交记录文件

**有的时候我们可能会遇到这么一个场景，针对与某一个功能改动，我们需要将这次改动对应的文件都提取出来；比较老实的做法可能是将分支切换到指定的节点，通过log对比，将修改的文件依次的提取出来；但是这显然是一个非常不可取的做法，对于那种改动量巨大的提交，手动操作很有可能会造成文件缺失的情况；**

**我们比较推荐的做法就是使用git archive命令，一般我们常见的git archive命令都是用在归档上面，比如当我们realse完成一个版本以后，我们都需要git tag对应的版本；但是其实我们也会经常使用git archive将对应的代码进行打包归档，归档以后的文件里不会包含git的相关版本信息，比较合适用于代码的传阅保存；**

**不过git archive还存在一个其他的用法，用来解决我们之前提出的问题；**

```
// 打包导出指定提交区间的文件改动
git archive -o <***.zip> <new_commit_id> $(git diff --name-only <old_commit_id> <new_commit_id>)


// 打包导出最后一次提交的文件改动
git archive -o <xxx.zip> HEAD $(gitdiff --name-only HEAD^)

其中需要注意的一点是，对于那些记录为删除操作的文件，可能会出现fatal: Pathspec的错误，这个时候可以使用 grep -v去排除这些文件;

git archive -o xxx.zip HEAD $(git diff HEAD^ --name-only|grep -v filename1|grep -v filename2)

```

#### 指定某次提交进行合并

**一般在我们的开发过程中会遇到如下的情况，我们的开发分支中存在commit1和commit2两条记录，此时我们将其中的commit1记录合并到master分支，但是不需要commit2这次记录；所以这个时候如果使用merge命令的话就会把commit2这次提交也合并到master上面去；如果我们采用手动对比修改提交方式的话，效率上就会特别差而且也容易出错；**

**git中正好有一条命令可以完美解决这个问题：git cherry-pick**

```
git cherry-pick可以选择某一个分支中的一个或几个commit(s)来进行操作。

// 单独合并一个提交
git cherry-pick <commit id>

// 单独合并一个提交 保留原提交者信息
git cherry-pick -x <commit id>


git cherry-pick也支持合并区间的方式，用以合并功能性的需求；

// 左开右闭，不包含start-commit-id
git cherry-pick <start-commit-id>..<end-commit-id>

// 闭区间，包含start-commit-id
git cherry-pick <start-commit-id>^..<end-commit-id>

```

#### 指定检出特定分支的文件

**一般我们会遇到这样一种情况，在当前的开发分支下，如果此时想要从其他分支检出特定的文件，比较笨的做法就是切换到其他分支然后拷贝出对应文件，再切回旧的分支；这样的做法无疑是很麻烦的，git checkout有一个对应的命令可以很好的解决我们的这个问题；**

```
// 检出特定分支的文件 path 就是你想要替换的目录或文件
git checkout <branch name> -- path

```
 




