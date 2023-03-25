---
layout: post
title: "Graph easy工具"
subtitle: "Graph easy是一种用于在Cli下的绘图工具"
date: 2021-06-22
author: "Cheney.Yin"
header-img: "//i.328888.xyz/2023/03/25/iADI63.jpeg"
tags:
 - Cli
 - graphviz
---

# 基本语法
`graph-easy`兼容graphviz语法

```shell
echo '[hello]<->[world]' | graph-easy
+-------+      +-------+
| hello | <--> | world |
+-------+      +-------+
```

```shell
echo '[hello]<->[world]->[OK]\n' | graph-easy
+-------+      +-------+     +----+
| hello | <--> | world | --> | OK |
+-------+      +-------+     +----+
```

```shell
echo '[hello]<->[world]->[OK]\n[good]->[world]' | graph-easy
+------+     +-------+     +----+
| good | --> | world | --> | OK |
+------+     +-------+     +----+
               ^
               |
               |
               v
             +-------+
             | hello |
             +-------+
```

# 嵌入子图

```shell
echo '(ns0-1: [veth1]) [veth0]<->[veth1] [veth0]<->[brt]' | graph-easy
  +-------+        +-----+
  | veth0 |   <--> | brt |
  +-------+        +-----+
    ^
    |
    |
    v
+ - - - - - +
' ns0-1:    '
'           '
' +-------+ '
' | veth1 | '
' +-------+ '
'           '
+ - - - - - +
```

# 连接线加标签

```shell
echo '(ns0-1: [veth1]) [veth0]<->{label:"pair"}[veth1] [veth0]<->[brt]' | graph-easy
  +-------+        +-----+
  | veth0 |   <--> | brt |
  +-------+        +-----+
    ^
    |
    | pair
    v
+ - - - - - +
' ns0-1:    '
'           '
' +-------+ '
' | veth1 | '
' +-------+ '
'           '
+ - - - - - +
```
