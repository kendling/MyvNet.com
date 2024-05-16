---
title: "如何让 VSCode 兼容 cygwin 下的 git"
slug: "how-to-make-vscode-compatible-cygwin-git"
date: 2024-05-16
tags: ["cygwin", "git", "vscode", "gitlens"]
categories: ["Coding"]
image: "gitlens.png"
---

## VSCode 官方没有支持 cygwin git 的计划

众所周知， `VSCode` 官方并没有支持 `cygwin` 版本的 `git` 相关的计划。

官方 [Issue](https://github.com/microsoft/vscode/issues/7998#issuecomment-227867757) 已经明确说明：We don't support cygwin git, sorry.

## 寻找解决方案

寻思着不管啥问题，总有别人会遇到，于是开启搜索大法。

果不其然，找到一些解决方案：

1. https://gist.github.com/nickbudi/4b489f50086db805ea0f3864aa93a9f8
2. https://github.com/nukata/cyg-git

还有其他类似的方案，一通尝试过后，以 `cyg-git` 最稳定。

## GitLens 仍无法工作

配置好 `cyg-git` 作为中转后，`VSCode` 已经完美支持 `cygwin` 下 `clone` 的仓库了，无论是完整的还是 `worktree` 。

可是 `GitLens` 插件仍然无法正常工作。

表现为使用了 `cygwin` 格式的路径：
```
[2024-05-16 08:38:00.823] [*   992ms] [e:\CodeRepo\work2] git rev-parse --show-toplevel (slow)
[2024-05-16 08:38:00.828] [      2ms] [/e/CodeRepo/work2] git rev-parse --git-dir --git-common-dir
```

## 分析问题

分析过程断断续续找了很久。

`GitLens` 运行过程中还会报以下的错误：
```
[2024-05-16 08:49:09.430] [      1ms] [/e/CodeRepo/work2] git rev-parse --abbrev-ref --symbolic-full-name @ @{u} -- • FAILED

Error: spawn F:/cygwin/git.exe ENOENT
```
开启了 `GitLens` 的调试选项 ***"gitlens.debug": true*** 也没用，没有找到具体的问题点。

最后下载了 `GitLens` 的代码进行调试，发现它实际调用 `git` 的时候并非如它的 log 输出一样。

如上面的日志：
```
[2024-05-16 08:38:00.823] [*   992ms] [e:\CodeRepo\work2] git rev-parse --show-toplevel (slow)
```
实际传给 git 的参数是：
```
F:/cygwin/git.exe -c core.longpaths=true -c core.quotepath=false -c color.ui=false rev-parse --show-toplevel
```
这就。。。

查看 `cyg-git` 里 `git-wrapper` 的代码，处理 `rev-parse` 参数的代码如下：
```bash
#!/usr/bin/zsh -
PATH="/usr/bin:$PATH"
wrap_output=""
for ((i = 1; i <= $#; i++)); do
    a="$argv[i]"
    if [ -z "$a" ]; then
        argv[i]=()
        ((i--))
    elif [ "$a" = 'rev-parse' -a $i = 1 ]; then # 注意看这里
        argv[1]="$a"
        wrap_output=yes
    elif [ "$a[1]" != '-' ]; then 
        argv[i]=`cygpath -u "$a"`
    fi
done
if [ $wrap_output ]; then
    if a=`git "$@"`; then
        if [ -z "$a" ]; then
        elif [ "$a[1]" = '-' ]; then
            echo $a
        else
            cygpath -w "$a"
        fi
    else
        x=$?
        echo $a
        exit $x
    fi
else
    exec git "$@"
fi
```

`git-wrapper` 的代码只处理 `rev-parse` 参数是第一个的情况。

所以 `GitLens` 的调用并没有被正确处理，和直接调用 `git` 的效果是一样的。因此出现了上面的 log ，路径变成了 `cygwin` 格式的 `/e/CodeRepo` 。

## 解决

问题找到了就好办，`git-wrapper` 修 bug 一波。
```bash
#!/usr/bin/zsh -
PATH="/usr/bin:$PATH"
wrap_output=""

for ((i = 1; i <= $#; i++)); do
    a="$argv[i]"
    if [ -z "$a" ]; then
        argv[i]=()
        ((i--))
    elif [ "$a" = 'rev-parse' ]; then
        wrap_output=yes
    elif [[ "$a" = *longpaths* ]]; then
		# git in cygwin not support core.longpaths
        argv[i]="gc.autodetach=false"
    elif [ "$a[1]" != '-' ]; then 
        argv[i]=`cygpath -u "$a"`
    fi
done

if [ $wrap_output ]; then
    if a=`git "$@"`; then
        if [ -z "$a" ]; then
        #elif [ "$a[1]" = '-' ]; then
        #    echo $a
        elif [ "$a[1]" = '/' ]; then
			array=(`echo $a | tr '\n' ' '`)
			for a in ${array[@]}; do
				cygpath -w "$a" | sed 's#\\#/#g'
			done
        else
            echo $a
        fi
    else
        x=$?
        echo $a
        exit $x
    fi
else
    exec git "$@"
fi
```
这里除了上面说到的 `rev-parse` 参数的问题还修正了另外一个 bug 。

原代码是 `git` 返回的内容非 `-` 开头的就用 `cygpath -w` 转换，这个可能会导致错误的结果，例如分支名 `release/v10` 会被替换成 `release\v10` 。

我改成了只转换 `/` 开头的内容，运行一段时间暂时没有发现其他问题。

## 整理分享

修改后的代码我放到 https://github.com/kendling/cyg-git 仓库了。

我在原作者的基础上修改了以下几点：

1. 正确处理 `rev-parse` 参数没有在第一个的情况
2. `git` 返回的内容只对 `/` 开头的内容进行 `cygpath -w` 转换
3. `git.c` 原作者是写死 `cygwin` 路径，我改成读取当前目录下的 `git.cfg` 文件
4. `git.c` 在 `git.cfg` 文件不存在的情况下，查找所有盘的 `:\cygwin64\bin` 和 `X:\cygwin\bin` 目录，找到则以此目录作为 `cygwin` 的路径

`git.c` 编译方法和原作者的一样：
```bash
# 我增加了 -O3 参数
x86_64-w64-mingw32-gcc -O3 -o git.exe git.c
# 或者
gcc -O3 -o git.exe git.c
```
