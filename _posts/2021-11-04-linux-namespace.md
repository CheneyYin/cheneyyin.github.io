---
layout: post
title: "Linux Namespace"
subtitle: "介绍Linux的几种常见Namespace"
date: 2021-11-04
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags: 
 - Linux
 - Linux Namespace
 - Docker
 - container
 - nsenter
 - unshare
---

# 概况

| 名称                             | 作用                                   | 版本   |
| -------------------------------- | -------------------------------------- | ------ |
| Mount (mnt)                      | 隔离挂载点                             | 2.4.19 |
| Process ID (pid)                 | 隔离进程ID                             | 2.6.24 |
| Network (net)                    | 隔离网络设备、端口号等                 | 2.6.29 |
| Interprocess Communication (ipc) | 隔离System V IPC和POSIX message queues | 2.6.19 |
| UTS Namespace (uts)              | 隔离主机名和域名                       | 2.6.19 |
| User Namespace (user)            | 隔离用户和用户组                       | 3.8    |
| Control Group (Cgroup)           | 隔离Cgroups根目录                      | 4.6    |
| Time Namespace                   | 隔离系统时间                           | 5.6    |

# Mount Namespace

**Mount Namespace**是Linux内核实现的第一个namespace，自**2.4.19**版本开始加入。

**Mount Namespace**可以用来隔离不同的进程或进程组看到的挂载点。可以实现在不同的进程中看到不同的挂载目录。因此，**Mount Namespace**可以使容器内只能看到自己的挂载信息，在容器内的挂载操作不会影响主机的挂在目录。

## 实例演示

使用`unshare`命令演示实例。

```shell
sudo unshare --mount --fork /bin/bash
```

`--mount`表示启用**mount namespace**，不从父进程继承挂载命名空间。

`--fork`表示在执行`/bin/bash`前进行fork 出子进程。使用`--fork`和不使用的区别可以反映在进程树上，

```shell
pstree -ps 10883 #10883进程是通过--fork
systemd(1)───alacritty(9378)───zsh(9380)───sudo(10881)───unshare(10882)───zsh(10883)

pstree -ps 17378 #17378进程没有通过--fork
systemd(1)───alacritty(9378)───zsh(9380)───sudo(17377)───zsh(17378)
```



接下来，在挂载点隔离的`/bin/bash`进程中，执行挂载操作。

```shell
mount -t tmpfs -o size=20m tmpfs /tmp/tmpfs
```

挂在成功后，在`/tmp/tmpfs`目录下创建`hello`文件。

```shell
touch /tmp/tmpfs/hello
```

 在挂载点隔离进程下查看已经挂载的目录信息，

```shell
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       903G   23G  835G   3% /
dev             5.8G     0  5.8G   0% /dev
tmpfs           5.8G   40M  5.8G   1% /dev/shm
run             5.8G  1.5M  5.8G   1% /run
tmpfs           1.2G   76K  1.2G   1% /run/user/1000
/dev/sda1       300M  288K  300M   1% /boot/efi
tmpfs           5.8G  3.8M  5.8G   1% /tmp
tmpfs            20M     0   20M   0% /tmp/tmpfs
```

最后一行为刚刚创建的**20M**大小的tmpfs。

查看该目录`/tmp/tmpfs`，目录下存在`hello`文件。

```shell
ls /tmp/tmpfs
hello
```

查看进程下的**namespace**信息。

```shell
ll /proc/self/ns
total 0
95041 0 dr-x--x--x 2 root root 0 11月 17 10:55 .
95040 0 dr-xr-xr-x 9 root root 0 11月 17 10:55 ..
95053 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 cgroup -> 'cgroup:[4026531835]'
95048 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 ipc -> 'ipc:[4026531839]'
95052 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 mnt -> 'mnt:[4026532506]'
95046 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 net -> 'net:[4026531992]'
95049 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 pid -> 'pid:[4026531836]'
95050 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 pid_for_children -> 'pid:[4026531836]'
95054 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 time -> 'time:[4026531834]'
95055 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 time_for_children -> 'time:[4026531834]'
95051 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 user -> 'user:[4026531837]'
95047 0 lrwxrwxrwx 1 root root 0 11月 17 10:55 uts -> 'uts:[4026531838]'
```

其中挂载命名空间编号为`mnt:[4026532506]`。

在主机下查看目录`/tmp/tmpfs`、挂载目录信息、主机**Namespace**信息。

```shell
ll /tmp/tmpfs
total 0
```

```shell
sudo df -h
Filesystem      Size  Used Avail Use% Mounted on
dev             5.8G     0  5.8G   0% /dev
run             5.8G  1.5M  5.8G   1% /run
/dev/sda2       903G   23G  835G   3% /
tmpfs           5.8G   34M  5.8G   1% /dev/shm
/dev/sda1       300M  288K  300M   1% /boot/efi
tmpfs           5.8G  3.5M  5.8G   1% /tmp
tmpfs           1.2G   76K  1.2G   1% /run/user/1000
```

```shell
ll /proc/self/ns
total 0
102923 0 dr-x--x--x 2 cheney cheney 0 11月 17 11:02 .
102922 0 dr-xr-xr-x 9 cheney cheney 0 11月 17 11:02 ..
102935 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 cgroup -> 'cgroup:[4026531835]'
102930 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 ipc -> 'ipc:[4026531839]'
102934 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 mnt -> 'mnt:[4026531840]'
102928 0 lrwxrwxrwx 1 chene y cheney 0 11月 17 11:02 net -> 'net:[4026531992]'
102931 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 pid -> 'pid:[4026531836]'
102932 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 pid_for_children -> 'pid:[4026531836]'
102936 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 time -> 'time:[4026531834]'
102937 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 time_for_children -> 'time:[4026531834]'
102933 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 user -> 'user:[4026531837]'
102929 0 lrwxrwxrwx 1 cheney cheney 0 11月 17 11:02 uts -> 'uts:[4026531838]'
```

由上可知，主机`/tmp/tmpfs`目录下不存在`hello`文件，挂载目录信息下不存在`/tmp/tmpfs`，并且挂载隔离命名空间编号为`'mnt:[4026531840]'`，这与挂载隔离进程下的`mnt:[4026532506]`不同。所以，新建的**Mount Namespace**内挂载点是和外部完全隔离的。

# PID Namespace
**PID Namespace**用于隔离进程的PID，不同**PID Namespace**下的进程可以拥有相同的PID。如下，使用`unshare`命令在新的**PID Namespace**下启动`/bin/bash`，

```shell
sudo unshare --pid --fork /bin/bash
```

执行`ps`命令查看进程PID，
```shell
ps 
PID TTY          TIME CMD
10198 pts/0    00:00:00 sudo
10199 pts/0    00:00:00 unshare
10200 pts/0    00:00:00 bash
10204 pts/0    00:00:00 ps
```
发现进程`/bin/bash`的PID不为`1`。这是因为，`ps`命令会从`/proc`目录下读取进程信息，所以在新的进程下需要启动挂载隔离并重新挂载`/proc`。
```shell
sudo unshare --pid --mount-proc --fork /bin/bash
ps
PID TTY          TIME CMD
  1 pts/0    00:00:00 bash
  5 pts/0    00:00:00 ps
```
可以，看到`/bin/bash`的进程PID为1。

# UTS Namespace

**UTS Namespace**是主机名隔离，使得每个**UTS Namespace**下可以拥有独立的主机名。

```shell
hostname
cheney-hp
```

使用`unshare`命令为新的进程开启**UTS Namespace**，

```shell
sudo unshare --uts --fork /bin/bash
```

新启动的进程内查看主机名并且修改，

```shell
[cheney-hp uts]# hostname
cheney-hp
[cheney-hp uts]# hostname yan
[cheney-hp uts]# hostname
yan
```

可以看出，新启动的进程内，主机名已经更改为`yan`，此时原命名空间内进程主机名仍为`cheney-hp`，

```shell
hostname
cheney-hp
```

# IPC Namespace

**IPC Namespace**用于进程通讯隔离。

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

创建消息队列，

```shell
ipcmk -Q
Message queue id: 2
```

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
0x13e408aa 2          cheney     644        0            0
```

使用`unshare`为新启动的进程开启进程通信隔离，

```shell
sudo unshare --ipc --fork /bin/bash
```

在新进程中查看队列，

```shell
ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages
```

未发现编号为2的消息队列。

# User Namespace

**User Namespace**可以用来隔离用户和用户组，

分别以用户`cheney`和`root`身份创建文件，

```shell
39718892    0 -rw-r--r-- 1 cheney cheney    0 11月 22 11:11 cheney-hello
39718895    0 -rw-r--r-- 1 root   root      0 11月 22 11:11 root-hello
```

通过`unshare`命令，将当前用户映射为**User Namespace**下的`root`用户，

```shell
id
uid=1000(cheney) gid=1000(cheney) groups=1000(cheney),3(sys),90(network),98(power),985(video),991(lp),998(wheel),1001(docker)
```

```shell
unshare --user -r --fork /bin/bash
```

在新启动的`bash`进程内查看用户和目录，

```shell
id
uid=0(root) gid=0(root) groups=0(root),65534(nobody)
```

```shell
ls -lia
total 8
39718891 drwxr-xr-x 2 root   root   4096 11月 22 11:11 .
39584209 drwxr-xr-x 7 root   root   4096 11月 22 10:56 ..
39718892 -rw-r--r-- 1 root   root      0 11月 22 11:11 cheney-hello
39718895 -rw-r--r-- 1 nobody nobody    0 11月 22 11:11 root-hello
```

可以看到，文件`cheney-hello`的用户和用户组由`cheney:cheney`变为了`root:root`，而文件`root-hello`的用户信息由`root:root`转换为`nobody:nobody`。

在新启动进程中，分别向`cheney-hello`和`root-hello`写数据，

```shell
echo 'OK' >> ./cheney-hello
cat ./cheney-hello
OK
```

```shell
echo 'OK' >> ./root-hello
bash: ./root-hello: Permission denied
```

说明，新建的**User Namespace**下的用户`root`等同于宿主机上用户`cheney`，二者权限一致；并且该`root`用户并非宿主机的`root`用户。

执行`su`来切换用户失败，说明用户是隔离的，并且可以映射用户。

```shell
[cheney-hp user]# su root
su: Authentication service cannot retrieve authentication info
[cheney-hp user]# su cheney
su: Authentication service cannot retrieve authentication info
```

# Net Namespace

**Net Namespace**用来隔离网络设备、IP地址和端口信息等。

宿主机上查看网络信息，

```shell
ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eno1: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc fq_codel state DOWN group default qlen 1000
    link/ether c4:34:6b:02:34:dd brd ff:ff:ff:ff:ff:ff
    altname enp14s0
```

使用`unshare`为新的进程建立网络命名空间，

```shell
sudo unshare --net --fork /bin/bash
ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

进存在`loop`环路设备。

## 为进程定义网络命名空间

创建命名空间`ns0-1`

```shell
sudo ip netns add ns0-1
```

创建veth-pair，并把`veth1`分配给命名空间`ns0-1`，

```shell
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns ns0-1
```

为`veth0`分配IP地址，

```shell
sudo ip addr add 192.168.5.1/24 dev veth0
sudo ip link set veth0 up
```

在命名空间`ns0-1`内，为`veth1`分配IP地址，

```shell
sudo ip netns exec ns0-1 ip addr add 192.168.5.2/24 dev veth1
sudo ip netns exec ns0-1 ip link set veth1 up
```

测试网络。

```shell
ping 192.168.5.2 -Iveth0
PING 192.168.5.2 (192.168.5.2) from 192.168.5.1 veth0: 56(84) bytes of data.
64 bytes from 192.168.5.2: icmp_seq=1 ttl=64 time=0.048 ms
64 bytes from 192.168.5.2: icmp_seq=2 ttl=64 time=0.057 ms
```

使用`nsenter`命令复用`ns0-1`，

```shell
nsenter --net=/var/run/netns/ns0-1 /bin/bash
```

在新启动的进程中，查看网络设备，

```shell
ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
8: veth1@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether a6:27:92:0d:4c:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

```shell
ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
8: veth1@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether a6:27:92:0d:4c:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.5.2/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a427:92ff:fe0d:4cf5/64 scope link
       valid_lft forever preferred_lft forever
```

```shell
ping 192.168.5.1
PING 192.168.5.1 (192.168.5.1) 56(84) bytes of data.
64 bytes from 192.168.5.1: icmp_seq=1 ttl=64 time=0.069 ms
64 bytes from 192.168.5.1: icmp_seq=2 ttl=64 time=0.050 ms
64 bytes from 192.168.5.1: icmp_seq=3 ttl=64 time=0.058 ms
```

## 配置进程的网络命名空间

假设`/bin/bash`启动的网络命名空间隔离，

```shell
sudo unshare --net --fork /bin/bash
echo $$
2296
```

`ip netns`操作的目录为`/var/run/netns/`，将`/proc/2296/ns/net`链接到`/var/run/netns/`目录下便可以为其配置网络空间。

```shell
sudo ln -s /proc/2296/ns/net /var/run/netns/ns0-1
```

查看网络命名空间，

```shell
ip netns list
ns0-1
```

使用`ip`命令为PID为2296的进程创建网络设备，

```shell
sudo ip link add veth0 type veth peer name veth1
sudo ip link set veth1 netns ns0-1
sudo ip netns exec ns0-1 ip addr add 192.168.5.2/24 dev veth1
sudo ip netns exec ns0-1 ip link set veth1 up
sudo ip addr add 192.168.5.1/24 dev veth0
sudo ip link set veth0 up
```

PID为2296的进程中查看网络设备，可以发现`veth1`，

```shell
ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether a6:27:92:0d:4c:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

```shell
ip addr show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether a6:27:92:0d:4c:f5 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 192.168.5.2/24 scope global veth1
       valid_lft forever preferred_lft forever
    inet6 fe80::a427:92ff:fe0d:4cf5/64 scope link
       valid_lft forever preferred_lft forever
```

```shell
ping 192.168.5.1 -Iveth1 -c3
PING 192.168.5.1 (192.168.5.1) from 192.168.5.2 veth1: 56(84) bytes of data.
64 bytes from 192.168.5.1: icmp_seq=1 ttl=64 time=0.063 ms
64 bytes from 192.168.5.1: icmp_seq=2 ttl=64 time=0.057 ms
64 bytes from 192.168.5.1: icmp_seq=3 ttl=64 time=0.069 ms

--- 192.168.5.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2017ms
rtt min/avg/max/mdev = 0.057/0.063/0.069/0.005 ms
```

# unshare 和 nsenter

## unshare

> Run a program with some namespaces unshared from the parent.

即`unshare`可以指定创建新的命名空间，而不从父进程继承。

```shell
unshare --help

Usage:
 unshare [options] [<program> [<argument>...]]

Run a program with some namespaces unshared from the parent.

Options:
 -m, --mount[=<file>]      unshare mounts namespace
 -u, --uts[=<file>]        unshare UTS namespace (hostname etc)
 -i, --ipc[=<file>]        unshare System V IPC namespace
 -n, --net[=<file>]        unshare network namespace
 -p, --pid[=<file>]        unshare pid namespace
 -U, --user[=<file>]       unshare user namespace
 -C, --cgroup[=<file>]     unshare cgroup namespace
 -T, --time[=<file>]       unshare time namespace

 -f, --fork                fork before launching <program>
 --map-user=<uid>|<name>   map current user to uid (implies --user)
 --map-group=<gid>|<name>  map current group to gid (implies --user)
 -r, --map-root-user       map current user to root (implies --user)
 -c, --map-current-user    map current user to itself (implies --user)

 --kill-child[=<signame>]  when dying, kill the forked child (implies --fork)
                             defaults to SIGKILL
 --mount-proc[=<dir>]      mount proc filesystem first (implies --mount)
 --propagation slave|shared|private|unchanged
                           modify mount propagation in mount namespace
 --setgroups allow|deny    control the setgroups syscall in user namespaces
 --keep-caps               retain capabilities granted in user namespaces

 -R, --root=<dir>          run the command with root directory set to <dir>
 -w, --wd=<dir>            change working directory to <dir>
 -S, --setuid <uid>        set uid in entered namespace
 -G, --setgid <gid>        set gid in entered namespace
 --monotonic <offset>      set clock monotonic offset (seconds) in time namespaces
 --boottime <offset>       set clock boottime offset (seconds) in time namespaces

 -h, --help                display this help
 -V, --version             display version

For more details see unshare(1).
```

例如，`unshare`选择了选项`--net`，这意味着新的进程拥有新的网络命令空间，

```shell
sudo unshare --net --fork /bin/bash
ip addr
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

可以看到除了环路设备外没有其它网络设备。

`--net[=<file>]`是指创建的新的网络命名空间会另存在指定文件中，在进程关闭后命名空间不消失。

```shell
touch ./net-ns
sudo unshare --net=./net-ns --fork /bin/bash
ip link add veth0 type veth peer name veth1
ip link show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:a6:8e:fa:09:bd brd ff:ff:ff:ff:ff:ff
3: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:b9:c4:c1:bd:2c brd ff:ff:ff:ff:ff:ff
```

本质上是将`/proc/$pid/ns/net`挂载在了`net-ns`文件上（**mount bind**）。可以使用`nsenter`来复用命名空间。

## nsenter

> Run a program with namespaces of other processes.

`nsenter`可以为新启动的进程复用其它进程的命名空间。

```shell
nsenter --help

Usage:
 nsenter [options] [<program> [<argument>...]]

Run a program with namespaces of other processes.

Options:
 -a, --all              enter all namespaces
 -t, --target <pid>     target process to get namespaces from
 -m, --mount[=<file>]   enter mount namespace
 -u, --uts[=<file>]     enter UTS namespace (hostname etc)
 -i, --ipc[=<file>]     enter System V IPC namespace
 -n, --net[=<file>]     enter network namespace
 -p, --pid[=<file>]     enter pid namespace
 -C, --cgroup[=<file>]  enter cgroup namespace
 -U, --user[=<file>]    enter user namespace
 -T, --time[=<file>]    enter time namespace
 -S, --setuid <uid>     set uid in entered namespace
 -G, --setgid <gid>     set gid in entered namespace
     --preserve-credentials do not touch uids or gids
 -r, --root[=<dir>]     set the root directory
 -w, --wd[=<dir>]       set the working directory
 -F, --no-fork          do not fork before exec'ing <program>

 -h, --help             display this help
 -V, --version          display version

For more details see nsenter(1).
```

最常用的方法是`-t`来设置目标进程，配置`-m`、`-u`、`-i`、`-n`等决定复用的命名空间。

`--net[=<file>]`这种形式可以直接指定命名空间路径，通常为`/proc/$pid/ns/`目录下的文件，当然它们的链接文件和挂载点同样可以。

例如，复用`unshare`的另存的命名空间（挂载文件）

```shell
sudo nsenter --net=./net-ns /bin/bash
ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:a6:8e:fa:09:bd brd ff:ff:ff:ff:ff:ff
3: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:b9:c4:c1:bd:2c brd ff:ff:ff:ff:ff:ff
```

复用链接文件，

```shell
ln -s /proc/4779/ns/net ./net-ns-link
sudo nsenter --net=./net-ns-link /bin/bash
ip link
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether 52:a6:8e:fa:09:bd brd ff:ff:ff:ff:ff:ff
3: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ea:b9:c4:c1:bd:2c brd ff:ff:ff:ff:ff:ff
```

