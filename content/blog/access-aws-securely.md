+++
title = "安全地访问 AWS 资源"
date = 2023-06-18
draft = true

[taxonomies]
tags = ["aws", "security", "auth", "iam", "eks", "oidc"]
+++

AWS 作为企业广泛使用的云服务供应商，安全是其最高优先级。本文结合实际使用经验介绍部署在 EC2 或 EKS 上的应用如何安全地访问 AWS 资源。

<!-- more -->

我们知道访问 AWS 服务的 API 都需要提供 credential，常见的做法是直接配置 access key 和 secret key，本质上类似于我们访问网站提供 username, password，其中 access key 对应 username，secret key 对应 password。由于 AWS user 的 credentials 没有过期时间（属于静态 token），所以一旦泄漏安全风险非常高，安全上一般会采取定期 rotate 措施来增强安全。一个 AWS user 可以创建两个 credentials，就是用于 rotate 时，先新建一个 credential，rotate 期间两个 credentials 同时有效，等所有配置 credential 的程序或脚本都 rotate 完之后，将老的 credential 删掉。

尽管 AWS 提供这种 rotate credential 能力，但实际使用该机制进行认证的场景，几乎都很少能贯彻执行定期 rotate 策略，因为手动 rotate 很容易出错，并且往往会涉及跨团队协调所以非常费劲。更安全的做法是使用 IAM Role 的临时 credential。

## EC2 上访问 AWS 资源

EC2 上可以 attach 一个 IAM Role 或 Instance Profile，运行在 EC2 之上的程序可以不用显式配置 credential，即可使用 EC2 的 IAM Role。为了让 EC2 上的程序使用该 Role，需要额外配置

Role 上需要加上 ec2 的 trust relationship，表示允许 ec2 服务 assume 该 role。ec2 上部署的服务即可使用 ec2 上 attach 的 role。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

其原理是进程访问 metadata server 169.254.169.254，获取一个临时 token，该 token 是 ec2 service 分配的，然后可以 assume 成 instance 上的 role。

AWS SDK 里面实现了获取该 token 的流程，并且在 token 过期之前会自动刷新 token。

## EKS 上访问 AWS 资源

K8s 的出现给上述 EC2 的 Auth 机制提出了新的挑战，因为 k8s 上一个 node 上往往会运行很多 pod，不同 pod 所要求的权限不一样，如果使用 EC2 Role 授权机制，则 node role 上需要 Attach 一个所有 pod 的权限合集，而且任意 pod 都可以使用这些权限。

有 EKS 官方的 IAM Role for Service Account (IRSA) 出来之前，社区有 [kube2iam](https://github.com/jtblin/kube2iam) 和 [kiam](https://github.com/uswitch/kiam) 两个开源项目解决该问题。

IAM Role for Service Account (IRSA) 利用的是 Web Identity，该身份是由 k8s 提供的

Web Identity 实际是 EKS 的 master 作为 OIDC 的 Identity Provider (IdP)

使用时需要在 IAM Identity Providers 里面添加一个 IdP

为了让 EKS 里面的 Pod 能够使用某个 Role 访问 AWS 资源，需要给该 Role 增加一个 trust relationship policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::${accountId}:oidc-provider/${IdP}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "${IdP}:sub": "system:serviceaccount:${namespace}:${serviceAccountName}"
                }
            }
        }
    ]
}
```

其中 `${IdP}` 是配置在 IAM Identity Providers 里面的 IdP，格式是 `oidc.eks.${region}.amazonaws.com/id/ABCDEFGHIJKLMNOPQRSTUVWXYZ123456`。`${accountId}` 是 AWS account ID，`${namespace}` 是 k8s 里面的 namespace，用于将 Role 限定在特定的 namespace 使用，`${serviceAccountName}` 是 k8s 里面的 ServiceAccount Name。


web identity 使用下面两个环境变量：
- `AWS_ROLE_ARN`
- `AWS_WEB_IDENTITY_TOKEN_FILE`

https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/
