+++
title = "ARM Linux 上构建 x86-64 Docker 镜像"
date = 2023-06-11

[taxonomies]
tags = ["docker", "nixos"]
+++

近期申请了一台 Mack Book Pro M1 工作机，为了方便以后换机器配置环境的麻烦，装了个 [nixo](https://nixos.org/) 虚拟机。期间遇到一些小问题，但整体还是很满意的。写个文档记录一下如何在 arm64 nixos 上构建 x86-64 的 Docker 镜像。

<!-- more -->

## TL;DR

1. 在 /etc/nixos/configuration.nix 配置里面加上下面的配置


```nix
boot.binfmt.emulatedSystems = [ "x86_64-linux" ];

environment.SystemPackages = with pkgs; [ docker docker-buildx ];

virtualisation.docker.enable = true;
```

2. 创建 amd64 builder

```shell
docker buildx create --platform=amd64 --name my-amd64-builder --bootstrap
```

3. 构建 x86-64 镜像

```shell
docker buildx build --platform=linux/amd64 --builder my-amd64-builder --push -t my-image .
```


## Detailed

Docker 的跨平台构建是通过 BuildKit 扩展实现的，而跨平台则依赖 QEMU 模拟器。nixos 里面启用 QEMU 模拟，只需要在 /etc/nixos/configuration.nix 里面加上 `boot.binfmt.emulatedSystems = [ "x86_64-linux" ];`。docker-buildx 是 BuildKit 的跨平台扩展，nixos 下面声明式安装 docker-buildx：`environment.SystemPackages = with pkgs; [ docker docker-buildx ];`。如果之前没安装配置过 Docker，则还需要启用 docker 服务：`virtualisation.docker.enable = true;`，配置完 /etc/nixos/configuration.nix 后，运行 `sudo nixos-rebuild switch` 就安装并启用了 docker 服务。

docker buildx 默认有一个当前架构的 builder，可以通过 `docker buildx ls` 列出所有 builder。`docker buildx create --platform=amd64 --name my-amd64-builder --bootstrap` 用于创建一个新的 builder，不再使用的 builder 可以通过 `docker buildx rm my-amd64-builder` 方式删掉。