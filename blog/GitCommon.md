---
layout: post
title: "Git 常见命令及其操作"
date: 2017-4-10 23:16
comments: true
tags: 
	- 技术
---

---
##### 基本Git 命令

	1. 配置Git
	    git config --global user.name "YourName"
	    git config --global user.email "email@example.com"
	
	2. 创建版本库
	    git init
	
	3. 查看当前版本库的状态
	    git status
	
	4. 查看当前窗口的所有log
	    git log --pretty=oneline
	
	5. 查看历史log
	    git reflog
	
	6. 添加文件到暂存区
	    git add readme.txt
		git add .								一次添加所有的文件	

	7. 提交修改
	    git commit -m "example commit"
	
	8. 版本回退
	    git reset --hard 23230(commit id)       回退到指定版本       
	    git reset --hard HEAD^                  回退到上一个版本
        git reset --hard HEAD~8                 回退到前8的那个版本
	
	9. 撤销修改
	    git checkout -- readme.md
	
	10. 撤销暂存区(git add)的修改，重新放回工作区
	    git reset HEAD readme.txt
	
	11. 从版本库中删除文件
	    git rm test.txt

    12. 对比不同，查看修改的内容
        git diff readme.txt
    
	13. 删除已经被提交的文件（本地加入ignore，需要删除远程仓库中的）
		git rm -r build.xml						从版本库中删除（单个文件）
		git rm -r build/						从版本库中删除（整个build目录）
		git commit -m "ignore commit files"		提交
		git push origin master					推送到远程仓库
    14.回滚代码
        git checkout -b sm v4.9.7      本地建个sm分支跟踪 v4.9.7 tag
        git push -f origin sm:master   强制把master回滚到 sm 分支（相当于回滚到 4.9.7tag）
        git branch -D master       删除并重建本地master
        git checkout -b master --track origin/master

##### 远程仓库

	1. 使用Coding创建一个远程仓库
	
	2. 运行命令
		git remote add origin https://github.com/googlesamples/android-ndk.git
	
	3. 将本地库的所有内容推送到远程仓库上
		git push -u origin master	（初次提交）
		git push origin master		（初次提交后再次提交）
	
	4. 从远程仓库克隆
		git clone https://github.com/googlesamples/android-ndk.git		（远程 master分支）
		git clone -b dev https://github.com/googlesamples/android-ndk.git	（远程 dev分支）
		

##### 分支管理
    
    1. 创建分支
        git branch dev
    
    2. 切换分支
        git checkout dev
    
    3. 创建并切换分支
        git checkout -b dev
    
    4. 查看当前分支
        git branch
    
    5. 删除分支
        git branch -d dev
    
    6. 推送本地的分支到服务器
        git push origin dev
    
    7. 查看本地与远程的所有分支
        git branch -a
    
    8. 删除远程分支
        git push origin --delete dev
    
    9. 删除远程tag
        git push origin --delete tag tag_1

##### 解决冲突
    1. 使用 git merge dev 合并分支代码时，Git会使用<<<<<<,======,>>>>>>标记处不同分支的内容，
    待我们手动合并冲突后，保存之后重新提交即可完成分支合并
    
    2. 使用命令查看分支合并图
       git log --graph
       
##### 分支管理策略
 ![分支管理策略](manage_branch.png)
 
    1. 通常，合并分支时，如果可能，Git会使用Fast forward模式，但在这种模式下，删除分支后，会丢掉分支信息。
    如果要强制禁用Fast forward模式，Git就会在merge时生成一个新的commit，从分支历史就可以看出分支信息。
    
    2. 使用如下命令禁用Fast forward模式
        git merge --no-ff -m "merge with no-ff" dev
        
##### Bug分支
    
    1. 接到修复代号101bug的任务，创建一个分支issue-101来修复它。但是，当前正在dev上进行的工作还没有提交。
        存储工作现场，存储之后 git status将没有内容，即工作区是干净的。
        git stash    存储工作现场
    
    2. 新建分支issue-101修复bug，并合并到master分支。
        git checkout -b issue-101 
        git add motifity.java
        git commit motiffity.java
        git checkout master
        git merge --no-ff -m "merged bug fix 101" issue-101
        git branch -d issue-101
    
    3. 切回dev分支干活，并恢复现场
        git stash list      查看刚才的工作现场
        git stash apply     恢复最近的工作现场
        git stash drop      删除存储的工作现场
    
    4. 一步恢复工作现场，恢复的同时把stash内容也删了
        git stash pop     
    5. stash的其它命令
        git stash apply stash@{0}    恢复指定的工作现场
        git stash show -p stash@{0}   对比指定工作现场与当前所发生的改变
        git stash show -p            对比最近工作现场与当前所发生的改变
        git stash save xxxxx         带有指定信息的存储工作现场
        git stash drop  stash@{0}    抛弃指定的工作现场
        gut stash branch develop_test  以当前存储的工作现场创建并切换到分支 develop_test

        
##### Feature分支

    1. 创建一个分支，用于开发下一代星际飞船
        git checkout -b feature-vulcan
    
    2. 飞船开发完成，切换到dev分支准备合并
        git add vulcan
        git commit -m "feature vulcan"
        git checkout dev
    
    3. 经费不足，新功能取消
        git branch -d feature-vulcan    (失败，未经合并的分支)
        git branch -D feature-vulcan    强制删除

##### 多人协作

    1. 查看远程库信息
        git remote
        git remote -v       查看更详细的信息(Git库远程地址)
        
    2. 推送分支
        git push origin master
        git push origin dev
        
        master分支是主分支，因此需要时刻与远程同步；
        dev分支是开发分支，团队所有成员都需要在上面工作，所以也需要与远程同步；
        bug分支只用于在本地修复bug，就没有必要推送到远程了，除非老板要看你每周到底修复了几个bug；
        feature分支是否推送到远程，取决于你是否和你的小伙伴合作在上面开发

    3. 抓取（克隆）分支
        git clone https://github.com/googlesamples/android-ndk.git
        
        默认情况下，clone下来时，本地只能看到master分支
        需要从dev分支开发，在本地创建dev分支：
        
        git checkout -b dev origin/dev
        
    4. 指定本地dev分支与远程origin/dev分支的链接
        git branch --set-upstream dev origin/dev
    
    5. 从服务器拉取最新内容
        git pull
        
##### 标签管理

    1. 创建标签
        git tag v1.0
    
    2. 查看所有标签
        git tag
        
    3. 从某次提交创建标签
        git tag v0.9 82283
    
    4. 带说明文字
        git tag -a v0.1 -m "version 0.1 released" 2323
                -a 指定标签名
                -m 指定说明文字
        
        git show v0.1   查看说明文字
        
    5. 删除标签
        git tag -d v1.0
        
    6. 推送标签到远程
        git push origin v1.0
        
    7. 从远程库删除
        git push origin :refs/tags/v1.0
        
##### 忽略文件

    1. 忽略掉Git工作目录中不需要提交的文件，如.class文件。创建.gitignore文件即可

    2. 某些时候，无法添加文件到Git
        git add -f demo.java        -f 强制添加

    3. 如果.gitignore写的有问题，使用命令检查 
        git check-ignore -v demo.java       该命令会告知.gitignore哪里写得有问题
        
##### 其他配置
	
	1. 避免每次pull/push都输入账户名与密码。在config文件中加入如下命令
		[credential]   
     		helper = store

	2. Git默认对大小写不明感，加入如下配置命令不忽略大小写
		git config core.ignorecase false  

感谢好朋友小滔滔无私的付出，文章大部分是引用自他：[他的Github](https://github.com/chentao7v)
