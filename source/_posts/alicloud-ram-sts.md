---
title: Alicloud-RAM 与 STS 权限
categories:
  - Alicloud
tags:
  - RAM
abbrlink: fdc
date: 2022-03-20 23:37:32
---
### 一、前言
当通过OpenApi接口来调用云资源时（即替代控制台的操作），当前的方式有两种：

- 通过 AK+SK 方式直接调用，只要该AK所属的账号有相关权限即可调用对应资源（授权分为系统策略和自定义策略）
- Alicloud账号（RAM用户）/Alicloud服务（ECS等）/身份提供商（SSO） 通过扮演角色获取角色的临时令牌（即通过调用AssumeRole接口），通过该临时令牌（临时令牌可设定会话时间），通过 STS接口获取到的 临时AK+临时SK+临时STSToken 进行调用对应资源

### 二、官方概念介绍

1. STS概念
- AlicloudSTS（Security Token Service）是Alicloud提供的一种临时访问权限管理服务。RAM提供RAM用户和RAM角色两种身份。其中，RAM角色不具备永久身份凭证，而只能通过STS获取可以自定义时效和访问权限的临时身份凭证，即安全令牌（STS Token）

<!--more-->
建议阅读：[https://help.aliyun.com/document_detail/28756.html](https://help.aliyun.com/document_detail/28756.html)

2. RAM概念
- RAM用户：身份实体，可访问Alicloud资源的账号或程序；创建时可选择登录场景、AccessKey场景（通过程序调用API）。
- **RAM角色**：虚拟用户，向信任的RAM实体账号进行授权（根据STS令牌颁发短时有效的临时访问token）；创建角色后会生成Arn描述符（角色的描述符：每个RAM角色存在唯一值且遵循Alicloudarn命名规范）。
- RAM权限策略：一组权限集，使用简单的Policy语法进行描述（分为系统策略和自定义策略）；权限策略是实际细分授权资源集、操作集、授权条件的描述。

{% asset_img ram1.png %}

> 创建RAM角色时有三种类型：
> - **Alicloud账号**：允许RAM用户所扮演的角色。扮演角色的RAM用户可以属于自己的Alicloud账号，也可以属于其他Alicloud账号。此类角色主要用来解决跨账号访问和临时授权问题
> - **Alicloud服务**：允许云服务所扮演的角色。此类角色主要用于授权云服务代理您进行资源操作（服务又分为两种）
>    - 普通服务角色：您需要自定义角色名称，选择受信服务，并自定义权限策略
>    - 服务关联角色：您只需选择受信的云服务，云服务会自带预设的角色名称和权限策略
>    - 两种服务角色没太大区别，服务关联角色会多一个预设的配置（一般服务角色用户Alicloud跨服务间的调用，例如ECS的授予/收回RAM角色功能、RDS云服务调用KMS角色加密等，从某个云产品调用另一个云产品的授权）
> 
{% asset_img ram2.png %}
> - **身份提供商**：允许可信身份提供商下的用户所扮演的角色。此类角色主要用于实现与Alicloud的单点登录（SSO）
> 
> **常用的RAM角色一般为创建 Alicloud账号 方式（OSS官方推荐使用）**
> - 授权RAM角色介绍：[https://help.aliyun.com/document_detail/116819.html](https://help.aliyun.com/document_detail/116819.html)
> - OSS官方推荐使用 Alicloud账号 方式：[https://help.aliyun.com/document_detail/100624.html](https://help.aliyun.com/document_detail/100624.html)


### 三、创建STS角色，自定义授权OSS功能测试
1）测试账号信息
账号：devops_test@xxx.onaliyun.com
AK：xxxxx
SK：xxxxx
ARN：acs:ram::xxxxx:role/xxx-sts
OSS Bucket名称：oss-test
OSS授权目录：dir111/dir111_secondline1/

2）进行授权

- 创建RAM用户（子账号），生成AK SK （此步骤忽略）
- 测试账号添加STS权限

{% asset_img ram3.png %}

- 添加权限策略，使用自定义策略授权（OSS官方示例Policy：[https://help.aliyun.com/document_detail/266627.html](https://help.aliyun.com/document_detail/266627.html)）

{% asset_img ram4.png %}

- 添加RAM角色并授权Policy

{% asset_img ram5.png %}

{% asset_img ram6.png %}

3）测试验证（控制台无法登录RAM账号验证权限情况下，可以使用ossutil或ossbrowser工具进行验证）

- ossutil使用：[https://help.aliyun.com/document_detail/50451.html](https://help.aliyun.com/document_detail/50451.html)

{% asset_img ram7.png %}

- ossbrowser使用：[https://help.aliyun.com/document_detail/92268.html](https://help.aliyun.com/document_detail/92268.html)


4）验证列举和其他相关权限无误后，将ARN信息提供研发即可

> 权限流程：
> 1. 客户端程序/调用端发起扮演角色，此时在进入实际要获取的角色权限前，需要通过调用AssumeRole接口返回STS凭证（调用STS接口需要**AliyunSTSAssumeRoleAccess**权限，因此对应RAM账号需要授权该系统策略）
> 1. 通过返回的STS临时凭证（临时AK+临时SK+临时token）发起相关云资源接口的调用
> 1. 客户端使用STS发起调用时，会验证两个部分的权限策略Policy （注意：最后的权限取这两个权限Policy的交集）
> - STS扮演的角色本身授权的权限策略是否拥有对应云资源的权限（系统或自定义的Policy）
> - SDK/API调用时传入的policy_text参数值，在构造调用请求时传入（[https://help.aliyun.com/document_detail/100624.html](https://help.aliyun.com/document_detail/100624.html)
> {% asset_img ram8.png %}


### 四、结合实际需求

1. 开发提出需求：需要某一个oss bucket的STS ARN信息

2. 需要相关信息：
+ oss bucket具体需授权目录，必须
+ endpoint: oss bucket所属区域，非必须
+ bucket-name: oss bucket名称，必须
+ 调用OSS RAM账号，必须

3. 通过提供信息进行创建RAM角色、Policy策略新建（注意oss 细粒度的策略）、授权策略到RAM角色中，最后将新建的RAM角色的ARN描述符信息提供给研发。
