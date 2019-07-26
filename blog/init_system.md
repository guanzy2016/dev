# 容器初始化进程

容器平台中，通过挂载文件系统，在可读写层直接运行业务程序，共享宿主机内核，因而缺少初始化系统管理业务进程，主要有两个问题：

- 子进程无法优雅退出

docker 向容器初始化进程发出 SIGTERM 信号后，pid 为 1 的父进程不能将信号传递到子进程（正常 pid 1 的 init，systemd 等进程可以处理信号，传向子进程），
则子进程无法退出，容器无法退出，需要向父进程发 SIGKILL 信号强制退出。

- 子进程变僵尸进程

当业务进程创建了其他子进程后，业务进程退出，导致其他子进程变为孤儿进程，linux 中默认会由 pid 1 号进程接管回收，容器中的 1 号进程缺实现不了接管孤儿进程。

上面两个问题可以通过为容器配置 sbin-init，systemd，dumb-init 作为初始化进程，负责管理业务进程。

## sbin-init

sbin-init 是传统的初始化系统，运行脚本主要在 /etc/rc.d/init.d 下，当 init 命令启动时，它成为系统里所有自动启动程序的父程序或者祖父（grandparent）程序。首先，它运行 /etc/rc.d/rc.sysinit 脚本设置环境路径、启动 swap、检查文件系统并执行所有系统初始化所需的其他步骤 （[出处](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/6/html/installation_guide/s2-boot-init-shutdown-init)）。

sysVinit 的启动形式目前大部分已经被 systemd 所取代，并发启动，耗时更短 [systemd-replaces-init](https://www.tecmint.com/systemd-replaces-init-in-linux/)。

## systemd

systemd 提供的是系统和服务的管理器，用于运行系统程序，可参考如下 [描述](https://wiki.archlinux.org/index.php/Systemd_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87))：

systemd 是一个 Linux 系统基础组件的集合，提供了一个系统和服务管理器，运行为 PID 1 并负责启动其它程序。功能包括：支持并行化任务；同时采用 socket 式与 D-Bus 总线式激活服务；按需启动守护进程（daemon）；利用 Linux 的 cgroups 监视进程；支持快照和系统恢复；维护挂载点和自动挂载点；各服务间基于依赖关系进行精密控制。systemd 支持 SysV 和 LSB 初始脚本，可以替代 sysvinit。除此之外，功能还包括日志进程、控制基础系统配置，维护登陆用户列表以及系统账户、运行时目录和设置，可以运行容器和虚拟机，可以简单的管理网络配置、网络时间同步、日志转发和名称解析等。

## dumb-init

[dumb-init](https://github.com/Yelp/dumb-init) 是 Yelp 开发的，主要应用于容器平台的精简初始化系统，主要用于解决上述容器环境缺少初始化进程管理业务程序的问题，但 dumb-init 作为精简初始化系统相较于 sbin-init 和 systemd，系统初始化的功能还不完备，如 dumb-init 无法启动 dbus，dumb-init 作为初始化进程，容器内的 systemd 也无法使用。

## k8s pause container

k8s 以 pod 为最小管理单元，将一组共享命名空间的容器放在 pod 里，其中 pause 容器是 pod 中其他容器的父容器，其他容器以子进程的方式加入到 pause 命名空间以共享，pause 中运行的代码解决了前文描述的两个问题，传递进程信号，接管孤儿进程，避免出现僵尸进程。

进程命名空间共享 PodShareProcessNamespace 在 beta 版本中默认开启，pod yaml 中需要配置：
[shareProcessNamespace](https://v1-12.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/share-process-namespace/)：

```
apiVersion: v1
kind: Pod
metadata:
 name: demo4
 annotations:
    "beyond-ac.admission.kubernetes.io/include-lxcfs" : "true"
 labels:
  name: richcontainer
spec:
 shareProcessNamespace: true
 containers:
 - name: richcontainer-demo
   image: centos-rich:1.0
   command: ["sleep"]
   args: ["1000"]
   securityContext:
      privileged: true
```

创建 pod 后查看容器：

```
[root@gzy-node ~]# docker ps |grep demo
0d62cffe6679        816fb7a0c95c           "sleep 1000"             15 minutes ago      Up 15 minutes                           k8s_richcontainer-demo_demo4_default_5d00ca16-af4c-11e9-ab06-525400671df1_0
7cd4472f2d94        k8s.gcr.io/pause:3.1   "/pause"                 15 minutes ago      Up 15 minutes                           k8s_POD_demo4_default_5d00ca16-af4c-11e9-ab06-525400671df1_0
[root@gzy-node ~]#
[root@gzy-node ~]# docker inspect 0d62cffe6679 |grep 7cd4472f2d94
        "ResolvConfPath": "/var/lib/docker/containers/7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2/resolv.conf",
        "HostnamePath": "/var/lib/docker/containers/7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2/hostname",
            "NetworkMode": "container:7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2",
            "IpcMode": "container:7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2",
            "PidMode": "container:7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2",
                "io.kubernetes.sandbox.id": "7cd4472f2d94e549d9802838fece28434a1b26a4e9939ce7d97fc8cb84ac23b2",
```

可以看到，主要运行业务程序的容器共享 pause 容器的:

- 网络命名空间 Network
- 进程间通信命名空间 Ipc
- 配置生效的进程命名空间 Pid

此时 demo 主容器以子进程的方式被加入到 pause 容器命名空间中
[pause container](http://dockone.io/article/2785), 容器中的 1 号进程为 pause，转发进程信号，接管孤儿进程由 pause 容器负责：

```
[root@gzy-vm kubevirt_file]# kubectl exec -it demo4 bash
[root@demo4 /]# ps -elf
F S UID        PID  PPID  C PRI  NI ADDR SZ WCHAN  STIME TTY          TIME CMD
4 S root         1     0  0  80   0 -   253 sys_pa 02:23 ?        00:00:00 /pause
4 S root         7     0  0  80   0 -  1090 hrtime 02:23 ?        00:00:00 sleep 1000
4 S root        13     0  0  80   0 -  2955 do_wai 02:23 pts/0    00:00:00 bash
4 R root        28    13  0  80   0 - 12935 -      02:24 pts/0    00:00:00 ps -elf
```

所以，从 k8s pod 的角度， dumb-init 功能可完全被 pause 容器实现，不需要额外再为容器配置 dumb-init 初始化进程。