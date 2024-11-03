---
layout: post
title: "Linux文本处理"
subtitle: "整理Linux文本处理脚本"
date: 2024-11-03
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags:
 - Linux
 - shell
 - sed
 - awk
 - tr
---

# Linux文本处理

## 多行连接

```shell
> for e in `seq 1 7`; do echo $e; done                                       
1
2
3
4
5
6
7
```

使用`sed`正序连接
```shell
> for e in `seq 1 7`; do echo $e; done | sed -n '1h;1!H;1!g;$!d; s/\n/,/g; p'
1,2,3,4,5,6,7
```
或者
```shell
> for e in `seq 1 7`; do echo $e; done | sed ':a;N;$!ba;s/\n/,/g' 
1,2,3,4,5,6,7
```

使用`sed`逆序连接
```shell
> for e in `seq 1 7`; do echo $e; done | sed '1!G;h;$!d; s/\n/,/g'
7,6,5,4,3,2,1
```

使用`tr`正序连接
```shell
> for e in `seq 1 7`; do echo $e; done | tr -s '\n' ',' | sed 's/,$//'
1,2,3,4,5,6,7
```

使用`tr`逆序连接
```shell
> for e in `seq 1 7`; do echo $e; done | tr -s '\n' ',' | sed 's/,$//' | rev
7,6,5,4,3,2,1
```

使用`awk`正序连接
```shell
> for e in `seq 1 7`; do echo $e; done | awk 'ORS=","' | sed 's/,$//'       
1,2,3,4,5,6,7
```

使用`awk`逆序连接
```shell
> for e in `seq 1 7`; do echo $e; done | awk 'ORS=","' | sed 's/,$//' | rev
7,6,5,4,3,2,1
```
