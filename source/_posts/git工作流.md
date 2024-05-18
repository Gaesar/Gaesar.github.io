---
title: git工作流
date: 2024-05-18 23:56:28
tags: git
---

# git 工作流

## 做修改

1.git clone 把 remote中的项目克隆到本地

![image-20240518200138594](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518200138594.png)

2.修改代码的时候在本地建立一个新的feature branch，避免破坏主分支，并且有利于多人合作

​	复制当前的branch到新的branch

![image-20240518200513376](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518200513376.png)

3.git diff查看做的修改

![image-20240518200704883](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518200704883.png)

4.git add添加修改的文件到暂存区，把想要commit的文件告知给git

![image-20240518200827002](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518200827002.png)

5.git commit把修改提交到本地git

![image-20240518200929709](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518200929709.png)

6.git push把分支同步到远端 

![image-20240518201036636](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518201036636.png)

​	此时有可能别人修改了主分支

![image-20240518201202335](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518201202335.png)

7.切换到local main分支，把远端的main分支同步到local

![image-20240518201354944](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518201354944.png)

8.切换到local feature分支，rebase一下，rebase就是先把我的修改放到一边，把主分支的同步过来，然后再把我的修改放上去

​	如果出现了rebase conflic需要手动选择保留哪一个

![image-20240518201649833](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518201649833.png)

​	使用rebase相当于在最新的修改上添加了我们的修改，比merge更好

9.把当前的feature同步到远端

![image-20240518201948312](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518201948312.png)

10. 远端合并代码到主分支, pull request，主分支是属于整个项目的，不属于任何个人，feature分支是属于个人的，项目的主人负责审查代码和整合代码

    ![image-20240518202303533](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518202303533.png)

11. 把一个分支上的所有commit合并成一个commit合并到分支中，为了使main branch commit history尽可能地简洁，main branch每一个commit都是正常工作的

![image-20240518202514111](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518202514111.png)

​	合并成功后会把远端这个分支删除 delete branch

12.在本地删除branch

![image-20240518202830422](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518202830422.png)

13.再次从主分支上pull最新的主分支下来

![image-20240518202937294](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518202937294.png)

## 修改撤回

1.修改了磁盘文件，但还没有add到暂存区

​	git check out撤销，git restore也行

2.add到暂存区，但还没commit到本地git

​	git reset撤销，git resote --staged也行，移出暂存区

​	git check out HEAD，移出暂存区并且恢复磁盘文件，HEAD代表最新的一次commit

3.commit到本地git了，但还没有push到remote

​	git reset --soft HEAD~1 恢复到前一个commit的状态，只是移出了本地git

​	git reset HEAD~1 相当于 git reset --mixed HEAD~1. 移出git，然后再移出暂存区

​	git reset --hard HEAD~1 恢复到前一个commit的初始状态， 移出git，然后再移出暂存区，删除磁盘修改

​	git revert HEAD 增加一个commit和前一个commit相反

![image-20240518205919178](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518205919178.png)

reset只能回到之前一个commit的状态，revert可以去除之前的一个commit的效果

![image-20240518210108065](../assets/git%E5%B7%A5%E4%BD%9C%E6%B5%81/image-20240518210108065.png)

4.push到了remote

公有分支只能向前走，不能后退，只能使用git revert命令

个人分支只有自己使用，把这个commit直接删掉

git reset --hard HEAD~1 git push -f
