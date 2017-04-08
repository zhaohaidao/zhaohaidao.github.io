---
layout: post
title: git学习总结
---

# git checkout 与 git reset
+ git checkout
    - 抹去工作区中最近的改动(restore working tree files)
+ git reset
    - 从HEAD恢复到指定的commit版本中

# 合并提交
+ git rebase -i
    - 通过sqush，可以将多个commit合并，需要显式处理conflicts
+ git reset $(commit_id) （需要合并的commits集合之前最新的commit）
    - 加入提交了多次commit，可以reset到提交之前的版本，这时所有的改动都会进入工作区，重新提交即可
    - 注：需谨慎，如果提交记录是a->b->c，由c reset到 a，是没问题的，但是此时，从a，又reset到c就会删除a->b时提交的代码

# git branch

# git reflog
+ 保存了所有的操作历史，查找丢失commit的利器

# git log