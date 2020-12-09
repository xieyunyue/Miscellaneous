# build ARM image on x86
支持CI自动化build多平台镜像，需要能够在x86环境下支持ARM指令。下面是一种支持的方式：
https://blog.csdn.net/whatday/article/details/86776463  
第一步是要模拟出一个 ARM 环境。当然，我们可以用 QEMU 直接开一个 ARM 架构的完整 Linux 虚拟机，然后在里面运行 Docker 构建镜像。但是这样做的问题是，
需要额外管理一个 Docker，难以将系统资源在主系统和虚拟机之间灵活分配，并且难以使用脚本自动化，即难以整合到 CI、CD 中。

更好的方案是 qemu-user-static，是 QEMU 虚拟机的用户态实现。它可以直接在 amd64 系统上运行 ARM、MIPS 等架构的 Linux 程序，将指令动态翻译成 x86 指令。
这样 ARM 系统环境中的进程与主系统的进程一一对应，资源分配灵活，并且易于脚本自动化。

但是还有一个问题：当 ARM 进程尝试运行其它进程时，qemu-user-static 并不会接管新建的进程。如果新的进程仍然是 ARM 架构，那么 Linux 内核就无法运行它。
因此，需要开启 Linux 内核的 binfmt 功能，该功能可以让 Linux 内核在检测到 ARM、MIPS 等架构的程序时，自动调用 qemu-user-static。开启该功能，
并且注册 qemu-user-static 虚拟机后，运行 ARM 程序就和运行 x86 程序一样，对用户来说毫无差别。

在Intel CPU上最简单的使用QMEU模拟ARM环境只需要两步：  
1.Base镜像里必须存在一个针对ARM的QEMU Intel二进制文件  
2.QEMU必须已经在CI的build代理中注册  

所以查看很多社区镜像支持跨平台build arm镜像时，都会看到这一句
```
$ docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
```
通过这个启动这个容器去注册ARM的处理器
https://github.com/multiarch/qemu-user-static/blob/master/README.md#binfmt_misc-register
multiarch/qemu-user-static镜像执行注册脚本，当运行容器时，为所有支持的处理器注册下面的/proc/sys/fs/binfmt_misc/qemu-$arch文件，除了当前的处理器。
由于/proc/sys/fs/binfmt_misc在主机和容器内部是通用的，注册脚本会修改主机上的文件。按照我的理解，这实际上就是开启了 内核的binfmt功能。

第一步QEMU的二进制文件，这篇文章里https://blog.hypriot.com/post/setup-simple-ci-pipeline-for-arm-images/ 用了一个已经包含二进制文件的base镜像。
```
FROM resin/rpi-raspbian
```

当前也可以直接把二进制文件拷贝到镜像里。

```
curl -sSL https://github.com/multiarch/qemu-user-static/releases/download/$(QEMUVERSION)/x86_64_qemu-$(QEMUARCH)-static.tar.gz

dockerfile
ARG ALPINE_ARCH=amd64
FROM ${ALPINE_ARCH}/alpine:3.11
COPY qemu-aarch64-static /usr/bin/

ARG ARCH=amd64

RUN apk add --no-cache ca-certificates

ADD k8s-keystone-auth-${ARCH} /bin/k8s-keystone-auth
```
