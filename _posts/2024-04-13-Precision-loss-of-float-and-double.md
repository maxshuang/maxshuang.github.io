---
layout: post
title: Precision Loss of Float
subtitle: picture from https://www.pexels.com/search/wild%20animals/
author: maxshuang
categories: Program
banner:
  image: /assets/images/post/programming-float/pexels-monicore-134058.jpg
  opacity: 0.618
  background: "#000"
  height: "55vh"
  min_height: "38vh"
  heading_style: "font-size: 3.00em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: Program
---

## Background

今天讲个轻松一点的话题，关于浮点型数的精度丢失问题，它源于我对**3 个问题**的疑问：
1. 二进制表示十进制小数的天然精度丢失，比如十进制 0.1 对应的二进制是 0.00(1100)(1100)...
2. 浮点数本身对小数点后可表示位数有限制，导致对小数点后数据本身就会截断，比如 1.123456789 可能就只能表示成 1.123457
3. 这个比较诡异，float 最大可表示范围可以达到 $2^{127}$，但是实际上当一个整数大于 $2^{24}$ 的时候，还是出现了*整数精度丢失问题*，导致一些程序在用 float 解析一些大数时需要使用 *decimal string* 来表示 "100000123"。

在 [online compiler](https://www.programiz.com/cpp-programming/online-compiler/) 上运行以下测试程序：
```
#include <iostream>

int main() {
    //1. binary format precision loss
    float small_ft=0.3;
    std::cout << "1. original float: 0.3" << std::endl;
    std::cout << "1. after assigned to float(rounded):" << small_ft << std::endl;
    std::cout << "1. check equity:" << (small_ft==0.3) << std::endl;
    
    //2. significand precision loss
    float big_sf=1.123456789;
    std::cout << "2. original float: 1.123456789" << std::endl;
    std::cout << "2. after assigned to float:" << big_sf << std::endl;
    
    //3. big int precision loss in float
    int big_int=100000123;
    std::cout << "3. original big int:" << big_int << std::endl;
    
    float ft_int=big_int;
    std::cout << "3. after assigned to float:" << ft_int << std::endl;
    std::cout << "3. after assigned to float(int):" << (int)ft_int << std::endl;
    return 0;
}
```
```
Result:
1. original float: 0.3
1. after assigned to float(rounded):0.3
1. check equity:0
2. original float: 1.123456789
2. after assigned to float:1.123457
3. original big int:100000123
3. after assigned to float:1e+08
3. after assigned to float(int):100000120
```

如果说 1,2 都是符合直觉的，任何表示方式都会有一定的精度限制，那 3 就有点反直觉了，大整数不需要小数点后的精度，并且也没有超过 float 可表示的最大正整数范围，但还是出现了精度丢失，这也**太诡异**了。

## float 标准
为了说明这个问题，我们先快速 walk through 下 [float 标准](http://www.dsc.ufcg.edu.br/~cnum/modulos/Modulo2/IEEE754_2008.pdf) 提供的性质。
![float standard](/assets/images/post/programming-float/float-format.png)

对于 32 bits float(double 同理)，其中最高 1 bit 表示 $sign$, following 8 bits 表示 $exponent$(base 2), lowest 23 bits 表示 $significand$，也就是我们说的精度。

这里需要特别注意的点： *这是科学计数法表示形式*。  
举个例子(科学计数法表示 base 10)：   
$0.000123$ => $1.23 * 10^{-4}$
$22345$ => $2.2345*10^{4}$

对于 float 标准所定义的规则，$exponent$ 部分的 base 是 2, 而 $significand$ 部分是科学计数法表示的小数点后有效位数。这个和我们平时的概念还有点不一样，比如我们会认为 $123.456$ 的 significand 是 "456", 但是对于 float 而言它是 "23456"。

所以大体上 $123.456$ 表示成 $1.23456 * 10^{2}$ => $-1^{s}* 1.significand * 2^{exponent}$, 这里的 exponent 可以想象将 $123.456$ 一直除以 2, 直到整数部分为 1。  
=>  $-1^{0}* 1.929 * 2^{6}$，$exponent$=6, $significand$=0.929。

实际的 float 还不是这样的，有一些细节：
1. $exponent$ 有 -127 bias， 也就是原来的 8 bits 表示 [0, 255), 而实际上需要计算 $exponent-127$, 表示 [-127, 128)。
2. float 标准预留了 $exponent$ 为全 0 值和全 1 值，分别表示 *小到无法表示的值* infinity 和 *大到无法表示的值* NaN。
3. 实际的 exponent 可表示范围为 [-126, 127)，-127(0-127) 和 128(255-127) 被预留成其他含义了。

所以实际的 float 表示形式为 $-1^{s}* 1.significand * 2^{exponent-127}$。
从这个形式上看，float 的可表示范围会在 $[-2^{127}, 2^{127}]$ 附近， 而可表示的最小精度在 $2^{-126}$ 左右，更小精度值就无法表示了。

## 为什么会出现精度丢失
有了上面的 float 标准做铺垫，我们来看看 1,2,3 不同场景下的精度丢失。

**场景1：float 表示 0.3**  
这个场景是*二进制天然无法表示某些 10 进制小数*。

0.3 转成十进制科学计数法为 $3 * 10^{-1}$, 转成二进制科学计数法 $1.2 * 2^{-2}$  
=> $sign$=0, $exponent$=125(-2+127),   $significand=b00110011 001100110011010$  
其中 significand 是对 0011 无限循环模式 round up 到 23 bits。

我们验证下：
```
#include <iostream>

int main() {
    union {
       float input;
       int output;
    }u;
    u.input=0.3;
    std::cout << "sign:" << (u.output >> 31) << std::endl;
    std::cout << "exponent:" << (u.output >> 23) << std::endl;
    
    int tmp= (u.output & 0x7FFFFF);
    std::cout << "significand:";
    for(int t=22; t>=0; --t) {
        std::cout << ((tmp>>t)&0x1);
    }
    std::cout << std::endl;
}
```
```
Result:
sign:0
exponent:125
significand:00110011001100110011010
```

**场景2：float 表示 1.123456789**  
这个场景是 23 bits significand 决定它能表达的最大精度是有限的，$2^{23}\approx8000000$，所以它可以表达大概 6～7 位小数。

1.123456789 表达成二进制科学计数法 $1.123456789 * 2^{0}$  
=> $sign$=0, $exponent$=127(0+127), $significand=b00110011 001100110011010\approx1.12346$

我们验证下：
```
#include <iostream>

int main() {
    union {
       float input;
       int output;
    }u;
    u.input=0.3;
    std::cout << "sign:" << (u.output >> 31) << std::endl;
    std::cout << "exponent:" << (u.output >> 23) << std::endl;
    
    int tmp= (u.output & 0x7FFFFF);
    std::cout << "significand:";
    for(int t=22; t>=0; --t) {
        std::cout << ((tmp>>t)&0x1);
    }
    std::cout << std::endl;
}
```
```
Result:
original float: 1.123456789
after assigned to float(rounded):1.12346

sign:0
exponent:127
significand(int):1035631
significand:00011111100110101101111
```

**场景3：float 表示 16800013**  
场景 3 是比较 counterintuitive 的场景，float 标准可以表示 $[-2^{127}, 2^{127}]$ 范围的浮点数，但不代表在可表示范围内都不会发生精度丢失。

考虑 16800013 的十进制科学计数法为 $1.6800013 * 10^{7}$, 由于 float 的二进制表示法中 significand 只能表示 6～7 位十进制小数点后的精度，所以 0.6800013 一定是会被截断的。

让我们直接切换成二进制科学计数法表示 $1.001358807086*2^{24}$, 这里的 0.001358807086 是我们截断之后的十进制小数，用 23 bits 的二进制表示后会进一步截断，所以最后用 float 表示出来的整数就是截断之后的。

我们验证下：

```
#include <iostream>

int main() {
    union {
       float input;
       int output;
    }u;
    u.input=16800013;
    
    std::cout << "original float: 16800013" << std::endl;
    std::cout << "after assigned to float(rounded):" << u.input << std::endl << std::endl;
    std::cout << "float to int:" << (int)u.input << std::endl << std::endl;
    
    std::cout << "sign:" << (u.output >> 31) << std::endl;
    std::cout << "exponent:" << (u.output >> 23) << std::endl;
    
    int tmp= (u.output & 0x7FFFFF);
    std::cout << "significand:";
    for(int t=22; t>=0; --t) {
        std::cout << ((tmp>>t)&0x1);
    }
    std::cout << std::endl;
}
```
```
original float: 16800123
after assigned to float(rounded):1.68e+07
float to int:16800012

sign:0
exponent:151
significand:00000000010110010000110
```

从结果可以看出，将 16800013 赋值给 float 之后，再转换成 int 就会变成 16800012。

我特意选择了 16800013， 因为它是比 $2^{24}$ 稍大的数，而 24 bits 表示的是 23 bits significand + 1 bit of leading 1, 所以 $[-2^{24}, 2^{24}]$ 这个范围内的整数都可以完整表示。

大于 $2^{24}$，则其二进制表示的 bits 数就会大于 24, 将小数点移动到 leading 1 之后，之后的 bits 数大于 23, 就会被 significand 截断。
```
Decimal: 16800013
Hex:     0x100590D
Binary:  0001 [0000 0000 0101 1001 0000 1101]
float:      1.[0000 0000 0101 1001 0000 110] * 2^(24) 
```

## double 标准
双精度的 double 和 float 分析流程是一样的，可以参照上面的分析，其标准为([origin](https://www.geeksforgeeks.org/difference-between-float-and-double/))：  
![double-standard](/assets/images/post/programming-float/double-standard.png)

在 double 的标准下，由于其 significand 最多可以有 63 bit，所以对于正整数而言，在大概 $10^18$ 之内都不会出现截断的问题。这也是 lua 可以使用 number 类型表示整数和浮点数的原因。(from Programming in Lua)
 ![lua-number](/assets/images/post/programming-float/lua-number.png)

## Conclusion
float 和 double 是编程语言非常常见的内置类型，但是它的编码方式决定了它的含义没有整型和字符串类型那样直观。

在大部分场景中我们都不会碰到和关注 float 精度丢失的问题，因为 float 本身的 round up 可能掩盖了问题，但在一些计费和严重依赖科学计算的领域，就需要特别关注浮点型精度丢失的各种场景。

本文重点解释了 float 在表示特殊小数，长有效位数和大整数场景出现精度丢失的原因。