---
title: Yalmip使用学习
date: 2018-04-25 13:58:24
categories: 数学建模
toc: true
tags:
---

## yalmip学习

### 1. yalmip简介

#### 1.1 什么是yalmip

> yalmip是由Lofberg开发的一种免费的优化求解工具，其最大特色在于集成许多外部的最优化求解器，形成一种统一的建模求解语言，提供了Matlab的调用API，减少学习者学习成本。
<!-- more -->
#### 1.2 yalmip安装方式

这里以**MATLAB**的安装方式为例，在[官网](https://yalmip.github.io/)上下载最新包，将其解压至`matlab`的`toolbox`文件夹下（当然也放置在其他文件夹），打开`matlab`软件添加Path路径即可。最后键入`which sdpvar`命令，显示`sdpvar`路径则安装成功。

### 2.yalmip求解优化问题的四部曲

#### 2.1 创建决策变量

`yalmip`一共有三种方式创建决策变量，分别为:

1. `sdpvar`-创建实数型决策变量
2. `intbar`-创建整数型决策变量
3. `binvar`-创建`0/1`型决策变量

不过值得注意的是，在创建`n*n`的决策变量时，`yalmip`默认是对称方阵，所以要创建非对称方针时，需要这样写：

```C
xxxvar(n,n,'full')
```

#### 2.2 添加约束条件

比起`matlab`自带的各种优化函数所要写明的约束条件，`yalmip`的约束条件写起来是非常舒适直观的。

比如要写入`0<=x1+x2+x3<=1`。

那么可以这样写:

```C
% 创建决策变量
x = sdpvar(1,3);
% 添加约束条件
C = [0<=x(1)+x(2)+x(3)<=1];
```

是不是非常爽呢。这才是人类语言（和我初见python的感觉差不多）

#### 2.3 参数配置

关于参数设置，我们大多数是用来设置求解器`solver`的，当然还有其它的选项，可以通过`doc sdpsettings`查看。

#### 2.4 求解问题

最后就是求解问题了。

首先要明确求解目标`z`，`yalmip`默认是求解最小值问题，所以遇到求解最大值的问题，只需要在原问题的基础上添加一个负号即可。

求解调用格式：

```C
optimize(target,constraints,opstions)
```

#### 2.5 几个常用的其它指令

1. `check`：可以检查约束条件是否被满足（检查约束条件的余值）
2. `value`：可以查看变量或表达式的值
3. `assign`: 可以给变量赋值，这个命令调试时很重要

### 3.举两个栗子

#### 3.1 简单例子

如题


$$
\\begin{equation}
z = max(\\frac{x_1+2x_2}{2x_1+x_2}) \\\\
\\left\\{
             \\begin{array}{c}
           	x_1 + x_2 \ge 2	\\\\
           	x_2 - x_1 \le 1	\\\\
           	x_1 \le 1
             \\end{array}
\\right.
\\end{equation}
$$


代码如下，附有详细解释，就不说明了：

```matlab
% 清除工作区
clear;clc;close all;
% 创建决策变量
x = sdpvar(1,2);
% 添加约束条件
C = [
    x(1) + x(2)  >= 2
    x(2)-x(1) <=1
    x(1)<=1
    ];
% 配置 找不到lpsolve求解器请检查是否安装，或者直接不适用'solver'字段。
% ops = sdpsettings('verbose',1);
ops = sdpsettings('verbose',1,'solver','lpsolve');
% 目标函数
z = -(x(1)+2*x(2))/(2*x(1)+x(2)); % 注意这是求解最大值
% 求解
result = optimize(C,z,ops);
if result.problem == 0 % problem =0 代表求解成功
    value(x)
    -value(z)   % 反转
else
    disp('求解出错');
end
```

求解结果:

```C
ans =

   0.500000999998279   1.500000333331623


ans =

   1.399999360001426

```



#### 3.2 解决经典的TSP问题

关于TSP的理论，这里我就不详细介绍了，百度有很多。在遇到yalmip之前，我学习的求解TSP的第一解法就是利用lingo来求解，后来学习了几种智能算法，如遗传算法，模拟退火，蚁群算法等等都可以解决这个问题。现在，学习了yalmip之后，我们可以完全抛弃lingo那种简陋的ide。废话不多说，先贴上约束条件：

$$
\\min Z = \\sum_{i=1}^{n}\\sum_{j=1}^{n} d_{ij}x_{ij}   \\\\
    s.t. \\left\\{  
    \\begin{array}{c}
        \\sum_{i=1,i\\neq j}^{n}x_{ij} = 1,\\qquad j = 1,\\cdots,n      \\\\
        \\sum_{j=1,j\\neq i}^{n}x_{ij} = 1,\\qquad i = 1,\\cdots,n\\\\
        u_i-u_j + nx_{ij} \\le n-1,\\qquad 1< i\\neq j \\le n   \\\\
        x_{ij} = 0 \\text{或} 1,\\qquad u,j=1,\\cdots,n  \\\\
        u_i\\text{为实数},\\qquad i=1,\\cdots,n
    \\end{array}
    \\right.
$$


再来看看代码：

```matlab
% 利用yamlip求解TSP问题
clear;clc;close all;
d = load('tsp_dist_matrix.txt')';
n = size(d,1);
% 决策变量
x = binvar(n,n,'full');
u = sdpvar(1,n);
% 目标
z = sum(sum(d.*x));
% 约束添加
C = [];
for j = 1:n
    s = sum(x(:,j))-x(j,j);
    C = [C,   s  == 1];
end
for i = 1:n
    s = sum(x(i,:)) - x(i,i);
    C = [C, s  == 1];
end
for i = 2:n
    for j = 2:n
        if i~=j
            C = [C,u(i)-u(j) + n*x(i,j)<=n-1];
        end
    end
end
% 参数设置
ops = sdpsettings('verbose',0);
% 求解
result= optimize(C,z,ops);
if result.problem == 0
    value(x)
    value(z)
else
    disp('求解过程中出错');
end

```

这里用到的```tsp_dist_matrix.txt```如下：

```C
0 7 4 5 8 6 12 13 11 18
7 0  3 10 9 14 5 14 17 17
4 3 0 5 9 10 21 8 27 12
5 10 5 0 14 9 10 9 23 16
8 9 9 14 0 7 8 7 20 19
6 14 10 9 7 0 13 5 25 13
12 5 21 10 8 13 0 23 21 18
13 14 8 9 7 5 23 0 18 12
11 17 27 23 20 25 21 18 0 16
18 17 12 16 19 13 18 12 16 0
```

最后来看看结果吧：

```C
>> value(x)

ans =

   NaN     0     0     0     0     0     0     0     1     0
     0   NaN     1     0     0     0     0     0     0     0
     0     0   NaN     1     0     0     0     0     0     0
     1     0     0   NaN     0     0     0     0     0     0
     0     0     0     0   NaN     0     1     0     0     0
     0     0     0     0     1   NaN     0     0     0     0
     0     1     0     0     0     0   NaN     0     0     0
     0     0     0     0     0     1     0   NaN     0     0
     0     0     0     0     0     0     0     0   NaN     1
     0     0     0     0     0     0     0     1     0   NaN

>> value(z)

ans =

    77

```

最后，发现了`yalmip`的一个`bug`，在书写`yalmip`的约束条件时，如下：
```x(1) + x(2)-2  >=  0```
注意x(2)-2这个-2是紧贴x(2)的。这样求解释正确的，如果换成
```x(1) + x(2) -2 >= 0```
`x(2)-2`这个`-2`不是紧贴`x(2)`的。这样求解则会出现问题。

**建议把所有常数写在等式的右边**
