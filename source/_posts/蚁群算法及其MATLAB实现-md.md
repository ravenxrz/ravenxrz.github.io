---
title: 蚁群算法及其MATLAB实现
date: 2017-08-25 13:59:13
categories: 数学建模
toc: true
tags: 算法
---

## 1. 蚁群算法原理

### 1.1 蚁群算法的基本思想
 蚁群算法的基本原理来源于自然界蚂蚁觅食的最短路径原理，蚂蚁在寻找食物源时，能在其走过的路径上释放一种蚂蚁特有的分泌物--**信息素**，使得一定范围内的其他蚂蚁能够察觉到并由此影响他们以后的行为。**当一些路径上通过的蚂蚁越来越多时，其留下的信息素也越来越多，以致信息素强度增大，所以蚂蚁选择选该路径的概率也越高，从而更增加了该路径的信息素强度，这种选择过程被称为蚂蚁的自催化行为**。由于其原理是一种正反馈机制，因此，也可将蚂蚁王国理解为所谓的增强型学习系统。

<!-- more -->

### 1.2 蚁群算法的数学模型

这个利用TSP问题来说明这个数学模型，对于TSP问题，设蚂蚁群体中蚂蚁的数量为$m$，城市的数量为$n$，城市$i$与城市$j$之间的距离为$d_{ij}$，t时刻城市$i$与城市$j$连接路径上的信息素浓度为$c_{ij}(t)$。初始时刻，蚂蚁被放置在不同的城市里，且各城市键连接路径上的信息素浓度相同。然后蚂蚁将按一定概率选择线路，不放设$p^K_{ij}(t)$为t时刻蚂蚁$k$从城市$i$转移到城市$j$的概率。“蚂蚁TSP”策略收到两方面的左右，首先是访问某城市的期望，另外便是其他蚂蚁释放的信息素浓度。所以定义：

$$
p^k_{ij}(t) =\\frac{[c_{ij}(t)]^a \times [n_{ij}(t)]^b}{sum([c_{ij}(t)]^a \times [n_{ij}(t)^b]} ),j∈allowk \\\\
             0,j∉allowk
$$
- $n_{ij}(t)$为启发函数，表示蚂蚁从城市i转移到城市j的期望；
- $allowk$为蚂蚁$k$待访问城市集合，开始时，$allowk$中有$n-1$个元素，即包括除了蚂蚁$k$出发城市的其他多个城市，随着时间的推移，$allowk$中的元素越来越少，直至为空；
- $a$ 为信息素重要程度因子
- $b$为启发函数因子

 在蚂蚁遍历各城市的过程中，与实际情况相似的是，在蚂蚁释放信息素的同事，各个城市之间连接路径上的信息素的强度也在通过挥发等方式逐渐消失。为了描述这个特征，设ρ表示信息素挥发程度。这样所有蚂蚁完成走完一遍所有城市之后，各个城市键连接路径上的信息素浓度为

$$
 c_{ij}(t+1)  =  （1-\\rho）*c_{ij}(t)+\\Delta c_{ij} \\\\
  \\Delta c_{ij} = \\sum{}{} \\Delta c_{ij}^k
$$

- $\\Delta c_{ij}^k$为第k只蚂蚁在城市i与城市k连接路径上释放信息素而增加的信息素浓度
- $\\Delta c_{ij}$ 为所有蚂蚁在城市i与城市j连接路径上释放信息素而增加的信息素浓度。
- 
   一般情况下

$$
\\Delta cijk = \\frac{Q}{L_k} , 若蚂蚁k从城市i访问了城市j \\\\
            0 ,否则
$$


- $Q $为信息素常数
- $L_k$ 为第k只蚂蚁经过路径总长度

### 1.3 蚁群算法流程
![](https://pic.superbed.cn/item/5cfbacb8451253d178d93f2f.png)

## 2. 蚁群算法的MATLAB实现

### 2.1 算法设步骤

1.数据准备
2.计算城市距离矩阵
3.初始化参数
4.迭代寻找最佳路径
5.结果显示

### 2.2 程序代码
程序中使用到的文件"Chap9_citys_data.xlsx"链接如下:
链接：https://pan.baidu.com/s/1MStyADIrhFtDHoVJUuTjzg 
提取码：t24f 
```
%--------------------------------------------------------------------------
%% 数据准备
% 清空环境变量
clear all
clc

% 程序运行计时开始
t0 = clock;
%导入数据
citys=xlsread('Chap9_citys_data.xlsx', 'B2:C53');
%--------------------------------------------------------------------------
%% 计算城市间相互距离
n = size(citys,1);
D = zeros(n,n);
for i = 1:n
    for j = 1:n
        if i ~= j
            D(i,j) = sqrt(sum((citys(i,:) - citys(j,:)).^2));
        else
            D(i,j) = 1e-4;      %设定的对角矩阵修正值
        end
    end    
end
%--------------------------------------------------------------------------
%% 初始化参数
m = 75;                              % 蚂蚁数量
alpha = 1;                           % 信息素重要程度因子
beta = 5;                            % 启发函数重要程度因子
vol = 0.2;                           % 信息素挥发(volatilization)因子
Q = 10;                               % 常系数
Heu_F = 1./D;                        % 启发函数(heuristic function)
Tau = ones(n,n);                     % 信息素矩阵
Table = zeros(m,n);                  % 路径记录表
iter = 1;                            % 迭代次数初值
iter_max = 100;                      % 最大迭代次数 
Route_best = zeros(iter_max,n);      % 各代最佳路径       
Length_best = zeros(iter_max,1);     % 各代最佳路径的长度  
Length_ave = zeros(iter_max,1);      % 各代路径的平均长度  
Limit_iter = 0;                      % 程序收敛时迭代次数
%-------------------------------------------------------------------------
%% 迭代寻找最佳路径
while iter <= iter_max
    % 随机产生各个蚂蚁的起点城市
      start = zeros(m,1);
      for i = 1:m
          temp = randperm(n);
          start(i) = temp(1);
      end
      Table(:,1) = start; 
      % 构建解空间
      citys_index = 1:n;
      % 逐个蚂蚁路径选择
      for i = 1:m
          % 逐个城市路径选择
         for j = 2:n
             has_visited = Table(i,1:(j - 1));           % 已访问的城市集合(禁忌表)
             allow_index = ~ismember(citys_index,has_visited);    % 参加说明1（程序底部）
             allow = citys_index(allow_index);  % 待访问的城市集合
             P = allow;
             % 计算城市间转移概率
             for k = 1:length(allow)
                 P(k) = Tau(has_visited(end),allow(k))^alpha * Heu_F(has_visited(end),allow(k))^beta;
             end
             P = P/sum(P);
             % 轮盘赌法选择下一个访问城市
            Pc = cumsum(P);     %参加说明2(程序底部)
            target_index = find(Pc >= rand);
            target = allow(target_index(1));
            Table(i,j) = target;
         end
      end
      % 计算各个蚂蚁的路径距离
      Length = zeros(m,1);
      for i = 1:m
          Route = Table(i,:);
          for j = 1:(n - 1)
              Length(i) = Length(i) + D(Route(j),Route(j + 1));
          end
          Length(i) = Length(i) + D(Route(n),Route(1));
      end
      % 计算最短路径距离及平均距离
      if iter == 1
          [min_Length,min_index] = min(Length);
          Length_best(iter) = min_Length;  
          Length_ave(iter) = mean(Length);
          Route_best(iter,:) = Table(min_index,:);
          Limit_iter = 1; 
          
      else
          [min_Length,min_index] = min(Length);
          Length_best(iter) = min(Length_best(iter - 1),min_Length);
          Length_ave(iter) = mean(Length);
          if Length_best(iter) == min_Length
              Route_best(iter,:) = Table(min_index,:);
              Limit_iter = iter; 
          else
              Route_best(iter,:) = Route_best((iter-1),:);
          end
      end
      % 更新信息素
      Delta_Tau = zeros(n,n);
      % 逐个蚂蚁计算
      for i = 1:m
          % 逐个城市计算
          for j = 1:(n - 1)
              Delta_Tau(Table(i,j),Table(i,j+1)) = Delta_Tau(Table(i,j),Table(i,j+1)) + Q/Length(i);
          end
          Delta_Tau(Table(i,n),Table(i,1)) = Delta_Tau(Table(i,n),Table(i,1)) + Q/Length(i);
      end
      Tau = (1-vol) * Tau + Delta_Tau;
    % 迭代次数加1，清空路径记录表
    iter = iter + 1;
    Table = zeros(m,n);
end
%--------------------------------------------------------------------------
%% 结果显示
[Shortest_Length,index] = min(Length_best);
Shortest_Route = Route_best(index,:);
Time_Cost=etime(clock,t0);
disp(['最短距离:' num2str(Shortest_Length)]);
disp(['最短路径:' num2str([Shortest_Route Shortest_Route(1)])]);
disp(['收敛迭代次数:' num2str(Limit_iter)]);
disp(['程序执行时间:' num2str(Time_Cost) '秒']);
%--------------------------------------------------------------------------
%% 绘图
figure(1)
plot([citys(Shortest_Route,1);citys(Shortest_Route(1),1)],...  %三点省略符为Matlab续行符
     [citys(Shortest_Route,2);citys(Shortest_Route(1),2)],'o-');
grid on
for i = 1:size(citys,1)
    text(citys(i,1),citys(i,2),['   ' num2str(i)]);
end
text(citys(Shortest_Route(1),1),citys(Shortest_Route(1),2),'       起点');
text(citys(Shortest_Route(end),1),citys(Shortest_Route(end),2),'       终点');
xlabel('城市位置横坐标')
ylabel('城市位置纵坐标')
title(['ACA最优化路径(最短距离:' num2str(Shortest_Length) ')'])
figure(2)
plot(1:iter_max,Length_best,'b')
legend('最短距离')
xlabel('迭代次数')
ylabel('距离')
title('算法收敛轨迹')
%--------------------------------------------------------------------------
%% 程序解释或说明
% 1. ismember函数判断一个变量中的元素是否在另一个变量中出现，返回0-1矩阵；
% 2. cumsum函数用于求变量中累加元素的和，如A=[1, 2, 3, 4, 5], 那么cumsum(A)=[1, 3, 6, 10, 15]。
```

程序结果：

![](https://pic3.superbed.cn/item/5cfbacba451253d178d93f65.png)

![](https://pic3.superbed.cn/item/5cfbacbb451253d178d93f98.png)

## 3.算法关键参数的设定

#### 1.参数设定的准则

1. 尽可能在全局上搜索最优解，保证解得最有型
2. 算法尽快手链，以节省寻优时间
3. 尽量反映客观存在的规律，以保证这种仿生算法的真实性

#### 2.蚂蚁数量
一般设置蚂蚁数量为城市数的1.5倍比较稳妥

#### 3.信息素因子
 信息素因素a反映蚂蚁在运动过程中所积累的信息量在知道蚁群搜索中的相对重要程度。**当a∈[1,4]时**，综合求解性能较好

#### 4.启发函数因子
启发函数因子b，反映了启发式信息在知道蚁群搜索过程中的相对重要程度，其大小反映了蚁群巡游过程中小言行、确定性因素的作用强度。b过大是，蚂蚁在某个局部点上选择局优的可能性大。**b∈[3,4.5]**，综合求解性能较好。

#### 5.信息素挥发因子
 信息素挥发因子ρ描述信息素的消失水平，而1-ρ则为信息素残留因子。**ρ∈[0.2,0.5]时**，综合求解能力较好

#### 6. 最大迭代次数

一般去100-500

#### 7. 组合参数设计策略

可按照一下策略来进行参数的组合设计：

1. 确定蚂蚁数目，蚂蚁数目与城市规模之比约为1。5
2. 参数粗调，即调整取值范围较大的a,b以及Q
3. 参数微调，即调整取值范围较小的ρ

## 4.总结

蚁群算法有一下特点：

1. 从算法的性质而言，蚁群算法是在寻找一个比较好的局部最优解，而不是强调全局最优解
2. 开始时算法收敛速度较快，在随后寻优过程中，迭代一定次数后，容易出现停滞现象
3. 蚁群算法对TSP及相似问题具有良好的适应性，无论城市规模大还是小，都能进行有效地求解，而且求解速度相对较快
4. 蚁群算法解得稳定性较差，及时参数不变，每次执行程序都有可能得到不同界，为此需要多执行几次，已寻找最佳解。
5. 蚁群算法中有多个需要设定的参数，而且这些参数对程序又都有一定的影响，所以选择合适的参数组合在算法设计过程中也非常重要。
