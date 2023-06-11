+++
title = "如何调试 Docker 容器"
description = ""
date = 2019-12-22

[taxonomies]
tags = ["Docker", "diagnosis"]
+++

[Docker](https://www.docker.com/) 容器作为虚拟化技术的一种，其轻量级的优点同时也带来了一些缺点：不彻底的隔离性给安全带来了隐患，裁减镜像增加了运维调试难度。本文以 [tidb-docker-compose](https://github.com/pingcap/tidb-docker-compose) 为例，谈谈在 Docker 容器出现问题时，如何进行调试排查。

<!-- more -->

在开始之前，可以通过以下命令用 [Docker Compose](https://docs.docker.com/compose/) 在本地快速启动一个 [TiDB](https://github.com/pingcap/tidb) 集群。

```sh
$ git clone https://github.com/pingcap/tidb-docker-compose
$ cd tidb-docker-compose
$ docker-compose pull && docker-compose up -d
```

## 方法 1：通过日志进行排查

当应用出现问题时，最简单直接的方式就是通过应用日志进行排查了。一般容器应用程序都是将日志输出到标准输出，然后被 Docker 收集。对于这种情况，可以通过 `docker logs` 和 `docker-compose logs` 命令查看日志。其中后者是 Docker Compose 提供的，其命令参数直接是 service name，而不是包含目录名字的容器名。所以，对于用 Docker Compose 启动的容器，使用后者更方便，但是这个命令需要在包含 `docker-compose.yml` 的目录下执行。

1. 查看容器运行状态。tidb-docker-compose 运行起来后，通过 `docker ps` 或 `docker-compose ps` 命令查看容器运行状态，如下图所示：

    ![docker ps](/images/docker-ps.png)

2. 查看日志。以下是查看 tispark-master 日志的方法：

    ```sh
    $ docker logs tidb-docker-compose_tispark-master_1
    $ docker-compose logs tispark-master
    ```

    此外，`docker logs` 和 `docker-compose logs` 也支持使用 `-f` 参数实现类似于 `tail -f` 的方式实时查看日志。

    由于 Docker Compose 日志会丢，所以 tidb-docker-compose 将 pd/tikv/tidb 的日志都输出到文件了，可以直接查看 logs 目录里对应的日志文件。

## 方法 2：进入容器环境中排查

如果通过日志无法分析出应用出问题的原因，一般可能就是运行环境问题，又分为[容器正常](#容器正常)和[容器闪退](#容器闪退)两种情况。

### 容器正常

这种情况下，最快捷的排查方式就是进入容器里面查看网络情况、文件权限、环境变量、内核参数等。

可以使用 `docker exec` 命令进入容器环境。相应地，Docker Compose 也提供了 `docker-compose exec` 命令，但不用加 `-it` 参数。

```sh
docker exec -it tidb-docker-compose_tikv0_1 sh
docker-compose exec tikv0 sh
```

![docker exec](/images/docker-exec.png)

### 容器闪退

容器闪退在 Docker Compose 和 [Kubernetes](https://kubernetes.io/) 里面比较常见。不熟悉 Docker 运行原理的同学可能会参照网上搜到的一些零散知识将应用改造成 docker-compose 或 Kubernetes 应用，结果应用程序水土不服。例如，因为环境变量、volume 挂载、权限等问题，刚一启动就崩了。

由于容器一运行就退出，而 `docker exec` 命令的使用对象是一个正在运行着的容器，所以没法通过这种方式去排查原因。

这时，可以尝试让容器以一个能正常运行的命令启动而不直接运行应用程序。最简单的方式就是 `sleep` 了，但 PID 为 1 的 `sleep` 程序可能会导致容器无法被停掉。

但我们可以采用 `tail -f /dev/null` 来替代正常容器的启动命令，这样能保证容器运行起来。然后，再通过前面提到的 `docker exec` 命令进入容器，查看容器网络和环境变量；也可以尝试手动将应用程序以较高的日志级别启动，看看问题出在哪里。目前 [TiDB Operator](https://github.com/pingcap/tidb-operator) 就是采用这种方式来调试出问题的 Pod，
效果还是很不错的。

1. 将应用程序的启动命令更改为 `tail -f /dev/null`。本例中将 `docker-compose.yml` 文件中 TiDB 的启动命令更改为 `tail -f /dev/null`。

    ![tail -f /dev/null](/images/tail-f-dev-null.png)

2. 使用以下命令删掉容器（本例中为 `tidb`），并重新启动。

    ```sh
    $ docker-compose stop tidb
    $ docker-compose rm tidb
    $ docker-compose up -d
    ```

    这样就解决了容器闪退的问题。

3. 通过 `docker exec` 命令进入容器，然后检查容器中的环境（例如网络、文件、环境变量等），再手动启动服务排查具体原因。

    ```sh
    $ docker-compose exec tidb sh
    # /tidb-server --store=tikv \
    --path=pd0:2379,pd1:2379,pd2:2379 \
    --config=/tidb.toml \
    --L=debug
    ```

    ![docker exec manual start](/images/docker-exec-manual-start.png)

## 方法 3：在宿主机上使用 GDB 或 perf 进行排查

对于 C、C++、[Rust](https://www.rust-lang.org/) 这种偏底层的应用程序，[GDB](https://en.wikipedia.org/wiki/GNU_Debugger) 和 [perf](https://en.wikipedia.org/wiki/Perf_(Linux)) 是很有效的线上问题排查工具。

在使用 GDB 调试运行中的应用程序时，需要应用程序的 PID 及其 binary。而在宿主机上能够直接看到容器里应用程序的 PID，所以可以在宿主机上直接通过 GDB/perf 工具对容器里的进程进行调试。不过只有[高版本的 GDB 才支持 namespace](https://sourceware.org/bugzilla/show_bug.cgi?id=18368)，一些版本相对较旧的系统（如 CentOS 系统）下就没法直接在宿主机上通过 GDB 调试 Docker 容器中的应用。

要在宿主机上通过 GDB 调试容器里的进程，需要先获取应用的二进制程序，以及应用在宿主机上的 PID。下面以调试 tikv1 为例，其中 `31842` 是 tikv1 在宿主机上的 PID：

```sh
$ docker cp tidb-docker-compose_tikv1_1:/tikv-server tikv-server
$ ps aux | grep tikv1
$ sudo gdb tikv-server -p 31842 -batch -ex "thread apply all bt" -ex "info threads"
```

对于 perf 也是类似的，直接找到容器应用在宿主机上的 PID，然后用该 PID 对容器进程进行调试。

## 方法 4：连接容器 namespace 进行排查

有时通过 `docker exec` 命令进入容器了，但是很多人为了减小容器镜像体积和攻击面而裁减掉了许多 Linux 下的基本工具，例如 `ps` 命令在很多容器里面就没有，更别说 [curl](https://curl.haxx.se/) 这类高级网络工具了，所以进到容器里面了也没法查。当然有些人可能会根据容器的基础镜像利用其包管理工具就地安装需要用的调试工具，这在某些情况下也是行之有效的。

但是特殊情况下，容器里面可能会没有包管理器或甚至基础镜像是 [scratch](https://docs.docker.com/develop/develop-images/baseimages/#create-a-simple-parent-image-using-scratch)，这时由于连基本的 shell 都没有， `docker exec` 命令也没法使用。但这是不是意味着应用程序除了通过日志外完全无法调试呢？答案显然是否定的。

从 Docker 华丽的外表下回到容器的本质上，Linux 上的容器是利用 Linux 内核提供的 [namespace](https://en.wikipedia.org/wiki/Linux_namespaces) 和 [cgroup](https://en.wikipedia.org/wiki/Cgroups) 实现的应用运行环境。namespace 包括 net、PID、uts、ipc、user、mount，将不同容器运行环境相互隔离，但是不同容器又可以共享某些 namespace。比如，Docker 早期支持的 [links](https://docs.docker.com/network/links/) 就可以让多个容器共享 net namespace；Kubernetes 里的 [Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/) 也是共享 net namespace，Pod 甚至还可以共享 [PID namespace](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/)。

而**调试容器应用的难点**正在于容器环境是一个隔离的环境，这个环境是一个孤岛，里面什么东西也没有。如果我们能打通这个孤岛，调试难的问题也就迎刃而解。

明白了原理，我们就来实际操作一下。

1. 进入 namespace。

    以下示例用 [tidb-debug](https://github.com/pingcap/tidb-docker-compose/blob/master/docker/debug/Dockerfile) 镜像作为调试工具镜像。该镜像里面包含很多常用工具，镜像体积相对比较大，实际调试时可以是任何包含所需工具的镜像。注意下面命令中启动 debugger 容器时加了 `SYS_PTRACE` 参数，这是因为 GDB 和 strace 之类的调试工具需要 ptrace 权限。

    ```sh
    $ docker run -it --rm --name=debugger --cap-add=SYS_PTRACE \
    --PID=container:tidb-docker-compose_tikv0_1 \
    --network=container:tidb-docker-compose_tikv0_1 \
    uhub.ucloud.cn/pingcap/tidb-debug
    ```

    通过执行以上命令，即进入了 tikv0 的 PID 和 network namespace。

2. 根据具体问题进行调试。

    本例中，可以使用 tidb-debug 容器中的各种网络工具对 tikv0 容器进行调试，也可以使用 GDB/perf 之类的工具对 tikv-server 进程进行调试。

    如果要需要 tikv-server 的二进制程序，可以利用 `docker cp` 命令经主机中转一下拷贝到 tidb-debug 容器中。但是由于 PID 已经共享，实际从 tidb-debug 容器中就能看到 PID 为 1 的 tikv-server 进程，而 `/proc/1/exe` 即是 tikv-server 的二进制程序。

    ```sh
    # ldd /proc/1/exe
    # gdb /proc/1/exe 1 -batch -ex "thread apply all bt" -ex "info threads"
    # curl http://pd0:2379/pd/api/v1/stores
    ```

    实际上 tidb-docker-compose 项目中提供了一个方便的[脚本](https://github.com/pingcap/tidb-docker-compose/blob/master/tools/container_debug)用于调试诊断。

    ```sh
    $ tools/container_debug -s tikv2 -p /tikv-server
    # gdb /tikv-server 1 -batch -ex "thread apply all bt" -ex "info threads"
    # ./run_flamegraph.sh 1
    ```

    社区 [kubectl-debug](https://github.com/aylei/kubectl-debug) 和 [docker-debug](https://github.com/zeromake/docker-debug) 也是利用这个原理实现了 K8s 和 Docker 上的容器便捷调试工具。

## 参考文档

* [debugging-docker-containers-with-gdb-and-nsenter](https://blog.wnohang.net/index.php/2015/05/05/debugging-docker-containers-with-gdb-and-nsenter/)
