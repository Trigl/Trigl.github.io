---
layout:     post
title:      "「剑指 Offer」面试题 11：数值的整数次方"
date:       2019-01-07 02:00:00
author:     "Ink Bai"
header-img: "/img/post/offer-sword-11.jpg"
tags:
    - 面试
---

> 给定一个 double 类型的浮点数 base 和 int 类型的整数 exponent，求 base 的 exponent 次方。不得使用库函数，同时不需要考虑大数问题。

求一个浮点数的整数次方，我们很容易想到的方法就是累乘，这样做的时间复杂度是 O(n)。我们也可以通过如下公式求 a 的 n 次方：

> n 为偶数：`a^n = a^(n/2) * a^(n/2)`
n 为奇数：`a^n = a^((n-1)/2) * a^((n-1)/2) * a`

假如输入的指数 exponent 是 32，如果我们用累乘的方法需要循环做 31 次乘法。但是使用上面的公式以后，我们可以先求平方，再在平方的基础上求 4 次方，在 4 次方的基础上求 8 次方，在 8 次方的基础上求 16 次方，最后在 16 次方的基础上求 32 次方，这样只需要做 5 次乘法就可以得到最终结果，时间复杂度是 O(logn)。

具体实现的时候我们还需要注意一些细节：

1. 在指数 exponent 为负数的时候，奇数 base 为 0 是没有意义的，这个时候要做异常处理。
2. 上面判断 base 为 0 的时候，不要直接用 `base == 0.0`，因为计算机表示小数（包括 float 和 double 型小数）都有误差，我们不能直接用等号来判断两个小数是否相等。如果两个小数的差的绝对值很小，比如小于 0.0000001，就可以认为它们相等，所以判断小数相等时需要我们自己实现一个 equal 方法。
3. 用右移运算符代替除以 2 的操作，用位与运算符代替求余运算（%）来判断一个数是奇数还是偶数，这是因为位运算的效率比乘除法及求余运算的效率要高很多。

综合考虑上面的内容最终实现的代码如下：

```java
public class Solution {
    public double Power(double base, int exponent) {
        if (equal(base, 0.0) && exponent < 0) {
            throw new RuntimeException("Base can't be zero when exponent is minus!");
        }

        int absExponent = exponent;
        if(exponent < 0) {
            absExponent = -exponent;
        }

        double result = powerWithUnsignedExponent(base, absExponent);
        if(exponent < 0) {
            result = 1.0 / result;
        }
        return result;
    }

    // 无符号数求整数次方
    private double powerWithUnsignedExponent(double base, int exponent) {
        if(exponent == 0) {
            return 1;
        }
        if (exponent == 1) {
            return base;
        }

        double result = powerWithUnsignedExponent(base, exponent >> 1);
        result *= result;
        if((exponent & 1) == 1) {
            result *= base;
        }
        return result;
    }

    // 自定义 euqal 方法来比较两个浮点数是否相等
    private boolean equal(double num1, double num2) {
        if (num1 - num2 > -0.0000001 && num1 - num2 < 0.0000001) {
            return true;
        } else {
            return false;
        }
    }
}
```
