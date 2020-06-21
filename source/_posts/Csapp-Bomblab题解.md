---
title: Csapp-Bomblab题解
categories: Csapp
tags: Bomblab
abbrlink: 7915569e
date: 2020-06-21 23:06:10
---

今天继续csapp学习.

前一个lab: [Csapp-Datalab 详解 ](https://www.ravenxrz.ink/archives/2d758396.html)

本次lab, bomblab.

## 0. 说明

 这个实验相当好玩, 题如其名, 类似一个拆炸弹的过程. 实验只给了一个可执行文件, 需要学生通过gdb反汇编这个可执行文件, "拆弹"共有6个阶段, 每个阶段需要用户输入一个特定的字符串, 一旦输入错误, 炸弹就会爆炸,程序终止.

做完整个实验, **学生可以学会如何使用gdb, 能够看懂 gcc所编译出来的汇编代码. 掌握阅读汇编代码的能力.**

gdb的使用可参考: [gdb 调试基础 ](https://www.ravenxrz.ink/archives/37784c45.html)

**本次解释, 均已代码注释+图形解释.**

**所有汇编代码, 可通过 objdump -d bomb获得.**

<!-- more -->

## 1. phase 1

下面是phase 1的汇编代码:

```asm
0000000000400ee0 <phase_1>:
   400ee0:   48 83 ec 08             sub    $0x8,%rsp
   400ee4:   be 00 24 40 00          mov    $0x402400,%esi             ; 0x402400是重点
   400ee9:   e8 4a 04 00 00          callq  401338 <strings_not_equal> ; 根据函数可知, 这里要比较两个字符串是否相同
   400eee:   85 c0                   test   %eax,%eax
   400ef0:   74 05                   je     400ef7 <phase_1+0x17>
   400ef2:   e8 43 05 00 00          callq  40143a <explode_bomb>
   400ef7:   48 83 c4 08             add    $0x8,%rsp
   400efb:   c3                      retq   
```

所以,我们需要查看内存地址 0x402400的字符串是什么, 通过 `x /100cb 0x402400`命令,可得:

![image-20200621205547340](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200621205547340.png)

找到 `\0`的位置, 之前所有的字符组成的字符串即为答案:

```
Border relations with Canada have never been better.
```

## 2. phase 2

源代码和注释:

```asm
 0000000000400efc <phase_2>:
   400efc:   55                      push   %rbp
   400efd:   53                      push   %rbx
   400efe:   48 83 ec 28             sub    $0x28,%rsp
   400f02:   48 89 e6                mov    %rsp,%rsi
   400f05:   e8 52 05 00 00          callq  40145c <read_six_numbers>    ;重点在这个函数, 内部采用sscanf实现
   400f0a:   83 3c 24 01             cmpl   $0x1,(%rsp)                  ;输入的第一值必须为1
   400f0e:   74 20                   je     400f30 <phase_2+0x34>
   400f10:   e8 25 05 00 00          callq  40143a <explode_bomb>
   400f15:   eb 19                   jmp    400f30 <phase_2+0x34>
   400f17:   8b 43 fc                mov    -0x4(%rbx),%eax              ; 这里为 x_{i}
   400f1a:   01 c0                   add    %eax,%eax                    ; x_{i}=2*x_{i},将当前值加倍
   400f1c:   39 03                   cmp    %eax,(%rbx)                  ; 这里 为x_{i+1}, 
   400f1e:   74 05                   je     400f25 <phase_2+0x29>        ; 要求 x_{i+1} = x{i}, 结合这三行代码, 要求为,上一个数的2倍=下一个数 
   400f20:   e8 15 05 00 00          callq  40143a <explode_bomb>
   400f25:   48 83 c3 04             add    $0x4,%rbx
   400f29:   48 39 eb                cmp    %rbp,%rbx
   400f2c:   75 e9                   jne    400f17 <phase_2+0x1b>
   400f2e:   eb 0c                   jmp    400f3c <phase_2+0x40>
   400f30:   48 8d 5c 24 04          lea    0x4(%rsp),%rbx
   400f35:   48 8d 6c 24 18          lea    0x18(%rsp),%rbp
   400f3a:   eb db                   jmp    400f17 <phase_2+0x1b>
   400f3c:   48 83 c4 28             add    $0x28,%rsp
   400f40:   5b                      pop    %rbx
   400f41:   5d                      pop    %rbp
   400f42:   c3                      retq                                                
```

这个题的重点有两个:

1. 函数 read_sim_numers. 内部sscanf实现. 从函数名也可一直,本题要求输入值的数量为6.
2. 指令地址 0x400f17开始的地方, 结合这里的4行代码, 可以推断,本题要求输入值的约束为:

$$
x_{i} \times 2 = x_{i+1} \quad i = {0\dots5}
$$

且 0x400f0a要求第一个参数为1.

所以,答案为:

```
1 2 4 8 16 32 64
```

## 3. phase 3

```asm
 0000000000400f43 <phase_3>:
   400f43:   48 83 ec 18             sub    $0x18,%rsp
   400f47:   48 8d 4c 24 0c          lea    0xc(%rsp),%rcx
   400f4c:   48 8d 54 24 08          lea    0x8(%rsp),%rdx
   400f51:   be cf 25 40 00          mov    $0x4025cf,%esi
   400f56:   b8 00 00 00 00          mov    $0x0,%eax
   400f5b:   e8 90 fc ff ff          callq  400bf0 <__isoc99_sscanf@plt>
   400f60:   83 f8 01                cmp    $0x1,%eax            ; 输入参数的数量应该>1
   400f63:   7f 05                   jg     400f6a <phase_3+0x27>
   400f65:   e8 d0 04 00 00          callq  40143a <explode_bomb>
   400f6a:   83 7c 24 08 07          cmpl   $0x7,0x8(%rsp)       ; 0x80(rsp)地址空间的value : 0x7
   400f6f:   77 3c                   ja     400fad <phase_3+0x6a> ; 是否符号小
   400f71:   8b 44 24 08             mov    0x8(%rsp),%eax
   400f75:   ff 24 c5 70 24 40 00    jmpq   *0x402470(,%rax,8)
   400f7c:   b8 cf 00 00 00          mov    $0xcf,%eax
   400f81:   eb 3b                   jmp    400fbe <phase_3+0x7b>
   400f83:   b8 c3 02 00 00          mov    $0x2c3,%eax
   400f88:   eb 34                   jmp    400fbe <phase_3+0x7b>
   400f8a:   b8 00 01 00 00          mov    $0x100,%eax
   400f8f:   eb 2d                   jmp    400fbe <phase_3+0x7b>
   400f91:   b8 85 01 00 00          mov    $0x185,%eax
   400f96:   eb 26                   jmp    400fbe <phase_3+0x7b>
   400f98:   b8 ce 00 00 00          mov    $0xce,%eax
   400f9d:   eb 1f                   jmp    400fbe <phase_3+0x7b>
   400f9f:   b8 aa 02 00 00          mov    $0x2aa,%eax
   400fa4:   eb 18                   jmp    400fbe <phase_3+0x7b>
   400fa6:   b8 47 01 00 00          mov    $0x147,%eax
   400fab:   eb 11                   jmp    400fbe <phase_3+0x7b>
   400fad:   e8 88 04 00 00          callq  40143a <explode_bomb>
   400fb2:   b8 00 00 00 00          mov    $0x0,%eax
   400fb7:   eb 05                   jmp    400fbe <phase_3+0x7b>
   400fb9:   b8 37 01 00 00          mov    $0x137,%eax          
   400fbe:   3b 44 24 0c             cmp    0xc(%rsp),%eax       ; 0x137 : 0x($rsp) , 通过x打印命令,可知0x($rsp)就是要求输入的第二参数
   400fc2:   74 05                   je     400fc9 <phase_3+0x86> ; 所以第二参数为0x137
   400fc4:   e8 71 04 00 00          callq  40143a <explode_bomb>
   400fc9:   48 83 c4 18             add    $0x18,%rsp
   400fcd:   c3                      retq 
```

这里注意三个点:

1. 输入的字段数>1
2. 第二个参数为0x137, 在指令地址为0x400fbe地方可见.

答案:

```
1 311
```

## 4. phase 4

```asm
0000000000400fce <func4>:
   400fce:   48 83 ec 08             sub    $0x8,%rsp
   400fd2:   89 d0                   mov    %edx,%eax
   400fd4:   29 f0                   sub    %esi,%eax            ; 第一次, eax=17; 第二次进入, 
   400fd6:   89 c1                   mov    %eax,%ecx            ; 第一次进入时, ecx = 14
   400fd8:   c1 e9 1f                shr    $0x1f,%ecx           ; 第一次进入时, ecx = 0
   400fdb:   01 c8                   add    %ecx,%ea
   400fdd:   d1 f8                   sar    %eax                 ; 第一次进入时, ecx = 7
   400fdf:   8d 0c 30                lea    (%rax,%rsi,1),%ecx   ; 第一次进入时, ecx = 7
   400fe2:   39 f9                   cmp    %edi,%ecx            ; ecx : edi
   400fe4:   7e 0c                   jle    400ff2 <func4+0x24>  ; ecx <= edi
   400fe6:   8d 51 ff                lea    -0x1(%rcx),%edx      ; 第一次进入,ecdx = 6
   400fe9:   e8 e0 ff ff ff          callq  400fce <func4>       ; 递归调用
   400fee:   01 c0                   add    %eax,%eax            
   400ff0:   eb 15                   jmp    401007 <func4+0x39>
   400ff2:   b8 00 00 00 00          mov    $0x0,%eax           ; eax  = 0 
   400ff7:   39 f9                   cmp    %edi,%ecx           ; ecx : edi
   400ff9:   7d 0c                   jge    401007 <func4+0x39> ; ecx >= edi 结合上面 ecx <=edi ==> ecx = edi, 最后一层应该在这里返回, 而前文已经推断得到ecx=7
   400ffb:   8d 71 01                lea    0x1(%rcx),%esi
   400ffe:   e8 cb ff ff ff          callq  400fce <func4>
   401003:   8d 44 00 01             lea    0x1(%rax,%rax,1),%eax    ; 最后一层返回eax时,eax 应该=0
   401007:   48 83 c4 08             add    $0x8,%rsp
   40100b:   c3                      retq
   
 000000000040100c <phase_4>:
   40100c:   48 83 ec 18             sub    $0x18,%rsp
   401010:   48 8d 4c 24 0c          lea    0xc(%rsp),%rcx
   401015:   48 8d 54 24 08          lea    0x8(%rsp),%rdx
   40101a:   be cf 25 40 00          mov    $0x4025cf,%esi
   40101f:   b8 00 00 00 00          mov    $0x0,%eax
   401024:   e8 c7 fb ff ff          callq  400bf0 <__isoc99_sscanf@plt>
   401029:   83 f8 02                cmp    $0x2,%eax            ; 需要输入2个参数
   40102c:   75 07                   jne    401035 <phase_4+0x29>
   40102e:   83 7c 24 08 0e          cmpl   $0xe,0x8(%rsp)
   401033:   76 05                   jbe    40103a <phase_4+0x2e>
   401035:   e8 00 04 00 00          callq  40143a <explode_bomb>
   40103a:   ba 0e 00 00 00          mov    $0xe,%edx            ; 构造函数参数3
   40103f:   be 00 00 00 00          mov    $0x0,%esi            ; 构造函数参数2
   401044:   8b 7c 24 08             mov    0x8(%rsp),%edi       ; 构造函数参数1 edi=输入的第一个参数
   401048:   e8 81 ff ff ff          callq  400fce <func4>       ; 进入func4函数
   40104d:   85 c0                   test   %eax,%eax            ; 这行和下一行,要求func4返回的参数一定等于0
   40104f:   75 07                   jne    401058 <phase_4+0x4c>
   401051:   83 7c 24 0c 00          cmpl   $0x0,0xc(%rsp)       ; 第二参数为0
   401056:   74 05                   je     40105d <phase_4+0x51>
   401058:   e8 dd 03 00 00          callq  40143a <explode_bomb>
   40105d:   48 83 c4 18             add    $0x18,%rsp
   401061:   c3                      retq
```

这是一个递归题目, 难点在参数1, 参数2的确定非常简单,直接为0.

而参数1,则需要带入到代码中,还原递归的过程.

还原为递归代码:

```c
void func4(int x, int y, int z) {
    int t = z - y;
    int k = t  31;
    t = (t + k)  1;
    k = t + y;
    if(k <= x) {
        t = 0;
        if(k >= x) {
            return;
        }else {
            y = k + 1;
            func4(x, y, z);
        }
    }else {
        z = k - 1;
        func4(x, y, z);
    }
}
```

最终可注意到, x = 7 3 1都可以.

答案:

```
7 0 或 3 0 或 1 0
```

## 5. phase 5

```asm
 0000000000401062 <phase_5>:
   401062:   53                      push   %rbx
   401063:   48 83 ec 20             sub    $0x20,%rsp
   401067:   48 89 fb                mov    %rdi,%rbx
   40106a:   64 48 8b 04 25 28 00    mov    %fs:0x28,%rax
   401071:   00 00 
   401073:   48 89 44 24 18          mov    %rax,0x18(%rsp
   401078:   31 c0                   xor    %eax,%eax
   40107a:   e8 9c 02 00 00          callq  40131b <string_length>
   40107f:   83 f8 06                cmp    $0x6,%eax            ; 输入6个字段
   401082:   74 4e                   je     4010d2 <phase_5+0x70>
   401084:   e8 b1 03 00 00          callq  40143a <explode_bomb>
   401089:   eb 47                   jmp    4010d2 <phase_5+0x70>
   -----------------------使用坐标 start-----------------------------------
   40108b:   0f b6 0c 03             movzbl (%rbx,%rax,1),%ecx   ; rbx存放的的是输入字符串的首地址, rax是索引下表,所以这里时将输入的每个字符传递给ecx
   40108f:   88 0c 24                mov    %cl,(%rsp)           ;
   401092:   48 8b 14 24             mov    (%rsp),%rdx          ; 取ecx的低16位给rdx
   401096:   83 e2 0f                and    $0xf,%edx            ; 取ecx的低4位. 
   401099:   0f b6 92 b0 24 40 00    movzbl 0x4024b0(%rdx),%edx  ;关键代码:以0x4024b0地址为基, 加上rdx偏移量,替换原有字符串. 
   4010a0:   88 54 04 10             mov    %dl,0x10(%rsp,%rax,1)
   4010a4:   48 83 c0 01             add    $0x1,%rax
   4010a8:   48 83 f8 06             cmp    $0x6,%rax
   4010ac:   75 dd                   jne    40108b <phase_5+0x29>
   ----------------------使用坐标 end--------------------------------------
   4010ae:   c6 44 24 16 00          movb   $0x0,0x16(%rsp)
   4010b3:   be 5e 24 40 00          mov    $0x40245e,%esi       ; 关键代码, 查看内存地址空间0x40245e的字符串为 flyers, 比较前文获取的字符串是否等于flyers
   4010b8:   48 8d 7c 24 10          lea    0x10(%rsp),%rdi      ; 
   4010bd:   e8 76 02 00 00          callq  401338 <strings_not_equal> ;
   4010c2:   85 c0                   test   %eax,%eax
   4010c4:   74 13                   je     4010d9 <phase_5+0x77>
   4010c6:   e8 6f 03 00 00          callq  40143a <explode_bomb>
   4010cb:   0f 1f 44 00 00          nopl   0x0(%rax,%rax,1)
   4010d0:   eb 07                   jmp    4010d9 <phase_5+0x77>
   4010d2:   b8 00 00 00 00          mov    $0x0,%eax
   4010d7:   eb b2                   jmp    40108b <phase_5+0x29>
   4010d9:   48 8b 44 24 18          mov    0x18(%rsp),%rax
   4010de:   64 48 33 04 25 28 00    xor    %fs:0x28,%rax
   4010e5:   00 00 
   4010e7:   74 05                   je     4010ee <phase_5+0x8c>
   4010e9:   e8 42 fa ff ff          callq  400b30 <__stack_chk_fail@plt>
   4010ee:   48 83 c4 20             add    $0x20,%rsp
   4010f2:   5b                      pop    %rbx
   4010f3:   c3                      retq 
```

这个题相当有意思,给我一种破译密码的感觉.

可以理解为, 当前手上有一份加密码表, 我们要找到一个合适的坐标, 通过这个坐标去, 然后去密码表上查找对应码字,使得这些码字组成等于"flyers".

解释:

- 坐标: 就是我们要输入的字符串的每个字符. 然后取这些字符的低4位作为坐标
- 密码表: 0x4024b地址空间.得到坐标后, 查看以0x4024b为首地址的字符串加上这些坐标,得到解密后的码字.
- 目标码: 0x40245e地址空间. 我们解密出来的码字要等于这个地址空间所拥有的字符串, 经查看,为flyers

另外,值得注意的是, 以字符的低4bit作为坐标, 这样会有多个字符有相同的低4bit. 查看acsii码表.

![image-20200621224830418](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200621224830418.png)

如果以数字表示坐标,则坐标为:

```
9 15 6 7
```

对比acsii码:

第一个坐标9, 对应到acsii表的第9行:

```
) 9 I Y i y
```

也就是说, 第一个坐标,可以是这6个字符中的任意一个.

同理,第二个坐标15, 也就是F行:

```
/ ? 0 _ o DEL
```

最后, 个人答案:

```c
9/.567
```

## 6. phase 6

phase 6是bomblab中最难的题, 我对代码做了详细的解释:

```asm
00000000004010f4 <phase_6>:
  4010f4:       push   %r14
  4010f6:       push   %r13
  4010f8:       push   %r12
  4010fa:       push   %rbp
  4010fb:       push   %rbx
  4010fc:       sub    $0x50,%rsp
  401100:       mov    %rsp,%r13
  401103:       mov    %rsp,%rsi
  -----------------1.输入字段数是否为6检验 start--------------------------
  401106:       callq  40145c <read_six_numbers> ; 读取6个字段
  40110b:       mov    %rsp,%r14
  40110e:       mov    $0x0,%r12d
  -----------------2.输入的6个字段不能重复检验 start---------------------
  401114:       mov    %r13,%rbp
  401117:       mov    0x0(%r13),%eax
  40111b:       sub    $0x1,%eax                ; eax = eax -1
  40111e:       cmp    $0x5,%eax                ; eax : 5
  401121:       jbe    401128 <phase_6+0x34>    ; 相等,则通过, 这里是检验是否输入的是6个字段
  401123:       callq  40143a <explode_bomb>
  -----------------1.输入字段数是否为6检验 end--------------------------
  401128:       add    $0x1,%r12d
  40112c:       cmp    $0x6,%r12d
  401130:       je     401153 <phase_6+0x5f>
  401132:       mov    %r12d,%ebx
  401135:       movslq %ebx,%rax               ; rax作为输入数组的索引下标
  401138:       mov    (%rsp,%rax,4),%eax      ; 取出第rax个数,并存放在eax中
  40113b:       cmp    %eax,0x0(%rbp)          ; rbp其实就是外部循环的被用来比较的数(设为x_i),eax是内部循环找到的数(设为x_j)  j = i+1 到 5 ,i=0 到5
  40113e:       jne    401145 <phase_6+0x51>
  401140:       callq  40143a <explode_bomb>
  401145:       add    $0x1,%ebx               ; 内部循环+1
  401148:       cmp    $0x5,%ebx
  40114b:       jle    401135 <phase_6+0x41>
  40114d:       add    $0x4,%r13               ; 外部循环+1
  401151:       jmp    401114 <phase_6+0x20>
  -----------------2.输入的6个字段不能重复检验 end---------------------
  -----------------3. 所有数组元素x_{i} = 7 - x{i} start--------------
  401153:       lea    0x18(%rsp),%rsi         ; 指针走到数组头
  401158:       mov    %r14,%rax
  40115b:       mov    $0x7,%ecx               ; 7
  401160:       mov    %ecx,%edx
  401162:       sub    (%rax),%edx             ; x_i = 7 - x_i
  401164:       mov    %edx,(%rax)
  401166:       add    $0x4,%rax               ; 指针+1
  40116a:       cmp    %rsi,%rax               ; 是否走到数组尾
  40116d:       jne    401160 <phase_6+0x6c>
  40116f:       mov    $0x0,%esi
  401174:       jmp    401197 <phase_6+0xa3>   ; 这里直接产生了代码的跨越, 所以先调到 0x401197地址去看看
  -----------------3. 所有数组元素x_{i} = 7 - x{i} end--------------
  -----------------4. 链表节点处理代码段(将链表节点地址移动到stack的内存中) start-----
  401176:       mov    0x8(%rdx),%rdx
  40117a:       add    $0x1,%eax
  40117d:       cmp    %ecx,%eax
  40117f:       jne    401176 <phase_6+0x82>   ; 找到链表中相应的元素.链表node结构中有一个类似id的字段, 目前的数组元素就是用来找到对应的id, id的范围是1-6
  401181:       jmp    401188 <phase_6+0x94>
  401183:       mov    $0x6032d0,%edx          ; 本题的关键, 查看0x6032d0内存地址空间, 你会发现这里有一个nodex的段, 可以猜想是某种数据结构, 链表/tree/图等, 通过更多的x打印,会发现大概率是链表结构
  401188:       mov    %rdx,0x20(%rsp,%rsi,2)   ; 将链表的node地址,移动到stack空间去
  40118d:       add    $0x4,%rsi                ; rsi指针+1
  401191:       cmp    $0x18,%rsi               ; 是否走到数组尾
  401195:       je     4011ab <phase_6+0xb7>
  401197:       mov    (%rsp,%rsi,1),%ecx
  40119a:       cmp    $0x1,%ecx               ; 映射后的数组元素是否<=1
  40119d:       jle    401183 <phase_6+0x8f>
  40119f:       mov    $0x1,%eax               ;
  4011a4:       mov    $0x6032d0,%edx          ; 链表头节点地址
  4011a9:       jmp    401176 <phase_6+0x82>   ; 跳转到链表节点处理代码段
  -----------------4. 链表节点处理代码段(将链表节点地址移动到stack的内存中) end-----
  -----------------5. 更新链表next指针(按照映射后的数组元素连接) start-------------
  4011ab:       mov    0x20(%rsp),%rbx         ; 走到这里,说明0x6032d0内存地址中的所有链表node地址,都已经移动到了$rsp+0x20上的内存空间
  4011b0:       lea    0x28(%rsp),%rax
  4011b5:       lea    0x50(%rsp),%rsi
  4011ba:       mov    %rbx,%rcx
  4011bd:       mov    (%rax),%rdx
  4011c0:       mov    %rdx,0x8(%rcx)          ; 更新next指针
  4011c4:       add    $0x8,%rax               ; 下一个链表node
  4011c8:       cmp    %rsi,%rax               ; 是否移动到了最后一个node
  4011cb:       je     4011d2 <phase_6+0xde>
  4011cd:       mov    %rdx,%rcx
  4011d0:       jmp    4011bd <phase_6+0xc9>
  -----------------5. 更新链表next指针(按照映射后的数组元素连接) end-------------
  4011d2:       movq   $0x0,0x8(%rdx)          ; 走到这里,说明链表已经重新连接成功
  4011d9:
  4011da:       mov    $0x5,%ebp
  -----------------6. 验证链表是否是递减序列 start------------------------------
  4011df:       mov    0x8(%rbx),%rax
  4011e3:       mov    (%rax),%eax
  4011e5:       cmp    %eax,(%rbx)
  4011e7:       jge    4011ee <phase_6+0xfa>   ; 上一个node value >下一个node value才能通过,所以要求的递减序列
  4011e9:       callq  40143a <explode_bomb>
  4011ee:       mov    0x8(%rbx),%rbx          ; 指向下一个node
  4011f2:       sub    $0x1,%ebp               ; ebp = ebp-1 , ebp从5减到0
  4011f5:       jne    4011df <phase_6+0xeb>
  -----------------6. 验证链表是否是递减序列 end------------------------------
  4011f7:       add    $0x50,%rsp
  4011fb:       pop    %rbx
  4011fc:       pop    %rbp
  4011fd:       pop    %r12
  4011ff:       pop    %r13
  401201:       pop    %r14
  401203:       retq   
```

总体来说, 内存中存在一个6个节点的链表, 

比如:

```shell
41 -> 55 -> 15 -> 57 -> 12 -> 412 -> NULL
```

我们输入6个数字, 然后经过 **7 - 翻转**来控制链表的排序. 举个例子:

例如: 输入 

```shell
2 3 1 4 5 6
```

则代码会将 每个元素 = 7 - 每个元素. 则输入变为:

```shell
5 4 6 3 2 1
```

然后用 `5 4 6 3 2 1`控制链表的排列, 得到:

```shell
12 -> 57 -> 412 -> 15 -> 55 -> 41 -> NULL
```

最后检查排列的链表是否是降序的. 如果不是降序, 则bomb.

对于上面的链表,:

```shell
41 -> 55 -> 15 -> 57 -> 12 -> 412 -> NULL
```

正确答案应该为:

```shell
1 3 5 6 4 2
```

至于题目中的链表是什么, 查看内存地址空间**0x6032d0**即可知道.

![image-20200621225834535](https://cdn.jsdelivr.net/gh/ravenxrz/PicBed/img/image-20200621225834535.png)

即:

```shell
332 -> 168 -> 924 -> 681 -> 477 -> 443 -> NULL
```

所以答案:

```shell
4 3 2 1 6 5
```

## 7. 个人调试经验

第一次调试汇编， 整个过程并不算容易。前5道题目勉勉强强通过gdb调试出来了。 第6题， 只在gdb中调试， 调试了大半天也没能做出来。 因为gdb的显示空间有限， 而且无法添加注释。 所以后来选择通过objdump先把汇编dump出来， 在结合gdb一起看， 边看变写注释， 很快就能debug出来了。