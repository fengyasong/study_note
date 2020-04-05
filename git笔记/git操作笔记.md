# git切换到指定远程分支

* 查看远程所有分支

    `git branch -a`
    ```
    $ git branch -a
    * master
    remotes/origin/HEAD -> origin/master
    remotes/origin/master
    ```
    git branch不带参数,列出本地已经存在的分支，并且在当前分支的前面用*标记，加上-a参数可以查看所有分支列表，包括本地和远程，远程分支一般会用红色字体标记出来

* 新建分支并切换到指定分支

    `git checkout -b 本地分支名 origin/远程分支名`

* 查看本地分支及追踪的分支

    `git branch -vv`

    `* master 2eb3966 [origin/master] 完成markdown笔记`

    *表示当前所在分支，[远程分支]表示当前本地分支追踪的远程分支，最后一个是最近一次提交的注释。

* 将本地分支推送到远程
    `git push <远程主机名> <本地分支名>:<远程分支名>`