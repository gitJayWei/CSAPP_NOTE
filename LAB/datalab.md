## Data Lab

##### 1.bitXor

- 要求仅适用按位否~和按位与&实现按位异或

~~~cpp
// 由于与运算仅在两个1时才返回1，所以对于其它的二元逻辑运算，可以找出其中所有返回1的组合，用~和&表达再用|连接，此题不允许使用|，可以用德摩根律转化

/*
 *a^b=
 *1.(a|b)&(~a|~b)
 *2.~(~a&~b)&~(a&b)
 *3.(a&~b)|(~a&b)
 */

// 只有x为0、y为1 或 x为1、y为0时 x异或y为1，即x异或y等价于(~x & y) | (x & ~y)，用德摩根律去掉|得：~(~(~x & y) & ~(x & ~y))
int bitXor(int x, int y)
{
    return ~(~(~x & y) & ~(x & ~y));
}

// 另一个角度：只有x为0、y为0 或 x为1、y为1时 !(x异或y)为0,即!(x异或y)等价于(~x & ~y) | (x & y)，去掉|再两边同时取反得~(~x & ~y) & ~(x &y)
int bitXor(int x, int y)
{
    return ~(~x & ~y) & ~(x & y);
}
~~~

##### 2.isTmax

- 要求判断x是否为补码表示下的最大整数

~~~cpp
// 即判断x是不是0x0111...11

// 不难发现其特征为x+1等于~x，但是小心x = -1时也满足该关系，需排除
// 排除x为-1：x+1不能为0
// x == y 等价于 !(x ^ y)
int isTmax(int x)
{
    int next = x + 1;
    int equal = !(next ^ x);//判断x+1是否等于x
    int notMinusOne = !!next;
    return equal & notMinusOne;
    // 即：return  (!(x + 1) ^ x) & !!(x + 1)
}

// 
int isTmax(int x)
{
    int tmp = (x << 1) + 1;
    return !(~tmp | (x >> 31));
}
~~~

##### 3.conditional

- 实现与三目运算符表达式 x ? y : z 

~~~cpp
// 一个数和111...11按位与结果为自身，一个数与000...00按位与结果为000...00
// x为真时返回y，x为假时返回z，因此应该return (mask & y) | (~make & z)
// 其中x为假时mask为000...00，x为真时mask为111...11
// 可以这样构造mask：mask = !x + ~0
int conditional(int x, int y, int z)
{
    int mask = !x + ~0;
    return (mask & y) | (~mask & z);
}
~~~

##### 4.isNonNegative

- 当 x >=0 时，返回1；否则返回0

~~~cpp
// 提取x的符号位并取反即可
int isNonNegative(int x)
{
    return ~(x >> 31);
}
~~~

##### 5.isGreater

- 当 x > y 时，返回1，否则返回0

~~~cpp
// 不能直接看x与y差值的符号，因为x为正y为负时，x-y可能发生溢出从而导致差值可能为负
// x-y无非四种情况：负减正，正减负，负减负，正减正；后两种情况可以看x-y的符号(不会溢出)
// 前面两种情况可以根据符号直接判定x与y的大小
// 还需要注意的是如果x等于y那么x-y的符号也为0，所以需要额外添加变量is_equal来判断
int isGreater(int x, int y) 
{
  int sign_x = x >> 31;
  int sign_y = y >> 31;
  int sign_diff = (x + (~y + 1)) >> 31;

  int diff_sign_case = (~sign_x) & sign_y;
  int same_sign_case = ~(sign_x ^ sign_y) & ~sign_diff;
  int is_equal = !(x ^ y);

  return !!(diff_sign_case | same_sign_case) & !is_equal;
}
~~~

##### 6.absVal

- 计算x的绝对值

~~~cpp
// 还是利用111..11 & x == x; 000..00 & x == 000...00
//      和000..00 | x == x; 111..11 | x == 111...11
// 如果x为负，~(x >> 31) & x为000..00; 如果x为正，则为x
// 如果x为负，(x >> 31) & (~x + 1)为~x + 1; 如果x为正，则为000..00
// 然后将二者按位或即可
int absVal(int x)
{
    return (~(x >> 31) & x) | ((x >> 31) & (~x + 1));
}

// 111..11 ^ x == ~x ; 000..00 ^ x == x
// x >> 31提取符号位后再与x异或，如果x为负得到~x需要再加1; 如果x为正得到x本身(或者说再加0)
// 可以加上 !(x >> 31)
int absVal(int x)
{
 	int sign = x >> 31;
    return (x ^ sign) + !sign;
}
~~~

##### 7. isPower2

- 判断x是否恰好等于 $2^n$ ( n>=0 ) ，如果等于则返回1，否则返回0

~~~cpp
// 负数和0显然不是2的n次方
// 2的n次方只有这些数：0100..00; 0010..00; .. ; 0000..01
// 如何判断x只有一个1：(x - 1) & x == 000..00; x - 1即x + ~0
int isPower2(int x)
{
    int is_zero = !!x;
    int is_negative = !(x >> 31);
    int is_single_bit = !((x + ~0) & x);
    return is_zero & is_negative & is_single_bit;
}
~~~

##### 8.unsigned float_neg

- 求浮点数f的相反数，参数uf表示浮点数f的二进制编码对应的无符号整数；返回值为-f的二进制编码对应的无符号整数(如果输入NaN，直接返回uf)
- 合法的运算符：全部有符号数和无符号数的运算符、||、&&、if 和 while

~~~cpp
// 根据IEEE754，float编码格式为：1位符号位+8位指数位+23位尾数位
// 首先需要提取指数和尾数
// 提取指数：右移23位后与0xFF按位与即可
// 提取尾数：与0x7FFFFF按位与即可
// 然后检查是否为NaN，即检查是否指数全1且尾数非0
// 然后翻转符号位，即与0x80000000异或实现翻转符号位
unsigned float_neg(unsigned uf)
{
    unsigned exp = (uf >> 23) & 0xFF;
    unsigned frac = uf & 0x7FFFFF;
    if((!(exp ^ 0xFF)) && (!!frac))
    {
		return uf;
    }
    return uf & 0x8000000;
}
~~~

##### 9.float_i2f

- 返回浮点数(float)x在计算机中的二进制编码所对应的无符号数

~~~cpp
unsigned float_i2f(int x)
{
    
}
~~~

#### 