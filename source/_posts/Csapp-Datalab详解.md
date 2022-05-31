---
title: Csapp-Datalab详解
categories: Csapp
tags: Datalab
abbrlink: 2d758396
date: 2020-06-19 21:58:03
---

准备把csapp详细看一遍，所有lab都做一遍，加深理解。

本篇是datalab的个人解法，**所以很可能不是最优解**。

原课程地址：http://www.cs.cmu.edu/afs/cs/academic/class/15213-f15/www/schedule.html

ok，现在就来一道道题说明。

## 0. 说明

datalab着重于让学生理解 数字（integer，float point)在bit level上的表示与操作。通过限制学生的操作集（如仅能使用 ~, |, +等此操作），让学生在bit level上思考问题。

<!-- more -->

more detail:

**对于Integer类型的题型：**

1. 所能使用的常量范围在 0 - 255之间；
2. 只能使用函数参数和局部变量；
3. 有限操作集， ！，~， & ^ | + <<  >> （每道题有各自详细的操作集解释）
4. 不能使用任何的 if, else, do, while, for, switch等
5. 不能使用marco
6. 不能调用函数
7. 不能使用 && ||  -  ？
8. 不能使用类型转换
9. 不能自定义任何类型（如struct， union， array等），实际上，所有题型都只能使用int

**程序环境说明：**

1. 程序为32位程序，
2. 右移操作默认为算数右移（也就是补充符号位）

**对于float类型的题型：**

可以使用， if, else, loop等。只能使用int, unsigned类型。

不能，定义宏，调用函数，类型转换，自定义类型，使用任何float类型的操作。

## 1. bitXor

题目：

- 目标：实现x ^ y. 
- 限制：只能使用操作符 ~, &
- 最大操作次数： 14
- 难度：1

解法：

这道题，我是从“数字电路”中的真值表去思考的，异或操作的真值表如下:

|  X   |  Y   |  Z   |
| :--: | :--: | :--: |
|  0   |  0   |  0   |
|  0   |  1   |  1   |
|  1   |  0   |  1   |
|  1   |  1   |  0   |



所以可列：
$$
X \overline{Y} + \overline{X}Y = Z
$$
利用两次反，数不变原则，可变形为:
$$
\overline{ (\overline{X\overline{Y}})(\overline{\overline{X}Y}) } = Z
$$
则可得:

````c
int bitXor(int x, int y)
{
     return ~(  (~(x & ~y))) & (~(~x & y)  );
}
````

## 2. tmin

题目：

- 目标：返回最小补码数 ，即0x80000000. 
- 限制：只能使用操作符  ! ~ & ^ | + << >>
- 最大操作次数： 4
- 难度：1

解法：

这个题应该是最简单的了，没什么可说的。

```c
int tmin(void)
{
    return 1 << 31;
}
```

## 3. isTmax

题目：

- 目标：判断x是否是Tmax，如果是，返回1，否则返回0。
- 限制：只能使用操作符  ! ~ & ^ | +
- 最大操作次数： 10
- 难度：2

解法：

既然是要判断x == Tmax，那么就要思考Tmax的特殊性。 Tmax = 0x7f ff ff ff.

观察到一个性质， Tmax+1 = ~Tmax. 即 
$$
0X7fffffff+1 = \sim0X80000000
$$
但是还得想一下，还有其它数，有相同的性质吗？

yes，的确有 ， 0xff ff ff ff也满足这样的性质。

所以，现在问题需要加一个过滤，把0xff ff ff ff过滤掉。如何过滤呢？这就要思考 0x7f ff ff ff和0xff ff ff ff的不同了。

又观察到0xff ff ff ff+1 = 0，而0x7f ff ff ff + 1 = 0x80 00 00 00。  所以，可以利用这个特性，把0xff ff ff ff给过滤掉。 **具体是采用 !! 运算。**

对于!! 来说：

| 运算 | 原   | 运算后 |
| ---- | ---- | ------ |
| !!   | 0    | 0      |
| !!   | 非0  | 1      |

所以有： !!(0x7f ff ff ff + 1) = 1, 而!!(0xff ff ff ff+1) = 0。

于是，代码为:

```c
int isTmax(int x)
{
    return !((x + 1) ^ ~x) & !!(x + 1);
}
```

## 4. allOddBits

题目：

- 目标：如果x的二进位的所有奇数位全位1，则返回1，否则返回0。 *注：二进制最低位是第0位。*
- 例子：allOddBits(0xFFFFFFFD) = 0, allOddBits(0xAAAAAAAA) = 1
- 限制：只能使用操作符! ~ & ^ | + << >>
- 最大操作次数： 12
- 难度：2

解法：

 这个题目难度不大,生成0xaa aa aa aa即可，然后判定是否相同即可（通过异或+! 可以判定）

```c
int allOddBits(int x)
{
    int temp = 0xaa;
    temp = temp << 8 | temp; // 0x aa aa
    temp = temp << 8 | temp; // 0x aa aa aa
    temp = temp << 8 | temp; // 0x aa aa aa aa
    return !((x & temp) ^ temp);
}
```

## 5. negate

题目：

- 目标：返回-x
- 限制：只能使用操作符 ! ~ & ^ | + << >>
- 最大操作次数：5
- 难度：2

解法：

这题也很简单，取反+1即可，算是一个二进制的公式吧。只是要记住  -Tmin = Tmin

```c
int negate(int x)
{
    return (~x) + 1;
}
```

## 6. isAsciiDigit

题目：

- 目标： return 1 if 0x30 <= x <= 0x39 (ASCII codes for characters '0' to '9')
- 例子：
  -  isAsciiDigit(0x35) = 1.
  -  isAsciiDigit(0x3a) = 0.
  - isAsciiDigit(0x05) = 0.
- 限制：只能使用操作符   ! ~ & ^ | + << >>
- 最大操作次数：15
- 难度：3

解法：

这道题的整体思路也比较简单，考虑一个更通用的方法， 如何判定 x<=y?  也就是 x- y <= 0. 即 x-y 的符号位为1即可（这里没有考虑负溢出问题）。

所以看代码：

```c
int isAsciiDigit(int x)
{
    int acsii_zero = 0x30;
    int acsii_nine = 0x39;
    int negative_acsii_zero = ~acsii_zero + 1;
    int negative_acsii_nine = ~acsii_nine + 1;
    
    int result1 = !(((x + negative_acsii_zero) >> 31) & 1); //0x30 <= x
    int result2 = ((x + negative_acsii_nine) >> 31) & 1;    // x < 0x39
    int result3 = !(x + (negative_acsii_nine));             // x = 0x39
    return result1 & (result2 | result3);
}

```

## 7. conditional

题目：

- 目标：实现三目运算符  same as x ? y : z 
- 限制：只能使用操作符   ! ~ & ^ | + <<  >>
- 最大操作次数：16
- 难度：3

解法：

这道题其实颇为巧妙，同样用到了数电中的思想，注意，只是思想，实现起来是不一样的。
$$
RESULT  = XY +  \overline{X}{Z}
$$
X=1，则RESULT=Y， X=0，RESULT = Z。

但是这只是在1bit的情况下，要换算到Integer（32bit）范围内，X要么等于0xff ff ff ff, 要么等于0x0.

**问题转化为:**

**如果x = 0，则不做转化。**

**如果x= 非0， 则x要转换为0xff ff ff ff。**

这里又要用到 **!!**运算了。

**又观察到,**  

**0 -1 = 0xff ff ff ff**

**1 -0 = 0**

所以可写代码：

```c
int conditional(int x, int y, int z)
{
    // x 为0 flag= 0 , x不为0,flag = 1
    int flag = !!x;              // flag =1 or flag = 0
    int negative_one = ~0x1 + 1; // -1
    return (~(flag + negative_one) & y) | ((flag + negative_one) & z);
}
```

## 8. isLessOrEqual

题目：

- 目标：如果 x<=y ，则返回1， 否则返回0.
- 限制：只能使用操作符   ! ~ & ^ | + << >>
- 最大操作次数：24
- 难度：3

解法：

这道题目和前文的 `isAsciiDigit` 很相似，用的方法也是类似的，所以不赘述。

个人只是将运算按照 ”一、二、三、四象限“分成了四种情况考虑：

```c
int isLessOrEqual(int x, int y)
{
    int negative_y = ~y + 1;
    // 取出符号位
    int sx = (x >> 31) & 1;
    int sy = (y >> 31) & 1;
    
    // x>y ==> x-y 的符号位一定是0
    int x_less_equal_than_y = (((x + negative_y) >> 31) & 1) | !(x + negative_y);
    
    // x < 0, y>= 0 sx = 1 sy = 0, x < y
    int result1 = sx & (!sy);
    // x < 0 , y < 0 sx = 1, sy = 1, 且 x <= y
    int result2 = sx & sy & x_less_equal_than_y;
    // x >=0 , y >= 0, sx =0 , sy = 0, 且 x <= y
    int result3 = (!sx) & (!sy) & x_less_equal_than_y;
    // x >=0 , y < 0 , sx = 0, sy = 1, 这里 x > y 需要求反,并且不再其它condition中
    int result4 = (!sx) & sy;
    
    return result1 | result2 | result3 | ((!result4) & result1 & result2 & result3);
}
```

## 9. logicalNeg

题目：

- 目标：实现 ! 操作
- 例子： logicalNeg(3) = 0, logicalNeg(0) = 1
- 限制：只能使用操作符  ~ & ^ | + << >>
- 最大操作次数：12
- 难度：4

解法：

这里我的出发点是 0和其它数值有什么不同？

0的相反数是0. （注意Tmin的相反数也是Tmin）

所以设：

y = -x （前文有说-x的bit运算方式）

如果y的首位是0，则说明，x为0或Tmin。 既然0要映射为1，只要 y>>31 +1即可。

如果y的首位是1，则说明，x为非零非Tmin的数，这些值要映射为0，同样y>>31 +1 即可（注意是算数右移，所以y>>31 = 0xff ff ff ff)

现在，剩下的问题为如果区分0 和 Tmin。这个就很简单了。0的符号位为0，Tmin的符号位为1，再运用一次 x>>31+1即可。

所以，代码为:

```c
int logicalNeg(int x)
{
    int negative_x = ~x + 1;
    int result1 = ((x ^ negative_x) >> 31) + 1; // 相反数判定
    int result2 = (x >> 31) + 1;		// 排除Tmin
    return result1 & result2;
}
```

## 10. howManyBits

题目：

- 目标： 返回表示一个数（补码形式）所需要的最小bit数。
- 例子： 
  - howManyBits(12) = 5
  - howManyBits(298) = 10
  - howManyBits(-5) = 4
  - howManyBits(0)  = 1
  - howManyBits(-1) = 1
  - howManyBits(0x80000000) = 32
- 限制：只能使用操作符 ! ~ & ^ | + << >>
- 最大操作次数： 90
- 难度：4

解法：

这道题难度非常大。我认为是datalab中最难的一道题。如果能使用if，loop就比较简单，但是难就难在不能使用这些运算。

所以先说一下思路：

首先从正数出发，如果从最高位到最低位扫描，当你找到首个1时，此时的1所在bit位+1（加1是因为还要个符号位），即是所需要的符号位数量。当然了0是例外。

问题再稍作转化，**首次扫描到两个相邻的bit位分别是0和1时，就算是找到了所需要的符号位数量。**

再考虑一下负数，思考一下前面的黑体字，”扫描到两个bit位时0和1时，就算找到了“，那把负数的情况是不是就是 **”首次扫描到两个bit位分别为1和0时呢？“** （当然，-1除外）。

结合正负数，问题变为 **从高位向低位扫描，首次扫描到两个相邻bit位值不同时，就算是找到了所需要的符号位数量**，不过0和-1是特殊值。

前面说了这么多，其实并不是最终解法，前面说的目的为，**“对于负数，我们可以用对待正数的相同的算法去处理”**，即把负数翻转为正数即可（注意不是取相反数）。

先看下面代码：

关键代码1:

```c
// 如果 x< 0, 则翻转x
int reverse_x = ~x;
// 使用前面的conditional 来做, 三目运算符
int negative_one = ~0x1 + 1; // -1
int flag = !(x >> 31);
x = (~(flag + negative_one) & x) | ((flag + negative_one) & reverse_x);
```

到这里x一定是正数了。

现在就可以统一处理正负数了。问题是我们仍然无法通过if loop计算需要多少位bit来表示一个数。

为了更好的说明下面的算法，这里先贴一张图，以5bit数字进行说明：

![image-20200619232328439](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200619232328439.png)

现在的关键是如何得到这个 **剩余3bit信息：**

如果可以将当扫描位后的所有位置为1，我们就可以**将问题转化为统计 当前数字的二进制表示 有多少个1了。**（为什么要这么想，因为有**bitcount算法**啊）

![image-20200619232535359](https://ravenxrz-blog.oss-cn-chengdu.aliyuncs.com/img/github_img/image-20200619232535359.png)

关键代码2：

对于32位的数据，可以有：

```c
// 把最高位1后的所有位全部填充为 1
x = x | x >> 1;
x = x | x >> 2;
x = x | x >> 4;
x = x | x >> 8;
x = x | x >> 16;
```

接下来就是bitcount算法：

问题已转化为统计二进制中有多个1了。

先以4bit的数字为例：如何统计 **0 1 0 1**的1的个数？

我们可以通过掩码+移位的方式获取：

0 1 0 1 & 1 = 1

(0 1 0 1 >> 1) & 1 = 0

(0 1 0 1 >> 2) & 1 = 1

(0 1 0 1 >> 3) & 1 = 0

最后 1 + 0 + 1 + 0 = 2。

转化为代码为：

```c
int sum = 0;
int mask = 1;
int x = 5; //(0 1 0 1)
sum += x & mask;
sum += (x >> 1) & mask;
sum += (x >> 2) & mask;
sum += (x >> 3) & mask;
```

ok，上面是4bit的情况，如何计算32位呢。 我们当然可以写32次 += 操作来做。但是有更聪明的做法，运用分治的思想，将一个32bit的数，分成4个8bit的段。对每个段，运用上面的算法运算即可。

具体需要个人分析一下代码了，代码很短，所以也很好想清楚。

具体代码：

```c
    // 计算现在1的个数 分治算法 将整个二进制bit分成4段,每段8bit
    int sum = 0;
    int mask = 1;         // 0001
    mask = mask << 8 | 1; // 0000 0001 后文同理
    mask = mask << 8 | 1;
    mask = mask << 8 | 1;
    mask = mask << 8 | 1;
    
    sum += x & mask;
    sum += (x >> 1) & mask;
    sum += (x >> 2) & mask;
    sum += (x >> 3) & mask;
    sum += (x >> 4) & mask;
    sum += (x >> 5) & mask;
    sum += (x >> 6) & mask;
    sum += (x >> 7) & mask;
	
    sum =  (sum & 0xff) + ((sum >> 8) & 0xff) + ((sum >> 16) & 0xff) + ((sum >> 24) & 0xff);
```

ok,现在的sum就是 32bit的二进制中的1的个数了。

结合上面所有算法，可得最终的代码：

```c
int howManyBits(int x)
{
    // 如果 x< 0, 则翻转x
    int reverse_x = ~x;
    // 使用前面的conditional 来做
    int negative_one = ~0x1 + 1; // -1
    int flag = !(x >> 31);
    x = (~(flag + negative_one) & x) | ((flag + negative_one) & reverse_x);
    // 把最高位1后的所有位全部填充为 1
    x = x | x >> 1;
    x = x | x >> 2;
    x = x | x >> 4;
    x = x | x >> 8;
    x = x | x >> 16;
    // 计算现在1的个数 分治算法 将整个二进制bit分成8段,每段4bit
    int sum = 0;
    int mask = 1;         // 0001
    mask = mask << 8 | 1; // 0000 0001 后文同理
    mask = mask << 8 | 1;
    mask = mask << 8 | 1;
    mask = mask << 8 | 1;
    
    sum += x & mask;
    sum += (x >> 1) & mask;
    sum += (x >> 2) & mask;
    sum += (x >> 3) & mask;
    sum += (x >> 4) & mask;
    sum += (x >> 5) & mask;
    sum += (x >> 6) & mask;
    sum += (x >> 7) & mask;
    
    // 分段计算0的个数
    return (sum & 0xff) + ((sum >> 8) & 0xff) + ((sum >> 16) & 0xff) + ((sum >> 24) & 0xff) + 1; // 符号位
}
```

## 11. float_twice

题目：

- 目标： 给定一个浮点数f，返回2*f的二进制表示（单精度float point）。如果为Nan，直接返回参数。
- 限制：只能使用int或unsigned，其余所有操作都可以使用。
- 最大操作次数： 30
- 难度：4

解法：

其实能用if loop后，问题都变得比较简单。只用按照float的IEEE定义与实现取做就行了。

```c
unsigned float_twice(unsigned uf)
{
    // 获取exp
    unsigned exp = 0;
    exp = (uf & 0x7fffffff) >> 23;
    
    // Nan
    unsigned Nan = 0xff;
    if (exp == Nan) {
        return uf;
    }
    
    // exp == 0
    if (exp == 0) {
        // 除符号位外,left shift即可
        unsigned sign = uf >> 31 & 1;
        uf = uf << 1;
        uf = sign ? (uf | 0x80000000) : (uf & 0x7fffffff);
        return uf;
    }
    
    // here , exp != 0
    // exp + 1即可
    exp += 1;
    if (exp == 0xff) {
        // 变为无穷大
        return (uf & 0x80000000) | 0x7f800000;
    } else {
        return (uf & 0x807fffff) | (exp << 23);
    }
}
```

## 12. float_i2f

题目：

- 目标： 给定一个int类型的x，返回(float)x的二进制表示。
- 限制：只能使用int或unsigned，其余所有操作都可以使用。
- 最大操作次数： 30
- 难度：4

解法：

本题依然不算难，唯一需要注意的是，int转为float是有精度损失的。因为int是32bit，但是float的小数m只有23bit，在转化时，需要做舍入操作，舍入采用的是就近偶数（ nearest even）原则。

**我的解法超过了max ops了，并不是最优解**

代码：

```c
unsigned float_i2f(int x)
{
    // 0是特殊情况,本应该采用denormalized方式表达接近0的数,但是这里只有0这个数需要采用denormalized
    if (x == 0)
        return 0;
    
    // tmin 特殊 因为 -tmin = tmin ,二进制情况
    if (x == 0x80000000)
        return 0xcf000000;
    
    // 记录符号位
    int sign = (x >> 31) & 1;
    if (sign == 1)
        x = -x;
    
    const int bias = 127;
    int exp = 0;
    int m = 0;
    
    // 找到最高有效位
    int highest_one_offset = 30;                 // 略过符号位
    while (((x >> highest_one_offset) & 1) != 1) // 因为x!=0, 所有在遍历过程中一定会遇到1
        highest_one_offset--;
    exp = bias + highest_one_offset;
    
    //  构造截断mask,截出所有小数位
    int trunc_frac_mask = 1;
    for (int i = 0; i < highest_one_offset - 1; i++)
        trunc_frac_mask = (trunc_frac_mask << 1) | 1;
    
    // 最低有效位1的便宜量
    int lowest_one_offset = 0;
    while (((x >> lowest_one_offset) & 1) != 1) // 去掉所有末尾的0
        lowest_one_offset++;
    
    // 如果截断长度大于了23位,考虑舍入问题
    int frac_len = highest_one_offset - lowest_one_offset;
    if (frac_len <= 23) {
        // 不用舍入
        m = (x & trunc_frac_mask) >> lowest_one_offset;
        return sign << 31 | exp << 23 | m << (23 - highest_one_offset + lowest_one_offset);
    } else {
        // 需要舍入,(nearest even)
        int temp_frac = (x & trunc_frac_mask) >> lowest_one_offset;
        // 检验有效位后的第一位
        if ((temp_frac >> (frac_len - 23 - 1) & 1) == 0) {
            // 情况1:如果为0,说明将要舍入的部分 未达到小数范围一半直接舍入即可
            m = temp_frac >> (frac_len - 23);
        } else {
            // 情况2:如果为1, 检验是否后面的舍入位是否为全0
            int offset_r = 0;
            while ((offset_r < (frac_len - 23 - 1)) && (temp_frac >> (offset_r) & 1) == 0)
                offset_r++;
            if (offset_r < frac_len - 23 - 1) {
                // 如果后面的舍入位不全为0,则直接向上舍入
                m = (temp_frac >> (frac_len - 23)) + 1;
            } else {
                //  如果后面的舍入位(包含leading位)刚好为一般,及 ?.1000000这种形式,需要考虑偶数舍入
                if ((temp_frac >> (frac_len - 23) & 1) == 0) {
                    //情况3, 向下舍
                    m = temp_frac >> (frac_len - 23);
                } else {
                    // 情况4,向上舍
                    m = (temp_frac >> (frac_len - 23)) + 1;
                }
            }
        }
        return (sign << 31 | exp << 23) + m;
    }
}
```

## 13. float_f2i

题目：

- 目标： 给定一个float类型的x，返回(int)x的二进制表示。Anything out of range (including NaN and infinity) should return 0x80000000u.
- 限制：只能使用int或unsigned，其余所有操作都可以使用。
- 最大操作次数： 30
- 难度：4

解法：

同样，按照定义来即可。只是要注意tmin是特殊的，需要单独考虑。另外需要考虑如何判定out of range。

```c
int float_f2i(unsigned uf)
{
    //tmin 是特殊
    if (uf == 0xcf000000) {
        return 0x80000000;
    }
    
    // 获取exp
    unsigned exp = 0;
    exp = (uf & 0x7fffffff) >> 23;
    // 两个特殊情况, exp 全0或全1
    // if exp is 0
    if (exp == 0) {
        return 0;
    }
    // if ex = nan or inf
    if (exp == 0xff) {
        return 0x80000000;
    }
    
    // sign位
    int sign = (uf >> 31) & 1;
    // e就是小数位数
    int e = exp - 127; // bias = 127
    // m小数位
    int m;
    // 保留结果
    int result;
    // 指数 < 0
    if (e < 0) {
        return 0;
    } else if(e == 0){
        return sign ? -1 : 1;
    }
    else {
        // 指数 > 0
        // 找到小数位的第一个1
        int frac_leading_one_offset = 22;
        while (frac_leading_one_offset > 0 && ((uf >> frac_leading_one_offset) & 1) == 0)
            frac_leading_one_offset++;
        if (frac_leading_one_offset == 0) {
            // m为全0
            result = 1 << e;
            return sign ? -result : result;
        } else {
           // m 不为全0 需要是否考虑out of range
           if(frac_leading_one_offset + e >= 31) // 条件检测
           {
               // out of range
               return 0x80000000;
           }else{
               // 没有 out of range
               m = uf & 0x7fffff;
               int result = (1 << (e + 1)) + (m >> (23 - e));
               return sign ? -result : result;
           }
        }
    }
}
```

