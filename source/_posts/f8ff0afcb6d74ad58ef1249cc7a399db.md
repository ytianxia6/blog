---
layout: post
title: 迁移svn到git遇到的各种问题及解决
abbrlink: f8ff0afcb6d74ad58ef1249cc7a399db
tags:
  - 随笔
  - git
  - svn
date: 1552008171000
updated: 1552011841000
sticky: null
---

# 迁移 svn 到 git

要迁移 svn 到 git ,遇到了很多问题，所幸后来一一解决了。

没有使用迁移工具 subgit，而是按照[官网](https://git-scm.com/book/zh/v1/Git-%E4%B8%8E%E5%85%B6%E4%BB%96%E7%B3%BB%E7%BB%9F-%E8%BF%81%E7%A7%BB%E5%88%B0-Git)的步骤来做的。

# 一. 调用 git svn clone 时卡在初始化空的 git 库。

`Initialized empty Git repository in xxx/.git/`

## 原因：

svn 库版本太高，我使用的是 visual svn server 最新版本。

## 解决方案：

我是通过创建一个 ubuntu 虚拟机，在里面搭建新的 svn 服务器解决的。也可以通过其它方式搭建低版本 subversion 解决。

# 二. 不规范分支

svn 库中存在一些不标准的分支，这些分支的形式各种各样，总之不是 `/branches/xx` 的形式，有如 `2018/xx/分支名` 的不从 `branches` 开始的分支，有 `/branches/xx/分支名` 的多级分支，它们都不能在 git svn clone 时被正确识别。

## 解决方案：

这个问题有 3 种解决方案，都要通过 dump 文件解决。
如何导出 dump 文件不进行描述，

```
svnadmin dump xx > xx.dump
svnrdump dump svn/remote/url > xx.dump
```

### 1. 直接使用 dump 文件。

直接以文本方式打开 dump 文件修改 要修改的路径。但这种方式，我总是不成功。而且对大文件操作很慢

### 2. [svn-dump-reloc](https://metacpan.org/pod/distribution/SVN-DumpReloc/bin/svn-dump-reloc)

在网上找到这个工具，可以很方便的修改路径。 问题是这种方式每次修改都需要先 Load 到仓库，而不是直接修改 dump 文件。安装也很费劲。

### 3. sed 命令

可以直接通过 sed 命令修改路径，原理应该也 svn-dump-reloc 差不多

```
*# 比如要修改 路径 2018/xx 到 branches/2018_xx*
sed -i 's/Node-path: 2019\/xx/Node-path: branches\/2018_xx/g' xx.dump

sed -i 's/Node-copyfrom-path: 2019\/xx/Node-copyfrom-path: branches\/2018_xx/g' xx.dump

```

### 注意事项

不要在 git 里清理错误的分支和误传的大文件。这可能会导致加载失败。

# 三. 清理分支和标签时旧标签删除不掉

按照官网的步骤

> 你还需要一点  post-import（导入后）  清理工作。最起码的，应该清理一下  git svn  创建的那些怪异的索引结构。首先要移动标签，把它们从奇怪的远程分支变成实际的标签，然后把剩下的分支移动到本地。

> 要把标签变成合适的 Git 标签，运行

`$ git for-each-ref refs/remotes/tags | cut -d / -f 4- | grep -v @ | while read tagname; do git tag "$tagname" "tags/$tagname"; git branch -r -d "tags/$tagname"; done`

> 该命令将原本以  tag/  开头的远程分支的索引变成真正的（轻巧的）标签。
> 接下来，把  refs/remotes  下面剩下的索引变成本地分支：

`$ git for-each-ref refs/remotes | cut -d / -f 3- | grep -v @ | while read branchname; do git branch "$branchname" "refs/remotes/$branchname"; git branch -r -d "$branchname"; done`

按照这个过程来做，可以生成新的分支和标签，但旧的标签和分支删除不掉，新生成的分支也会带一个 origin/

## 原因

官网的脚本对路径计算时，计算的是 refs/remotes/xx 而真实情况是 refs/remotes/origin/xx，需要对路径进行修正 。

## 解决方案：

对脚本 进行修改：

```
*#标签*

git for-each-ref refs/remotes/origin/tags | cut -d / -f 5- | grep -v @ | while read tagname; do git tag "$tagname" "origin/tags/$tagname"; git branch -r -d "origin/tags/$tagname"; done

*#分支*

git for-each-ref refs/remotes/origin | cut -d / -f 4- | grep -v @ | while read branchname; do git branch "$branchname" "refs/remotes/origin/$branchname"; git branch -r -d "origin/$branchname"; done

```

现在，就可以得到正确清爽的路径了。

# 四. svn 库太大无法上传。

## 原因：

svn 时代，对文件大小没有太大限制。而 git 对路径有根本的限制。虽然可以在服务器修改，但太大也没有意义。

## 解决方案：

找到大文件，进行修改。

参照[官网](https://git-scm.com/book/zh/v1/Git-%E5%B7%A5%E5%85%B7-%E9%87%8D%E5%86%99%E5%8E%86%E5%8F%B2#%E6%A0%B8%E5%BC%B9%E7%BA%A7%E9%80%89%E9%A1%B9:-filter-branch)的办法删除所有分支中的过往。

```
*#查找文件名并删除*

git filter-branch -f --tree-filter "find * -type f -name 'xx*' -delete" -- --all

*#删除指定路径目录*
git filter-branch -f --tree-filter "rm -rf '要删除/文件夹'" -- --all
```

## 补充，找不到大文件路径怎么办

参照 <https://www.cnblogs.com/langzou/p/9877165.html>
`git verify-pack -v .git/objects/pack/pack-*.idx | sort -k 3 -g | tail -10`

<div style="display: none;">%23%20%E8%BF%81%E7%A7%BBsvn%20%E5%88%B0%20git%20%0A%0A%E8%A6%81%E8%BF%81%E7%A7%BBsvn%E5%88%B0git%20%2C%E9%81%87%E5%88%B0%E4%BA%86%E5%BE%88%E5%A4%9A%E9%97%AE%E9%A2%98%EF%BC%8C%E6%89%80%E5%B9%B8%E5%90%8E%E6%9D%A5%E4%B8%80%E4%B8%80%E8%A7%A3%E5%86%B3%E4%BA%86%E3%80%82%0A%0A%E6%B2%A1%E6%9C%89%E4%BD%BF%E7%94%A8%E8%BF%81%E7%A7%BB%E5%B7%A5%E5%85%B7subgit%EF%BC%8C%E8%80%8C%E6%98%AF%E6%8C%89%E7%85%A7%5B%E5%AE%98%E7%BD%91%5D(https%3A%2F%2Fgit-scm.com%2Fbook%2Fzh%2Fv1%2FGit-%E4%B8%8E%E5%85%B6%E4%BB%96%E7%B3%BB%E7%BB%9F-%E8%BF%81%E7%A7%BB%E5%88%B0-Git)%E7%9A%84%E6%AD%A5%E9%AA%A4%E6%9D%A5%E5%81%9A%E7%9A%84%E3%80%82%0A%0A%23%20%E4%B8%80.%20%E8%B0%83%E7%94%A8git%20svn%20clone%20%E6%97%B6%E5%8D%A1%E5%9C%A8%E5%88%9D%E5%A7%8B%E5%8C%96%E7%A9%BA%E7%9A%84git%E5%BA%93%E3%80%82%0A%0A%60%60%60bash%0AInitialized%20empty%20Git%20repository%20in%20xxx%2F.git%2F%0A%60%60%60%0A%0A%23%23%20%E5%8E%9F%E5%9B%A0%EF%BC%9A%0Asvn%20%E5%BA%93%E7%89%88%E6%9C%AC%E5%A4%AA%E9%AB%98%EF%BC%8C%E6%88%91%E4%BD%BF%E7%94%A8%E7%9A%84%E6%98%AFvisual%20svn%20server%20%E6%9C%80%E6%96%B0%E7%89%88%E6%9C%AC%E3%80%82%0A%0A%23%23%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%9A%0A%E6%88%91%E6%98%AF%E9%80%9A%E8%BF%87%E5%88%9B%E5%BB%BA%E4%B8%80%E4%B8%AAubuntu%20%E8%99%9A%E6%8B%9F%E6%9C%BA%EF%BC%8C%E5%9C%A8%E9%87%8C%E9%9D%A2%E6%90%AD%E5%BB%BA%E6%96%B0%E7%9A%84svn%E6%9C%8D%E5%8A%A1%E5%99%A8%E8%A7%A3%E5%86%B3%E7%9A%84%E3%80%82%E4%B9%9F%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E5%85%B6%E5%AE%83%E6%96%B9%E5%BC%8F%E6%90%AD%E5%BB%BA%E4%BD%8E%E7%89%88%E6%9C%ACsubversion%E8%A7%A3%E5%86%B3%E3%80%82%0A%0A%23%20%E4%BA%8C.%20%E4%B8%8D%E8%A7%84%E8%8C%83%E5%88%86%E6%94%AF%0A%0Asvn%E5%BA%93%E4%B8%AD%E5%AD%98%E5%9C%A8%E4%B8%80%E4%BA%9B%E4%B8%8D%E6%A0%87%E5%87%86%E7%9A%84%E5%88%86%E6%94%AF%EF%BC%8C%E8%BF%99%E4%BA%9B%E5%88%86%E6%94%AF%E7%9A%84%E5%BD%A2%E5%BC%8F%E5%90%84%E7%A7%8D%E5%90%84%E6%A0%B7%EF%BC%8C%E6%80%BB%E4%B9%8B%E4%B8%8D%E6%98%AF%20%60%2Fbranches%2Fxx%60%20%E7%9A%84%E5%BD%A2%E5%BC%8F%EF%BC%8C%E6%9C%89%E5%A6%82%20%602018%2Fxx%2F%E5%88%86%E6%94%AF%E5%90%8D%60%20%E7%9A%84%E4%B8%8D%E4%BB%8E%20%60branches%60%20%E5%BC%80%E5%A7%8B%E7%9A%84%E5%88%86%E6%94%AF%EF%BC%8C%E6%9C%89%20%60%2Fbranches%2Fxx%2F%E5%88%86%E6%94%AF%E5%90%8D%60%20%E7%9A%84%E5%A4%9A%E7%BA%A7%E5%88%86%E6%94%AF%EF%BC%8C%E5%AE%83%E4%BB%AC%E9%83%BD%E4%B8%8D%E8%83%BD%E5%9C%A8%20git%20svn%20clone%20%E6%97%B6%E8%A2%AB%E6%AD%A3%E7%A1%AE%E8%AF%86%E5%88%AB%E3%80%82%0A%0A%23%23%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%9A%0A%E8%BF%99%E4%B8%AA%E9%97%AE%E9%A2%98%E6%9C%893%E7%A7%8D%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%8C%E9%83%BD%E8%A6%81%E9%80%9A%E8%BF%87dump%E6%96%87%E4%BB%B6%E8%A7%A3%E5%86%B3%E3%80%82%0A%0A%E5%A6%82%E4%BD%95%E5%AF%BC%E5%87%BAdump%20%E6%96%87%E4%BB%B6%E4%B8%8D%E8%BF%9B%E8%A1%8C%E6%8F%8F%E8%BF%B0%EF%BC%8C%0A%60%60%60%20bash%0Asvnadmin%20dump%20xx%20%3E%20xx.dump%0Asvnrdump%20dump%20svn%2Fremote%2Furl%20%3E%20xx.dump%0A%60%60%60%0A%0A%23%23%23%201.%20%E7%9B%B4%E6%8E%A5%E4%BD%BF%E7%94%A8dump%E6%96%87%E4%BB%B6%E3%80%82%20%0A%E7%9B%B4%E6%8E%A5%E4%BB%A5%E6%96%87%E6%9C%AC%E6%96%B9%E5%BC%8F%E6%89%93%E5%BC%80dump%E6%96%87%E4%BB%B6%E4%BF%AE%E6%94%B9%20%E8%A6%81%E4%BF%AE%E6%94%B9%E7%9A%84%E8%B7%AF%E5%BE%84%E3%80%82%E4%BD%86%E8%BF%99%E7%A7%8D%E6%96%B9%E5%BC%8F%EF%BC%8C%E6%88%91%E6%80%BB%E6%98%AF%E4%B8%8D%E6%88%90%E5%8A%9F%E3%80%82%E8%80%8C%E4%B8%94%E5%AF%B9%E5%A4%A7%E6%96%87%E4%BB%B6%E6%93%8D%E4%BD%9C%E5%BE%88%E6%85%A2%0A%23%23%23%202.%20%5Bsvn-dump-reloc%5D(https%3A%2F%2Fmetacpan.org%2Fpod%2Fdistribution%2FSVN-DumpReloc%2Fbin%2Fsvn-dump-reloc)%0A%E5%9C%A8%E7%BD%91%E4%B8%8A%E6%89%BE%E5%88%B0%E8%BF%99%E4%B8%AA%E5%B7%A5%E5%85%B7%EF%BC%8C%E5%8F%AF%E4%BB%A5%E5%BE%88%E6%96%B9%E4%BE%BF%E7%9A%84%E4%BF%AE%E6%94%B9%E8%B7%AF%E5%BE%84%E3%80%82%20%E9%97%AE%E9%A2%98%E6%98%AF%E8%BF%99%E7%A7%8D%E6%96%B9%E5%BC%8F%E6%AF%8F%E6%AC%A1%E4%BF%AE%E6%94%B9%E9%83%BD%E9%9C%80%E8%A6%81%E5%85%88Load%20%E5%88%B0%E4%BB%93%E5%BA%93%EF%BC%8C%E8%80%8C%E4%B8%8D%E6%98%AF%E7%9B%B4%E6%8E%A5%E4%BF%AE%E6%94%B9%20dump%E6%96%87%E4%BB%B6%E3%80%82%E5%AE%89%E8%A3%85%E4%B9%9F%E5%BE%88%E8%B4%B9%E5%8A%B2%E3%80%82%0A%23%23%23%203.%20sed%20%E5%91%BD%E4%BB%A4%0A%E5%8F%AF%E4%BB%A5%E7%9B%B4%E6%8E%A5%E9%80%9A%E8%BF%87sed%20%E5%91%BD%E4%BB%A4%E4%BF%AE%E6%94%B9%E8%B7%AF%E5%BE%84%EF%BC%8C%E5%8E%9F%E7%90%86%E5%BA%94%E8%AF%A5%E4%B9%9F%20svn-dump-reloc%20%E5%B7%AE%E4%B8%8D%E5%A4%9A%0A%60%60%60bash%0A%23%20%E6%AF%94%E5%A6%82%E8%A6%81%E4%BF%AE%E6%94%B9%20%E8%B7%AF%E5%BE%84%202018%2Fxx%20%E5%88%B0%20branches%2F2018_xx%0Ased%20-i%20's%2FNode-path%3A%202019%5C%2Fxx%2FNode-path%3A%20branches%5C%2F2018_xx%2Fg'%20xx.dump%0Ased%20-i%20's%2FNode-copyfrom-path%3A%202019%5C%2Fxx%2FNode-copyfrom-path%3A%20branches%5C%2F2018_xx%2Fg'%20xx.dump%0A%60%60%60%0A%0A%23%23%23%20%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9%0A%E4%B8%8D%E8%A6%81%E5%9C%A8git%20%E9%87%8C%E6%B8%85%E7%90%86%E9%94%99%E8%AF%AF%E7%9A%84%E5%88%86%E6%94%AF%E5%92%8C%E8%AF%AF%E4%BC%A0%E7%9A%84%E5%A4%A7%E6%96%87%E4%BB%B6%E3%80%82%E8%BF%99%E5%8F%AF%E8%83%BD%E4%BC%9A%E5%AF%BC%E8%87%B4%E5%8A%A0%E8%BD%BD%E5%A4%B1%E8%B4%A5%E3%80%82%0A%0A%0A%23%20%E4%B8%89.%20%E6%B8%85%E7%90%86%E5%88%86%E6%94%AF%E5%92%8C%E6%A0%87%E7%AD%BE%E6%97%B6%E6%97%A7%E6%A0%87%E7%AD%BE%E5%88%A0%E9%99%A4%E4%B8%8D%E6%8E%89%0A%E6%8C%89%E7%85%A7%E5%AE%98%E7%BD%91%E7%9A%84%E6%AD%A5%E9%AA%A4%0A%3E%E4%BD%A0%E8%BF%98%E9%9C%80%E8%A6%81%E4%B8%80%E7%82%B9%C2%A0post-import%EF%BC%88%E5%AF%BC%E5%85%A5%E5%90%8E%EF%BC%89%C2%A0%E6%B8%85%E7%90%86%E5%B7%A5%E4%BD%9C%E3%80%82%E6%9C%80%E8%B5%B7%E7%A0%81%E7%9A%84%EF%BC%8C%E5%BA%94%E8%AF%A5%E6%B8%85%E7%90%86%E4%B8%80%E4%B8%8B%C2%A0git%20svn%C2%A0%E5%88%9B%E5%BB%BA%E7%9A%84%E9%82%A3%E4%BA%9B%E6%80%AA%E5%BC%82%E7%9A%84%E7%B4%A2%E5%BC%95%E7%BB%93%E6%9E%84%E3%80%82%E9%A6%96%E5%85%88%E8%A6%81%E7%A7%BB%E5%8A%A8%E6%A0%87%E7%AD%BE%EF%BC%8C%E6%8A%8A%E5%AE%83%E4%BB%AC%E4%BB%8E%E5%A5%87%E6%80%AA%E7%9A%84%E8%BF%9C%E7%A8%8B%E5%88%86%E6%94%AF%E5%8F%98%E6%88%90%E5%AE%9E%E9%99%85%E7%9A%84%E6%A0%87%E7%AD%BE%EF%BC%8C%E7%84%B6%E5%90%8E%E6%8A%8A%E5%89%A9%E4%B8%8B%E7%9A%84%E5%88%86%E6%94%AF%E7%A7%BB%E5%8A%A8%E5%88%B0%E6%9C%AC%E5%9C%B0%E3%80%82%0A%3E%E8%A6%81%E6%8A%8A%E6%A0%87%E7%AD%BE%E5%8F%98%E6%88%90%E5%90%88%E9%80%82%E7%9A%84%20Git%20%E6%A0%87%E7%AD%BE%EF%BC%8C%E8%BF%90%E8%A1%8C%0A%3E%20%60%60%60%0A%3E%20%24%20git%20for-each-ref%20refs%2Fremotes%2Ftags%20%7C%20cut%20-d%20%2F%20-f%204-%20%7C%20grep%20-v%20%40%20%7C%20while%20read%20tagname%3B%20do%20git%20tag%20%22%24tagname%22%20%22tags%2F%24tagname%22%3B%20git%20branch%20-r%20-d%20%22tags%2F%24tagname%22%3B%20done%0A%3E%20%60%60%60%0A%3E%E8%AF%A5%E5%91%BD%E4%BB%A4%E5%B0%86%E5%8E%9F%E6%9C%AC%E4%BB%A5%C2%A0tag%2F%C2%A0%E5%BC%80%E5%A4%B4%E7%9A%84%E8%BF%9C%E7%A8%8B%E5%88%86%E6%94%AF%E7%9A%84%E7%B4%A2%E5%BC%95%E5%8F%98%E6%88%90%E7%9C%9F%E6%AD%A3%E7%9A%84%EF%BC%88%E8%BD%BB%E5%B7%A7%E7%9A%84%EF%BC%89%E6%A0%87%E7%AD%BE%E3%80%82%0A%3E%E6%8E%A5%E4%B8%8B%E6%9D%A5%EF%BC%8C%E6%8A%8A%C2%A0refs%2Fremotes%C2%A0%E4%B8%8B%E9%9D%A2%E5%89%A9%E4%B8%8B%E7%9A%84%E7%B4%A2%E5%BC%95%E5%8F%98%E6%88%90%E6%9C%AC%E5%9C%B0%E5%88%86%E6%94%AF%EF%BC%9A%0A%3E%60%60%60%0A%3E%24%20git%20for-each-ref%20refs%2Fremotes%20%7C%20cut%20-d%20%2F%20-f%203-%20%7C%20grep%20-v%20%40%20%7C%20while%20read%20branchname%3B%20do%20git%20branch%20%22%24branchname%22%20%22refs%2Fremotes%2F%24branchname%22%3B%20git%20branch%20-r%20-d%20%22%24branchname%22%3B%20done%0A%3E%60%60%60%0A%0A%E6%8C%89%E7%85%A7%E8%BF%99%E4%B8%AA%E8%BF%87%E7%A8%8B%E6%9D%A5%E5%81%9A%EF%BC%8C%E5%8F%AF%E4%BB%A5%E7%94%9F%E6%88%90%E6%96%B0%E7%9A%84%E5%88%86%E6%94%AF%E5%92%8C%E6%A0%87%E7%AD%BE%EF%BC%8C%E4%BD%86%E6%97%A7%E7%9A%84%E6%A0%87%E7%AD%BE%E5%92%8C%E5%88%86%E6%94%AF%E5%88%A0%E9%99%A4%E4%B8%8D%E6%8E%89%EF%BC%8C%E6%96%B0%E7%94%9F%E6%88%90%E7%9A%84%E5%88%86%E6%94%AF%E4%B9%9F%E4%BC%9A%E5%B8%A6%E4%B8%80%E4%B8%AA%20origin%2F%20%0A%0A%23%23%20%E5%8E%9F%E5%9B%A0%0A%E5%AE%98%E7%BD%91%E7%9A%84%E8%84%9A%E6%9C%AC%E5%AF%B9%E8%B7%AF%E5%BE%84%E8%AE%A1%E7%AE%97%E6%97%B6%EF%BC%8C%E8%AE%A1%E7%AE%97%E7%9A%84%E6%98%AF%20refs%2Fremotes%2Fxx%20%E8%80%8C%E7%9C%9F%E5%AE%9E%E6%83%85%E5%86%B5%E6%98%AF%20refs%2Fremotes%2Forigin%2Fxx%EF%BC%8C%E9%9C%80%E8%A6%81%E5%AF%B9%E8%B7%AF%E5%BE%84%E8%BF%9B%E8%A1%8C%E4%BF%AE%E6%AD%A3%20%E3%80%82%0A%0A%23%23%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%9A%0A%0A%E5%AF%B9%E8%84%9A%E6%9C%AC%20%E8%BF%9B%E8%A1%8C%E4%BF%AE%E6%94%B9%EF%BC%9A%0A%60%60%60%20bash%0A%23%E6%A0%87%E7%AD%BE%0Agit%20for-each-ref%20refs%2Fremotes%2Forigin%2Ftags%20%7C%20cut%20-d%20%2F%20-f%205-%20%7C%20grep%20-v%20%40%20%7C%20while%20read%20tagname%3B%20do%20git%20tag%20%22%24tagname%22%20%22origin%2Ftags%2F%24tagname%22%3B%20git%20branch%20-r%20-d%20%22origin%2Ftags%2F%24tagname%22%3B%20done%0A%0A%23%E5%88%86%E6%94%AF%0Agit%20for-each-ref%20refs%2Fremotes%2Forigin%20%7C%20cut%20-d%20%2F%20-f%204-%20%7C%20grep%20-v%20%40%20%7C%20while%20read%20branchname%3B%20do%20git%20branch%20%22%24branchname%22%20%22refs%2Fremotes%2Forigin%2F%24branchname%22%3B%20git%20branch%20-r%20-d%20%22origin%2F%24branchname%22%3B%20done%0A%0A%60%60%60%0A%0A%E7%8E%B0%E5%9C%A8%EF%BC%8C%E5%B0%B1%E5%8F%AF%E4%BB%A5%E5%BE%97%E5%88%B0%E6%AD%A3%E7%A1%AE%E6%B8%85%E7%88%BD%E7%9A%84%E8%B7%AF%E5%BE%84%E4%BA%86%E3%80%82%0A%0A%23%20%E5%9B%9B.%20svn%E5%BA%93%E5%A4%AA%E5%A4%A7%E6%97%A0%E6%B3%95%E4%B8%8A%E4%BC%A0%E3%80%82%0A%0A%23%23%20%E5%8E%9F%E5%9B%A0%EF%BC%9A%20%0Asvn%20%E6%97%B6%E4%BB%A3%EF%BC%8C%E5%AF%B9%E6%96%87%E4%BB%B6%E5%A4%A7%E5%B0%8F%E6%B2%A1%E6%9C%89%E5%A4%AA%E5%A4%A7%E9%99%90%E5%88%B6%E3%80%82%E8%80%8Cgit%20%E5%AF%B9%E8%B7%AF%E5%BE%84%E6%9C%89%E6%A0%B9%E6%9C%AC%E7%9A%84%E9%99%90%E5%88%B6%E3%80%82%E8%99%BD%E7%84%B6%E5%8F%AF%E4%BB%A5%E5%9C%A8%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%BF%AE%E6%94%B9%EF%BC%8C%E4%BD%86%E5%A4%AA%E5%A4%A7%E4%B9%9F%E6%B2%A1%E6%9C%89%E6%84%8F%E4%B9%89%E3%80%82%0A%0A%23%23%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88%EF%BC%9A%0A%E6%89%BE%E5%88%B0%E5%A4%A7%E6%96%87%E4%BB%B6%EF%BC%8C%E8%BF%9B%E8%A1%8C%E4%BF%AE%E6%94%B9%E3%80%82%0A%E5%8F%82%E7%85%A7%5B%E5%AE%98%E7%BD%91%5D(https%3A%2F%2Fgit-scm.com%2Fbook%2Fzh%2Fv1%2FGit-%E5%B7%A5%E5%85%B7-%E9%87%8D%E5%86%99%E5%8E%86%E5%8F%B2%23%E6%A0%B8%E5%BC%B9%E7%BA%A7%E9%80%89%E9%A1%B9%3A-filter-branch)%E7%9A%84%E5%8A%9E%E6%B3%95%E5%88%A0%E9%99%A4%E6%89%80%E6%9C%89%E5%88%86%E6%94%AF%E4%B8%AD%E7%9A%84%E8%BF%87%E5%BE%80%E3%80%82%0A%0A%60%60%60bash%0A%23%E6%9F%A5%E6%89%BE%E6%96%87%E4%BB%B6%E5%90%8D%E5%B9%B6%E5%88%A0%E9%99%A4%0Agit%20filter-branch%20-f%20--tree-filter%20%22find%20*%20-type%20f%20-name%20'xx*'%20-delete%22%20--%20--all%0A%0A%23%E5%88%A0%E9%99%A4%E6%8C%87%E5%AE%9A%E8%B7%AF%E5%BE%84%E7%9B%AE%E5%BD%95%0Agit%20filter-branch%20-f%20--tree-filter%20%22rm%20-rf%20'%E8%A6%81%E5%88%A0%E9%99%A4%2F%E6%96%87%E4%BB%B6%E5%A4%B9'%22%20--%20--all%0A%0A%60%60%60%0A%0A%23%23%20%E8%A1%A5%E5%85%85%EF%BC%8C%E6%89%BE%E4%B8%8D%E5%88%B0%E5%A4%A7%E6%96%87%E4%BB%B6%E8%B7%AF%E5%BE%84%E6%80%8E%E4%B9%88%E5%8A%9E%0A%0A%E5%8F%82%E7%85%A7%20https%3A%2F%2Fwww.cnblogs.com%2Flangzou%2Fp%2F9877165.html%0A%0A%60%60%60bash%0Agit%20verify-pack%20-v%20.git%2Fobjects%2Fpack%2Fpack-*.idx%20%7C%20sort%20-k%203%20-g%20%7C%20tail%20-10%0A%60%60%60%0A%0A</div>
