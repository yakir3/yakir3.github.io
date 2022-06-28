---
title: Git Flow 工作流
categories:
  - Linux
tags:
  - git
abbrlink: '539'
date: 2022-05-04 21:14:23
---
### 一、gitflow：版本分支管理策略（相当于对git的包装）
1. GitFlow描述
- 常用分支包括master、develop、feature、release、hotfix（support分支不常用）
- 其中master、develop是远程分支，feature、release、hotfix是本地分支。
   - 远程分支是指需要push到gitlab、github远程仓库中
   - 本地分支指开发人员的本地开发时使用的git版本控制环境

<!--more-->
2. GitFlow流程图及描述理解

{% asset_img gf1.png %}
{% asset_img gf2.png %}

- master：主干分支 

{% asset_img gf3.png %}

   - 最稳定分支、功能完整、可随时发布线上环境（只读分支，只能从hotfix/release合并 不能修改）
   - 在master分支上的推送应该打tag记录追溯；

- develop：开发分支 ß
   - 功能最新最全的分支，基于master分支克隆（仅首次克隆）；
   - feature分支本地自测通过后合并到develop分支然后删除；
   - 收集所有上线功能后（包含所有发布到下一个release的代码）从delevop拉去release分支进行提测；
   - release/hotfix分支上线完毕，合并到develop并推送；

- feature：功能开发分支 

{% asset_img gf4.png %}

   - 开发某部分新功能，基于develop分支克隆，功能开发完毕且本地自测通过（编译完成且无异常）合并到develop分支；
   - 可存在多个feature分支，即团队多人同时开发创建多个临时分支，功能完成后可选删除；

- release：测试分支 

{% asset_img gf5.png %}

   - 用于提交给测试人员进行功能测试，基于feature分支合并到develop之后，从develop分支克隆；
   - 测试过程发现BUG在本分支进行修复，修复时创建修复分支bugfix-*，修复完所有bug上线后一次性合并到develop/master分支并推送（完成功能），推送master分支时打tag；
   - 临时分支，功能上线后可选删除（开启release测试后，不允许develop分支新功能继续合并到release分支，新功能需放到下一个release测试及发布）；

- hotfix：补丁分支 

{% asset_img gf6.png %}

   - 基于master分支克隆，主要用于线上版本进行BUG修复；
   - 修复bug后合并到develop/master分支并推送（所有hotfix分支的修改会进入到下一个release），推送master分支时打tag；
   - 临时分支，修复bug后可选删除

3. 开发准则与约定
- 准则
   - 除了源码相关的东西之外，其他build产生的东西（如：maven的target文件夹，.idea文件夹等），均不能提交进入源码仓库，添加到.gitignore文件中忽略掉
   - 开发人员要严格按照我们约定的gitflow版本分支管理流程切换到指定分支，开发相应的功能
   - 任务完成后需要根据测试用例经过严格的自测才能推送develop，严禁将编译不通过，提交不完全的代码推送到远程分支
- 约定：
   - 主分支名称：master    主开发分支：develop
   - 标签（tag）名称：v*. release，其中“*” 为版本号，“release” 小写，如：v1.0.0. release
   - 新功能开发分支名称：feature-*，其中“*” 为对应jira（Aone）上的任务编号
   - 发布分支名称：release-*，其中“*” 为版本号，“release”小写，如：release-1.0.0，release分支上修复bug的分支名称为bugfix-*
   - master的bug修复分支名称：hotfix-*，其中“*” 为对应jira（Aone）上的任务编号

### 二、测试部分

- 本地git flow init 初始化仓库，提交develop分支

{% asset_img gf7.png %}
{% asset_img gf8.png %}

- 提交到远程测试用github仓库（使用ssh公钥认证方式），可以看到develop分支已有第一次提交

{% asset_img gf9.png %}

- 到tmp目录下新建工作目录，并clone下远程仓库到本地（模拟本地开发）

{% asset_img gf10.png %}

- 初始化仓库后，拉取develop分支到本地进行开发。此时可以切出feature分支进行功能开发

{% asset_img gf11.png %}
{% asset_img gf12.png %}

- 开发功能提交后进行提交到远程仓库并track远程分支。后续可以git push持续提交

{% asset_img gf13.png %}
{% asset_img gf14.png %}

- 功能开发完成后，将分支合并到develop分支并删除本地feature分支（加上-F参数可以同时删除远程分支）

{% asset_img gf15.png %}
finish提交后即将新功能合并到develop分支，完成新功能开发。后续即执行release发布操作，步骤类似

