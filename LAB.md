## CS:APP Lab

#### DataLab

[CSAPP DataLab - ImproperMelon - 博客园](https://www.cnblogs.com/immelon/articles/CSAPP_DataLab.html)

[【CSAPP】DataLab-腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/2389324?frompage=seopage&policyId=20240000)

[CSAPP深入理解计算机系统实验datalab解析_csapp data lab-CSDN博客](https://blog.csdn.net/u014124795/article/details/38471797)

[CSAPP: Data Lab 详解 | 码农家园](https://www.codenong.com/cs106267998/)

https://blog.csdn.net/weixin_42062229/article/details/93496909

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

#### BoomLab

- 首先使用反汇编工具对二进制文件bomb反汇编得到汇编文件bomb.asm

##### 1.phase_1

- 代码片段

  ~~~shell
  0000000000400ee0 <phase_1>:
  400ee0:		sub    $0x8,%rsp 		    	  # 在栈上开辟8个字节的空间
  400ee4:		mov    $0x402400,%esi 			  # %esi = 0x402400
  400ee9:		callq  401338 <strings_not_equal> # 调用函数
  400eee:		test   %eax,%eax				  # 判断%eax是否是0
  400ef0:		je     400ef7 <phase_1+0x17>	  # 如果%eax是0则跳到400ef7
  400ef2:		callq  40143a <explode_bomb>	  # 引爆炸弹
  400ef7:		add    $0x8,%rsp				  # 释放栈空间
  400efb:		retq   							  # 函数结束并返回
  ~~~

- 程序首先将位于地址0x402400作为函数strings_not_equal第一个形参，接着该调用函数，然后判断返回值，如果返回1则引爆炸弹，返回0则结束函数

- 由此推断，函数strings_not_equal必将接受另一个字符串(键盘输入，可知这就是拆弹密钥)，然后将其与位于地址0x402400的字符串判断是否相等，不相等则引爆炸弹

- 在gdb中执行命令

  ~~~shell
  (gdb) x/s 0x402400
  ~~~

  得到字符串“Border relations with Canada have never been better.”，这就是phase_1的密钥

- 输入上述字符串后输出

  ~~~shell
  Phase 1 defused. How about the next one?
  ~~~

  阶段1拆弹成功

##### 2.phase_2

- ~~~shell
  0000000000400efc <phase_2>:
  400efc:		push   %rbp			# 保护被调用者保护寄存器%rbp
  400efd:		push   %rbx			# 保护被调用者保护寄存器%rbx
  400efe:		sub    $0x28,%rsp	# 开辟40个字节的栈空间
  ~~~

- ~~~shell
  400f02:		mov    %rsp,%rsi				  # %rsi = %rsp
  400f05:		callq  40145c <read_six_numbers>  # 调用函数
  400f0a:		cmpl   $0x1,(%rsp)				  # 比较 1 与 (%rsp)
  400f0e:		je     400f30 <phase_2+0x34>	  # 如果(%rsp)等于1则跳到400f30
  400f10:		callq  40143a <explode_bomb>	  # 否则引爆炸弹
  ~~~

  程序将栈指针作为函数read_six_numbers的第二个参数，推测密钥是6个数字；(%rsp)必须是1否则就引爆炸弹，所以第一个数字是1
  
  注：第一个参数%rdi隐含为标准输入流，比如stdin
  
- 当第一个数字正确后，跳到400f30

  ~~~shell
  400f30:		lea    0x4(%rsp),%rbx    	  # %rbx = %rsp + 4
  400f35:		lea    0x18(%rsp),%rbp		  # %rbp = %rsp + 24
  400f3a:		jmp    400f17 <phase_2+0x1b>  # 无条件跳转到400f17
  ~~~

  将栈上第二个数字的地址放到%rbx中，并将地址%rsp+24放到%rbp中(该地址为六个数字结束的地址)

- 跳到400f17

  ~~~shell
  400f17:		mov    -0x4(%rbx),%eax       # %eax = (%rbx-4)
  400f1a:		add    %eax,%eax			 # %eax = %eax + %eax
  400f1c:		cmp    %eax,(%rbx)			 # 比较%eax与(%rbx)
  400f1e:		je     400f25 <phase_2+0x29> # 如果%eax=(%rbx)则跳到400f25
  400f20:		callq  40143a <explode_bomb> # 否则引爆炸弹
  ~~~

  由于%rbx指向栈上第二个数字，所以%eax值为第一个数字即1，翻倍后%eax值为2；%eax与(%rbx)相等才会跳转继续执行否则引爆炸弹，所以第二个数字是2

- 当第二个数字正确后，跳到400f25

  ~~~shell
  400f25:		add    $0x4,%rbx			 # %rbx = %rbx+4
  400f29:		cmp    %rbp,%rbx			 # 将%rbp与%rbx比较
  400f2c:		jne    400f17 <phase_2+0x1b> # %rbp不等于%rbx则跳到400f17
  400f2e:		jmp    400f3c <phase_2+0x40> # 否则跳到400f3c结束函数
  ...
  400f3c:		add    $0x28,%rsp   		 # 释放栈空间
  400f40:		pop    %rbx					 # 弹出%rbx	
  400f41:		pop    %rbp					 # 弹出%rbp
  400f42:		retq   						 # 函数结束并返回
  ~~~

  由于%rbx指向栈上第二个数字，+4后指向第三个数字；如果%rbp不等于%rbx则跳到400f17，显然这是一个循环；

  - 第一次跳到400f17，%rbx指向第三个数字，所以%eax等于第二个数字2，翻倍后等于4；(%rbx)必须等于%eax才不会引爆炸弹，所以第三个数字是4
  - 同理，每次循环%eax都会翻倍并等于下一个数字，所以剩余数字是8，16，32
  - 当%rbp(%rsp+24)和%rbx相等时不再跳转，循环结束，栈被清空，函数返回

- 输入1 2 4 8 16 32后输出

  ~~~shell
  That’s number 2, keep going!
  ~~~

  阶段2拆弹成功

##### 3.phase_3

- ~~~shell
  0000000000400f43 <phase_3>:
  400f43:		sub    $0x18,%rsp              # 开辟24字节的栈空间
  400f47:		lea    0xc(%rsp),%rcx		   # %rcx = %rsp+12
  400f4c:		lea    0x8(%rsp),%rdx     	   # %rdx = %rsp+8
  400f51:		mov    $0x4025cf,%esi		   # %esi = 0x4025cf
  400f56:		mov    $0x0,%eax 			   # %eax = 0
  400f5b:		callq  400bf0 <__isoc99_sscanf@plt> # 调用函数
  400f60:		cmp    $0x1,%eax			   # 比较%eax和1
  400f63:		jg     400f6a <phase_3+0x27>   # %eax > 1 则跳到400f6a
  400f65:		callq  40143a <explode_bomb>   # 否则引爆炸弹
  ~~~
  
  查询资料得，函数sscanf是C语言标准库里的函数，用于从字符串中读取格式化的数据，具体作用为从用户输入的字符串读取内容，根据格式字符串(存在%esi，第二个参数)解析输入的内容，再将解析得到的数据写入%rdx与%rcx指向的地址并返回成功匹配的参数数量
  
  - 在gdb中执行命令

    ~~~shell
    (gdb) x /s 0x4025cf
    ~~~
  
    得 “%d %d”
  
  由此可推测用户需要两个整数，分别存储在%rsp+8和%rsp+12处(如果sscanf读取到的整数少于两个会引爆炸弹)
  
- 假设成功跳转到400f6a(为方便描述，将%rsp+8和+12处的两个数记为num1和num2)

  ~~~shell
  400f6a:		cmpl   $0x7,0x8(%rsp)		 # 比较num1和立即数7
  400f6f:		ja     400fad <phase_3+0x6a> # 大于跳转到400fad
  ~~~

  如果num1 > 7就跳到400fad处引爆炸弹，num1 <= 7就继续执行

- 假设num1 <= 7
  
  ~~~shell
  400f71:		mov    0x8(%rsp),%eax         # %eax = (%rsp + 8)
  400f75:		jmpq   *0x402470(,%rax,8)	  # 跳转到间接地址
  ~~~

  程序将num1写入%eax然后跳转到间接地址0x402470 + %rax* 8即0x402470 + num1* 8处，这是一个跳转表，索引num1取值不同跳转到的地址不同
  
  - 使用gdb执行
  
    ~~~shell
    (gdb) x/8xg 0x402470
    ~~~
  
    得到跳转表：
  
    ~~~shell
    0x402470:       0x0000000000400f7c      0x0000000000400fb9
    0x402480:       0x0000000000400f83      0x0000000000400f8a
    0x402490:       0x0000000000400f91      0x0000000000400f98
    0x4024a0:       0x0000000000400f9f      0x0000000000400fa6
    ~~~
  
    由于num1介于0\~7，所以上面的地址依次对应num1=0\~7的情况
  
- 根据上面的跳转表的地址依次对应到num1的取值：
  
  ~~~shell
  400f7c:		mov    $0xcf,%eax             # num1 = 0, %eax = 207
  400f81:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400f83:		mov    $0x2c3,%eax			  # num1 = 2, %eax = 707
  400f88:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400f8a:		mov    $0x100,%eax			  # num1 = 3, %eax = 256
  400f8f:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400f91:		mov    $0x185,%eax			  # num1 = 4, %eax = 389
  400f96:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400f98:		mov    $0xce,%eax			  # num1 = 5, %eax = 206
  400f9d:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400f9f:		mov    $0x2aa,%eax			  # num1 = 6, %eax = 682	
  400fa4:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400fa6:		mov    $0x147,%eax			  # num1 = 7, %eax = 327
  400fab:		jmp    400fbe <phase_3+0x7b>  # 跳到400fbe
  400fad:		callq  40143a <explode_bomb>
  400fb2:		mov    $0x0,%eax
  400fb7:		jmp    400fbe <phase_3+0x7b>
  400fb9:		mov    $0x137,%eax			  # num1 = 1, %eax = 311
  400fbe:     ...
  ~~~
  
  (可以发现的是400fb2和400fb7处的指令永远也不会被执行，这里只能理解为逆向工程的干扰代码)
  
- 可以看到num1取不同值时，虽然%eax会被设置成不同的值，但是后续都会执行400fbe处的指令

  ~~~shell
  400fbe:		cmp    0xc(%rsp),%eax		 # 比较(%rsp+12)与%eax
  400fc2:		je     400fc9 <phase_3+0x86> # 相等则跳转到400fc9
  400fc4:		callq  40143a <explode_bomb> # 否则引爆炸弹
  400fc9:		add    $0x18,%rsp            # 清空栈
  400fcd:		retq   						 # 函数结束并返回
  ~~~
  
  这里程序比较num2和%eax，二者不相等就会引爆炸弹，相等才会结束函数；根据刚才num1取值不同%eax被设置成不同的值，可得密钥表：
  
  ~~~shell
  0 207；
  1 311；
  2 707；
  3 256；
  4 389；
  5 206；
  6 682；
  7 327；
  ~~~
  
- 任意输入上述跳转表中的一组数据，输出

  ~~~shell
  Halfway there! 
  ~~~

  阶段3拆弹成功

##### 4.phase_4

- ~~~shell
  000000000040100c <phase_4>:
  40100c:		sub    $0x18,%rsp			# 在栈上开辟24个字节的空间
  401010:		lea    0xc(%rsp),%rcx		# %rcx = %rsp + 12
  401015:		lea    0x8(%rsp),%rdx 		# %rdx = %rsp + 8
  40101a:		mov    $0x4025cf,%esi		# %esi = 0x4025cf
  40101f:		mov    $0x0,%eax			# %eax = 0
  401024:		callq  400bf0 <__isoc99_sscanf@plt> # 调用函数
  401029:		cmp    $0x2,%eax			# 比较2和%eax
  40102c:		jne    401035 <phase_4+0x29># 不相等则跳转到401035引爆炸弹
  ~~~
  
  和phase_3开头一样，sscnaf仍然是要从输入的字符串中格式化读取两个整数并存入%rsp+8和%rsp+12，输入的整数个数不是两个就引爆炸弹
  
- 假设成功读取到了两个整数(还是一样用num1, num2指代)
  
  ~~~shell
  40102e:		cmpl   $0xe,0x8(%rsp)		  # 比较num1和14
  401033:		jbe    40103a <phase_4+0x2e>  # num1 <= 14则跳到40103a
  401035:		callq  40143a <explode_bomb>  # 否则引爆炸弹
  40103a:		mov    $0xe,%edx			  # %edx = 14
  40103f:		mov    $0x0,%esi			  # %esi = 0
  401044:		mov    0x8(%rsp),%edi		  # %edi = num1
  401048:		callq  400fce <func4>		  # call func4
  
  40104d:		test   %eax,%eax
  40104f:		jne    401058 <phase_4+0x4c>
  401051:		cmpl   $0x0,0xc(%rsp)
  401056:		je     40105d <phase_4+0x51>
  401058:		callq  40143a <explode_bomb>
  40105d:		add    $0x18,%rsp
  401061:		retq   
  ~~~
  
  %edi, %esi, %edx是func的前三个形参，即func4初次调用为func4(num1, 0, 14);

- **func4**

  ~~~shell
  0000000000400fce <func4>:
  400fce:		sub   $0x8,%rsp 			# 在栈上开辟8字节的空间
  400fd2:		mov   %edx,%eax 			# %eax = %edx
  400fd4:		sub   %esi,%eax 			# %eax = %esi - %eax
  400fd6:		mov   %eax,%ecx 			# %ecx = %eax
  400fd8:		shr   $0x1f,%ecx 			# %ecx算术右移31位
  400fdb:		add   %ecx,%eax 			# %
  400fdd:		sar   %eax 					#
  400fdf:		lea   (%rax,%rsi,1),%ecx 	#
  400fe2:		cmp   %edi,%ecx				#
  400fe4:		jle   400ff2 <func4+0x24>	#
  400fe6:		lea   -0x1(%rcx),%edx		#
  400fe9:		callq  400fce <func4>		#
  400fee:		add   %eax,%eax				#
  400ff0:		jmp   401007 <func4+0x39>	#
  400ff2:		mov   $0x0,%eax				#
  400ff7:		cmp   %edi,%ecx				#
  400ff9:		jge   401007 <func4+0x39>	#
  400ffb:		lea   0x1(%rcx),%esi		#
  400ffe:		callq  400fce <func4>		#
  401003:		lea 0x1(%rax,%rax,1),%eax	#
  401007:		add   $0x8,%rsp				#
  40100b:		retq  						#
  ~~~

可以看到该函数有递归调用，为便于分析，根据汇编代码写出对应的C代码

函数共使用了五个寄存器，有三个是进行形参的传递，还有两个是函数内开辟的局部变量(其中一个是函数的返回值)

~~~c
// x: %edi y:%esi z:%edx k: %ecx t:%eax
int func4(int x, int y, int z)
{
    // 这里逐行翻译，不进行简化处理
    int t = z;   // mov   %edx,%eax
    t = t - y;   // sub   %esi,%eax
    int k = t;   // mov   %eax,%ecx
    k = k >> 31; // shr   $0x1f,%ecx
    t = t + k;   // add   %ecx,%eax
    t = t >> 1;  // sar   %eax
    k = t + y;   // lea   (%rax,%rsi,1),%ecx
    if (k <= x)  // cmp   %edi,%ecx
    {
        t = 0;      // mov   $0x0,%eax
        if (x <= k) // cmp   %edi,%ecx
        {
            return t;
        }
        else
        {
            y = k + 1;      // lea   0x1(%rcx),%esi
            func4(x, y, z); // callq  400fce <func4>
            t = 2 * t + 1;  // lea 0x1(%rax,%rax,1),%eax
            return t;
        }
    }
    else
    {
        z = k - 1;      // lea   -0x1(%rcx),%edx
        func4(x, y, z); // callq  400fce <func4>
        t = t + t;      // add   %eax,%eax
        return t;
    }
}
~~~

一开始z = 14；y = 0；x是输入的第一个数

- func4返回后

  ~~~shell
  test   %eax,%eax
  jne    401058 <phase_4+0x4c>
  cmpl   $0x0,0xc(%rsp)
  je     40105d <phase_4+0x51>
  callq  40143a <explode_bomb>
  add    $0x18,%rsp
  retq  
  ~~~

  如果返回值不为0，则跳转到phase_4+0x4c处即 40101a继续循环；如果返回值为0，则将输入的第二个数与0比较，若为0，则结束，若不为0，则引爆炸弹

  - 所以输入的第二个数为0，而输入的第一个数要使func4返回值为0

- 分析func4不难得x可取0，1，3，7....

- 随便选取一个输入，输出So you got that one.  Try this one. phase_4拆弹成功

##### 5.phase_5

- ~~~shell
  0000000000401062 <phase_5>:
  401062:		push   %rbx
  401063:		sub    $0x20,%rsp
  ~~~
  
  程序首先保护被调用者保存寄存器%rbx，然后在栈上开辟32字节的存储空间
  
- ~~~shell
  401067:		mov    %rdi,%rbx
  40106a:		mov    %fs:0x28,%rax
  401073:		mov    %rax,0x18(%rsp)
  ~~~

  将函数的第一个形参%rdi mov给%rbx；然后将金丝雀值mov给%rax又传递到栈上

- ~~~shell
  401078:		xor    %eax,%eax
  40107a:		callq  40131b <string_length>
  40107f:		cmp    $0x6,%eax
  401082:		je     4010d2 <phase_5+0x70>
  401084:		callq  40143a <explode_bomb>
  ~~~

  调用函数string_length并判断返回值是否是6，如果是继续执行，否则引爆炸弹

- 推测用户需要输入一个字符串，而该字符串长度必须为6

- 假设成功跳转到phase_5+0x70处

  ~~~shell
  4010d2:		mov    $0x0,%eax
  4010d7:		jmp    40108b <phase_5+0x29>
  ~~~

  将%eax置0并无条件跳转到phase_5+0x29

- phase_5+0x29

  ~~~shell
  40108b:		movzbl (%rbx,%rax,1),%ecx
  40108f:		mov    %cl,(%rsp)
  401092:		mov    (%rsp),%rdx
  401096:		and    $0xf,%edx
  401099:		movzbl 0x4024b0(%rdx),%edx
  4010a0:		mov    %dl,0x10(%rsp,%rax,1)
  4010a4:		add    $0x1,%rax
  4010a8:		cmp    $0x6,%rax
  4010ac:		jne    40108b <phase_5+0x29>
  ~~~

  将输入的字符串的第%rax个字节零扩展mov给%ecx；然后将该字符的低八位存入栈顶并mov给%rdx；通过与掩码0xf按位与提取字符的低4位；以该低4位索引，从0x4024b0表中查找对应字符并放入0x10(%rsp,%rax,1)处(栈上)，并将%rax不断+1使得处理完6个字符

  - 执行调试命令(gdb) print(char*)0x4024b0得：

    ~~~shell
    $1 = 0x4024b0 <array> "maduiersnfotvbylSo you think you can stop the bomb with ctrl-c, do you?"
    ~~~

- ~~~shell
  4010b3:		mov    $0x40245e,%esi
  4010b8:		lea    0x10(%rsp),%rdi
  4010bd:		callq  401338 <strings_not_equal>
  4010c2:		test   %eax,%eax
  4010c4:		je     4010d9 <phase_5+0x77>
  4010c6:		callq  40143a <explode_bomb>
  ~~~

  将地址0x40245e mov给%esi；然后将转化后的字符串的地址mov给%rdi并调用函数strings_not_equal；如果相等则继续执行，不相等则爆炸

  - 执行调试命令(gdb) x /s 0x40245e得被比较字符串为 “flyers”

- 假设两字符串相等，成功跳转到phase_5+0x77

  ~~~shell
  4010d9:		mov    0x18(%rsp),%rax
  4010de:		xor    %fs:0x28,%rax
  4010e7:		je     4010ee <phase_5+0x8c>
  4010e9:		callq  400b30 <__stack_chk_fail@plt>
  ~~~

  从栈上读取之前保存的金丝雀值并与原始数据进行异或，若相等则跳转继续执行，若不等说明栈被篡改并调用函数__stack_chk_fail@plt

- 假设栈没有被篡改，成功跳转到phase_5+0x8c

  ~~~shell
  4010ee:		add    $0x20,%rsp
  4010f2:		pop    %rbx
  4010f3:		retq  
  ~~~

  栈被释放，函数返回

- 所以这一阶段的关键在于如何构造一个字符串使得它按照上面的规则转化后与“flyers”相等

  - 按照上面的转换表，得未知字符串的低四位依次是1001、1111、1110、0101、0110、0111
  - 随便构造一个字符串满足上述要求即可，比如ionevw  

- 输入ionevw 后输出

  ~~~shell
  Good work!  On to the next... 
  ~~~

  阶段5拆弹成功

##### 6.phase_6

- ~~~shell
  00000000004010f4 <phase_6>:
  4010f4:		push   %r14		  #被调用者保护%r14
  4010f6:		push   %r13		  #被调用者保护%r13
  4010f8:		push   %r12		  #被调用者保护%r12
  4010fa:		push   %rbp		  #被调用者保护%rbp
  4010fb:		push   %rbx		  #被调用者保护%rbx
  4010fc:		sub    $0x50,%rsp #在栈上开辟80个字节的空间
  ~~~

- ~~~shell
  401100:		mov    %rsp,%r13	 # %r13 = %rsp
  401103:		mov    %rsp,%rsi     # %rsi = %rsp
  401106:		callq  40145c <read_six_numbers> 
  ~~~

  这里与phase_2一样，%rsi作为第二个参数，所以读入的六个数据被放到栈上，起始地址为%rsp；而第一个参数%rdi隐含为标准输入流

  为便于描述，将输入的六个数用num1~num6指代

- ~~~shell
  40110b:		mov	   %rsp,%r14 		# %r14指向栈顶
  40110e:		mov    $0x0,%r12d 		# %r12d = 0
  401114:		mov    %r13,%rbp 		# %rbp = %r13 = %rsp
  401117:		mov    0x0(%r13),%eax 	# %eax = (%r13) = num1
  40111b:		sub    $0x1,%eax 		# %eax = %eax - 1
  40111e:		cmp    $0x5,%eax 		# 比较%eax和5
  401121:		jbe    401128 <phase_6+0x34> # %eax <= 5则继续
  401123:		callq  40143a <explode_bomb> # 否则引爆炸弹
  401128:add    $0x1,%r12d 			# %r12d=%r12d+1=1
  40112c:cmp    $0x6,%r12d 			# 比较6与%r12d
  401130:je     401153 <phase_6+0x5f>	#如果%r12d等于6则跳到401153(循环结束)
  401132:mov    %r12d,%ebx
  401135:movslq %ebx,%rax
  401138:mov    (%rsp,%rax,4),%eax
  40113b:cmp    %eax,0x0(%rbp)
  40113e:jne    401145 <phase_6+0x51>	# %eax!=(%rbp)则继续
  401140:callq  40143a <explode_bomb>	# %eax=(%rbp)则爆炸
  401145:add    $0x1,%ebx				#%ebx+1
  401148:cmp    $0x5,%ebx
  40114b:jle    401135 <phase_6+0x41>	#%ebx<=5则跳到401135
  40114d:add    $0x4,%r13				#%r13+4
  401151:jmp    401114 <phase_6+0x20>	# 401114
  ~~~

  这段汇编共有四处跳转语句，其中401121和40113e处为判断的跳转；40114b和401151为循环的跳转；401130为结束循环的跳转；所以这是一个双层循环，利用两层嵌套的do-while语句，先把它们逐行翻译为C语言

  ~~~~c
  r14 = 0;
  r13 = 0;
  r12d = 0;
  while(1){
    rbp = r13;
    if(num[r13] - 1 > 5)
      goto bomb;
    r12d++;
    if(r12d == 6)
      break;
    for(ebx = r12d; ebx <= 5; ebx++){
      if(num[ebx] == num[rbp])
        goto bomb;
    }
    r13++;
  }
  ~~~~

  不难发现外层循环计数器为%r12d，内层循环计数器为%ebx；通过内层循环的if语句，可得这个双层循环的核心是验证输入的六个数每个都小于6且互不相同

- 401183对应的指令提到一个地址0x6032d0，输入命令

  ~~~shell
  (gdb) x /28 0x6032d0
  ~~~

  得

  ~~~shell
  0x6032d0 <node1>:       332     1       6304480 0
  0x6032e0 <node2>:       168     2       6304496 0
  0x6032f0 <node3>:       924     3       6304512 0
  0x603300 <node4>:       691     4       6304528 0
  0x603310 <node5>:       477     5       6304544 0
  0x603320 <node6>:       443     6       0       0
  0x603330:       0       0       0       0
  ~~~

  这是一个有六个节点的单链表，则按照链表继续分析

  ~~~shell
  40116f: mov    $0x0,%esi          # si = 0（循环计数器，遍历6个输入数）
  401174: jmp    401197			  #跳转到数值处理
  401176: mov    0x8(%rdx),%rdx     # rdx = 当前节点的next指针（遍历链表）
  40117a: add    $0x1,%eax          # eax++（计数器，记录遍历步数）
  40117d: cmp    %ecx,%eax          # 比较步数与索引（index）
  40117f: jne    401176             # 未到达目标索引则继续遍历
  401181: jmp    401188             # 到达目标节点，保存地址
  401183: mov    $0x6032d0,%edx     # edx = 链表头地址（初始节点）
  401188: mov    %rdx,0x20(%rsp,%rsi,2) # 将节点地址存入栈上临时数组
  40118d: add    $0x4,%rsi          # si +=4
  401191: cmp    $0x18,%rsi         # 检查是否存储完6个节点地址
  401195: je     4011ab             # 完成则构建链表，否则继续
  401197: mov    (%rsp,%rsi,1),%ecx # ecx = 输入数（num）
  40119a: cmp    $0x1,%ecx          # 检查num是否为1（可能对应索引6，需特殊处理）
  40119d: jle    401183             # 是则从链表头开始遍历，否则初始化计数器
  40119f: mov    $0x1,%eax          # eax = 1（计数器，从1开始遍历）
  4011a4: mov    $0x6032d0,%edx     # edx = 链表头地址
  4011a9: jmp    401176             # 开始遍历链表，寻找对应索引的节点
  ~~~

  分析知这一段的作用为用7依次减去原始数据得到索引，并按照该索引值获取链表对应的节点的地址，然后再依次将这六个地址存入栈上

  

- ~~~shell
  4011ab: mov    0x20(%rsp),%rbx   	# rbx = 数组第一个节点
  4011b0: lea    0x28(%rsp),%rax   	# rax = 数组第二个节点
  4011b5: lea    0x50(%rsp),%rsi    	# rsi = 数组末尾
  4011ba: mov    %rbx,%rcx         	# rcx = 当前节点
  4011bd: mov    (%rax),%rdx       	# rdx = 下一个节点
  4011c0: mov    %rdx,0x8(%rcx)    	# next指向下一个节点
  4011c4: add    $0x8,%rax        	# rax +=8指向下一个节点
  4011c8: cmp    %rsi,%rax            # 检查是否处理完所有节点
  4011cb: je     4011d2               # 是则设置最后一个节点的next为NULL
  4011cd: mov    %rdx,%rcx            # 更新当前节点为下一个节点
  4011d0: jmp    4011bd               # 继续连接节点
  4011d2: movq   $0x0,0x8(%rdx)       # 最后一个节点的next设为NULL
  ~~~

  依次连接节点

- ~~~shell
  4011da: mov    $0x5,%ebp         # ebp = 5
  4011df: mov    0x8(%rbx),%rax    # rax = 当前节点的next
  4011e3: mov    (%rax),%eax       # eax = 下一个节点的值
  4011e5: cmp    %eax,(%rbx)       # 比较当前节点与下一个节点
  4011e7: jge    4011ee            # 若当前值 ≥ 下一个值，继续
  4011e9: callq  40143a            # 否则引爆炸弹
  4011ee: mov    0x8(%rbx),%rbx    # rbx = next
  4011f2: sub    $0x1,%ebp         # ebp--
  4011f5: jne    4011df            # 未检查完则继续
  ~~~

  所以这一段是在检查链表有没有降序排列

- 由此，phase_6需要输入6个互不相同且介于1~6的数字，在经过给定方式的映射后的链表必须降序排列

- 根据给定映射关系(见上)分析得，需要输入的6个数字为4 3 2 1 6 5，输出

  ~~~shell
  Congratulations! You've defused the bomb!
  ~~~

  阶段6拆弹成功

##### 7.secret_phase



#### AttackLab

#### Architecture Lab

#### CacheLab

#### Shell Lab

前言：强烈建议先看完csapp第八章再做此实验，完整的tsh.c代码贴在文章末尾了

##### 1.准备知识 

1.  进程的概念、状态以及控制进程的几个函数（fork,waitpid,execve）。
1.  信号的概念，会编写正确安全的信号处理程序。
1.  shell的概念，理解shell程序是如何利用进程管理和信号去执行一个命令行语句。

##### 2.实验目的 

shell lab主要目的是为了熟悉进程控制和信号。具体来说需要比对16个test和rtest文件的输出，实现七个函数：

```java
void eval(char *cmdline)：分析命令，并派生子进程执行 主要功能是解析cmdline并运行
int builtin_cmd(char **argv)：解析和执行bulidin命令，包括 quit, fg, bg, and jobs
void do_bgfg(char **argv) 执行bg和fg命令
void waitfg(pid_t pid)：实现阻塞等待前台程序运行结束
void sigchld_handler(int sig)：SIGCHID信号处理函数
void sigint_handler(int sig)：信号处理函数，响应 SIGINT (ctrl-c) 信号 
void sigtstp_handler(int sig)：信号处理函数，响应 SIGTSTP (ctrl-z) 信号
```

##### 3.实验内容及操作步骤 

通过阅读实验指导书我们知道此实验要求我们完成tsh.c中的七个函数从而实现一个简单的shell，能够处理前后台运行程序、能够处理ctrl+z、ctrl+c等信号。

首先我们来看一下tsh.c具体内容。

首先定义了一些宏

```java
/* 定义了一些宏 */
#define MAXLINE    1024   /* max line size */
#define MAXARGS     128   /* max args on a command line */
#define MAXJOBS      16   /* max jobs at any point in time */
#define MAXJID    1<<16   /* max job ID */
```

定义了四种进程状态

```java
/* 工作状态 */
#define UNDEF 0 /* undefined */
#define FG 1    /* 前台状态 */
#define BG 2    /* 后台状态 */
#define ST 3    /* 挂起状态 */
```

然后定义了job\_t的任务的类，并且创建了jobs\[\]数组

```java
struct job_t {
            
   
     
                   /* The job struct */
    pid_t pid;              /* job PID */
    int jid;                /* job ID [1, 2, ...] */
    int state;              /* UNDEF, BG, FG, or ST */
    char cmdline[MAXLINE];  /* command line */
};
struct job_t jobs[MAXJOBS]; /* The job list */
```

接着是需要我们完成的七个函数定义

```java
void eval(char *cmdline)：分析命令，并派生子进程执行 主要功能是解析cmdline并运行
int builtin_cmd(char **argv)：解析和执行bulidin命令，包括 quit, fg, bg, and jobs
void do_bgfg(char **argv) 执行bg和fg命令
void waitfg(pid_t pid)：实现阻塞等待前台程序运行结束
void sigchld_handler(int sig)：SIGCHID信号处理函数
void sigint_handler(int sig)：信号处理函数，响应 SIGINT (ctrl-c) 信号 
void sigtstp_handler(int sig)：信号处理函数，响应 SIGTSTP (ctrl-z) 信号
```

下面就是一些辅助的函数

```java
int parseline(const char *cmdline, char **argv);   //获取参数列表，返回是否为后台运行命令
void sigquit_handler(int sig);  //处理SIGQUIT信号
void clearjob(struct job_t *job);  //清除job结构体 
void initjobs(struct job_t *jobs);  //初始化任务jobs[]
int maxjid(struct job_t *jobs);   //返回jobs链表中最大的jid号。
int addjob(struct job_t *jobs, pid_t pid, int state, char *cmdline);  //向jobs[]添加一个任务
int deletejob(struct job_t *jobs, pid_t pid);   //在jobs[]中删除pid的job
pid_t fgpid(struct job_t *jobs);  //返回当前前台运行job的pid号
struct job_t *getjobpid(struct job_t *jobs, pid_t pid);  //根据pid找到对应的job 
struct job_t *getjobjid(struct job_t *jobs, int jid);   //根据jid找到对应的job 
int pid2jid(pid_t pid);   //根据pid找到jid 
void listjobs(struct job_t *jobs);  //打印jobs
```

接着就是main函数，作用是在文件中逐行获取命令，并且判断是不是文件结束（EOF），将命令cmdline送入eval函数进行解析。我们需要做的就是逐步完善这个过程

接下来开始实验：

1.  使用make命令编译tsh.c文件（文件有所改变的话需要先使用make clean指令清空）

```java
bo@bo:~/shlab-handout$ make
gcc -Wall -O2    tsh.c   -o tsh
gcc -Wall -O2    myspin.c   -o myspin
gcc -Wall -O2    mysplit.c   -o mysplit
gcc -Wall -O2    mystop.c   -o mystop
gcc -Wall -O2    myint.c   -o myint
```

1.  使用make testXX指令比较traceXX.txt文件在编写的shell和reference shell的运行结果；或者也可以使用”./sdriver.pl -t traceXX.txt -s ./tsh -a “-p”
1.  如果在文件名前面加上r，则是执行标准的tshref，或者将tsh变为tshref。通过比对标准tshref和自制tsh的执行结果结果，可以观察tsh的功能是否正确。如果tsh的执行结果和tshref结果一致，说明结果是正确的

接下来我们开始补充函数

**eval（）函数**

函数功能：eval函数用于解析和解释命令行。eval首先解析命令行，如果用户请求一个内置命令quit、jobs、bg或fg（即内置命令）那么就立即执行。否则，fork子进程和在子进程的上下文中运行作业。如果作业正在运行前台，等待它终止，然后返回。

函数原型：`void eval(char *cmdline)`，传入的参数为cmdline，即命令行字符串

实现思路：仿照书上的eval函数写法和所需的功能来完成函数

1.  首先调用parseline函数解析命令行，如果为空直接返回，接着使用builtin\_cmd函数判断是否为内置命令，返回0说明不是内置命令，如果是内置命令直接执行。
1.  如果不是内置命令，那么先阻塞信号（具体在第四点分析），再调用fork创建子进程。在子进程中，首先解除阻塞，设置自己的id号，然后调用execve函数来执行job。
1.  父进程判断作业是否后台运行，是的话调用addjob函数将子进程job加入job链表中，解除阻塞，然后调用waifg函数等待前台运行完成。如果不在后台工作则打印进程组jid和子进程pid以及命令行字符串。
1.  因为子进程继承了他们父进程的阻塞向量，所以在执行新程序之前，子程序必须确保解除对SIGCHLD信号的阻塞。父进程必须使用sigprocmask在它派生子进程之前也就是调用fork函数之前阻塞SIGCHLD信号，之后解除阻塞；在通过调用addjob将子进程添加到作业列表之后，再次使用sigprocmask，解除阻塞。

完整代码：

```java
/**
 * 处理命令行输入，解析并执行命令（内置命令直接执行，外部命令通过子进程执行）
 * 管理进程前后台状态，处理信号竞争问题
 * @param cmdline 用户输入的命令行字符串
 */
void eval(char *cmdline)
{
    char* argv[MAXARGS];   // 存储解析后的命令及参数，供execve()函数使用
    int state = UNDEF;     // 进程状态：FG（前台）或BG（后台），初始为未定义
    sigset_t set;          // 信号集，用于阻塞/解除阻塞特定信号
    pid_t pid;             // 子进程ID

    // 1. 解析命令行，确定进程前后台状态
    // parseline函数：拆分cmdline到argv数组，返回1表示后台运行（以&结尾），0表示前台
    if(parseline(cmdline, argv) == 1)  
        state = BG;        // 后台进程
    else
        state = FG;        // 前台进程

    // 2. 处理空命令（如用户仅输入回车）
    if(argv[0] == NULL)    // 解析后无有效命令
        return;

    // 3. 若不是内置命令，则通过子进程执行外部命令
    if(!builtin_cmd(argv))  // builtin_cmd返回1表示内置命令（如cd、exit），直接在shell进程执行
    {
        // 4. 初始化信号集，准备阻塞关键信号（避免竞争条件）
        // 清空信号集
        if(sigemptyset(&set) < 0)
            unix_error("sigemptyset error");  // 错误处理函数（输出错误信息并退出）
        
        // 向信号集添加需要阻塞的信号：
        // SIGINT（Ctrl+C，中断信号）、SIGTSTP（Ctrl+Z，暂停信号）、SIGCHLD（子进程状态改变信号）
        if(sigaddset(&set, SIGINT) < 0 || 
           sigaddset(&set, SIGTSTP) < 0 || 
           sigaddset(&set, SIGCHLD) < 0)
            unix_error("sigaddset error");
        
        // 阻塞信号集中的信号：确保子进程创建并加入作业列表前，不会因信号处理导致状态不一致
        if(sigprocmask(SIG_BLOCK, &set, NULL) < 0)
            unix_error("sigprocmask error");

        // 5. 创建子进程执行外部命令
        if((pid = fork()) < 0)  // fork失败
            unix_error("fork error");
        else if(pid == 0)       // 子进程逻辑
        {
            // 子进程继承父进程的信号阻塞集，需解除阻塞以响应信号
            if(sigprocmask(SIG_UNBLOCK, &set, NULL) < 0)
                unix_error("sigprocmask error");
            
            // 设置子进程为新的进程组组长（进程组ID=子进程ID）
            // 目的：与shell进程组隔离，避免信号（如Ctrl+C）误影响shell
            if(setpgid(0, 0) < 0)
                unix_error("setpgid error");
            
            // 加载并执行外部命令：替换子进程的代码段为目标程序
            // argv[0]为命令路径，argv为参数列表，environ为环境变量
            if(execve(argv[0], argv, environ) < 0){
                printf("%s:Command not found\n", argv[0]);  // 命令不存在时提示
                exit(0);  // 子进程退出
            }
        }

        // 6. 父进程逻辑：将子进程添加到作业列表
        addjob(jobs, pid, state, cmdline);  // jobs为全局作业管理结构，记录所有进程状态

        // 解除信号阻塞：允许shell重新处理SIGINT、SIGTSTP、SIGCHLD信号
        if(sigprocmask(SIG_UNBLOCK, &set, NULL) < 0)
            unix_error("sigprocmask error");

        // 7. 根据进程状态（前后台）进行不同处理
        if(state == FG)
            waitfg(pid);  // 前台进程：shell阻塞等待其执行完毕
        else
            // 后台进程：直接输出作业信息（作业ID、进程ID、命令行），不等待
            printf("[%d] (%d) %s", pid2jid(pid), pid, cmdline);  // pid2jid：将进程ID映射为作业ID
    }
    return;
}
```

注意：

1.  每个子进程必须有自己独一无二的进程组id，通过在fork()之后的子进程中Setpgid(0,0)实现，这样当向前台程序发送ctrl+c或ctrl+z命令时，才不会影响到后台程序。如果没有这一步，则所有的子进程与当前的tsh shell进程为同一个进程组，发送信号时，前后台的子进程均会收到。
1.  在fork()新进程前要阻塞SIGCHLD信号，防止出现竞争，这是经典的同步错误，如果不阻塞会出现子进程先结束从jobs中删除，然后再执行到主进程addjob的竞争问题。

**builtin\_cmd函数 **

函数功能：识别并执行内置命令: quit, fg, bg, 和 jobs。

函数原型：`int builtin_cmd(char **argv)`，参数为argv 参数列表

实现思路：

1.  当命令行参数为quit时，直接终止shell
1.  当命令行参数为jobs时，调用listjobs函数，显示job列表
1.  当命令行参数为bg或fg时，调用do\_bgfg函数，执行内置的bg和fg命令
1.  不是内置命令时返回0

完整代码：

```java
/**
 * 判断命令是否为shell内置命令，并执行相应操作
 * @param argv 解析后的命令及参数数组（argv[0]为命令名）
 * @return 1 表示是内置命令并已处理；0 表示非内置命令
 */
int builtin_cmd(char **argv)
{
    // 1. 处理"quit"命令：退出shell
    if(!strcmp(argv[0], "quit"))  // 比较命令名是否为"quit"（strcmp返回0表示相等，!0为真）
        exit(0);                  // 退出当前shell进程（终止程序）

    // 2. 处理"bg"（后台运行）和"fg"（前台运行）命令
    else if(!strcmp(argv[0], "bg") || !strcmp(argv[0], "fg"))  // 命令为"bg"或"fg"
        do_bgfg(argv);  // 调用专门函数处理作业的前后台切换（如将暂停的作业继续运行）

    // 3. 处理"jobs"命令：列出所有后台作业状态
    else if(!strcmp(argv[0], "jobs"))  // 命令为"jobs"
        listjobs(jobs);  // 遍历作业列表（jobs），打印所有作业的ID、状态、命令行等信息

    // 4. 非内置命令：返回0，由调用者（eval）创建子进程执行外部命令
    else
        return 0;     /* not a builtin command */

    // 若为内置命令，处理完成后返回1
    return 1;
}
```

**do\_bgfg函数 **

函数功能：实现内置命令bg 和 fg

首先要明确的是bg和bg的作用

> `bg <job>`:将停止的后台作业更改为正在运行的后台作业。通过发送SIGCONT信号重新启动`<job>`，然后在后台运行它。`<job>`参数可以是PID，也可以是JID。ST -> BG
>
> `fg <job>`:将已停止或正在运行的后台作业更改为前台正在运行的作业。通过发送SIGCONT信号重新启`<job>`，然后在前台运行它。`<job>`参数可以是PID，也可以是JID。ST -> FG，BG -> FG

函数原型：`void do_bgfg(char **argv)`，参数为argv 参数列表

实现思路：

1.  判断argv\[\]是否带%，若为整数则传入pid，若带%则传入jid。接着调用getjobjid函数来获得对应的job结构体，如果返回为空，说明列表中并不存在jid的job，要输出提示。
1.  使用strcmp函数判断是bg命令还是fg命令
1.  若是bg，使目标进程重新开始工作，设置状态为BG(后台)，打印进程信息
1.  若是fg，使目标进程重新开始工作，设置状态为FG(前台)，等待进程结束

完整代码：

```java
/**
 * 处理bg和fg命令，实现作业的前后台切换及恢复运行
 * @param argv 解析后的命令及参数（argv[0]为"bg"或"fg"，argv[1]为目标作业的PID或%jobid）
 */
void do_bgfg(char **argv)
{
    int num;                     // 存储转换后的PID或JobID
    struct job_t *job;           // 指向目标作业的指针

    // 1. 检查是否提供了目标作业参数
    if(!argv[1]){
        // 未提供参数时报错（需要PID或%jobid）
        printf("%s command requires PID or %%jobid argument\n", argv[0]);
        return ;
    }

    // 2. 解析参数：区分JobID（%开头）和PID（纯数字）
    if(argv[1][0] == '%'){
        // 解析JobID（格式为%数字）
        // 将%后面的字符串转为整数，基数为10
        if((num = strtol(&argv[1][1], NULL, 10)) <= 0){
            // 转换失败（非数字或结果≤0），提示参数格式错误
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
        // 根据JobID从作业列表中查找作业
        if((job = getjobjid(jobs, num)) == NULL){
            printf("%%%d: No such job\n", num);  // 未找到对应作业
            return;
        }
    } else {
        // 解析PID（纯数字格式）
        if((num = strtol(argv[1], NULL, 10)) <= 0){
            // 转换失败，提示参数格式错误
            printf("%s: argument must be a PID or %%jobid\n", argv[0]);
            return;
        }
        // 根据PID从作业列表中查找作业
        if((job = getjobpid(jobs, num)) == NULL){
            printf("(%d): No such process\n", num);  // 未找到对应进程
            return;
        }
    }

    // 3. 处理bg命令：将作业放入后台运行
    if(!strcmp(argv[0], "bg")){
        job->state = BG;  // 更新作业状态为后台运行
        // 发送SIGCONT信号到整个进程组（-job->pid表示进程组ID），恢复暂停的作业
        if(kill(-job->pid, SIGCONT) < 0)
            unix_error("kill error");  // 信号发送失败时的错误处理
        // 打印后台作业信息（作业ID、进程ID、命令行）
        printf("[%d] (%d) %s", job->jid, job->pid, job->cmdline);
    }
    // 4. 处理fg命令：将作业放入前台运行
    else if(!strcmp(argv[0], "fg")) {
        job->state = FG;  // 更新作业状态为前台运行
        // 发送SIGCONT信号到进程组，恢复暂停的作业
        if(kill(-job->pid, SIGCONT) < 0)
            unix_error("kill error");
        // 等待前台作业完成（shell阻塞直到该作业结束）
        waitfg(job->pid);
    }
    // 5. 内部错误处理（理论上不会执行）
    else {
        puts("do_bgfg: Internal error");
        exit(0);
    }

    return;
}
```

**waitfg（）函数 **

函数功能：等待一个前台作业结束，或者说是阻塞一个前台的进程直到这个进程变为后台进程

函数原型：`void waitfg(pid_t pid)`，参数为进程ID

实现思路：判断当前的前台的进程组pid是否和当前进程的pid是否相等，如果相等则sleep直到前台进程结束。

完整代码：

```java
void waitfg(pid_t pid)
{
    sigset_t mask;
    sigemptyset(&mask);
    while (fgpid(jobs) != 0)
    {
        sigsuspend(&mask);
    }
    return;
}

```

**sigchld\_handler函数 **

函数功能：处理SIGCHILD信号

函数原型：`void sigchld_handler(int sig)`，参数为信号类型

首先了解一下父进程回收子进程的过程：当一个子进程终止或者停止时，内核会发送一个SIGCHLD信号给父进程。因此父进程必须回收子进程，以避免在系统中留下僵死进程。父进程捕获这个SIGCHLD信号，回收一个子进程。一个进程可以通过调用 waitpid 函数来等待它的子进程终止或者停止。如果回收成功，则返回为子进程的 PID, 如果 WNOHANG, 则返回为 0, 如果其他错误，则为 -1。

实现思路：

 *  用while循环调用waitpid直到它所有的子进程终止。
 *  检查己回收子进程的退出状态

    *  WIFSTOPPED：：引起返回的子进程当前是被停止的
    *  WIFSIGNALED：子进程是因为一个未被捕获的信号终止
    *  WIFEXITED：子进程通过调用exit 或者return正常终止
 *  然后分别用WSTOPSIG，WTERMSIG，WEXITSTATUS提取以上三个退出状态。注意如果引起返回的子进程当前是被停止的进程，那么要将其状态设置为ST

完整代码：

```java
/**
 * SIGCHLD信号处理函数：响应子进程状态变化（终止或暂停）
 * 负责更新作业状态、删除已结束作业，并输出相关信息
 * @param sig 信号值（此处应为SIGCHLD）
 */
void sigchld_handler(int sig)
{
    int status, jid;                // status：子进程状态；jid：作业ID
    pid_t pid;                      // 子进程PID
    struct job_t *job;              // 指向子进程对应的作业结构

    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigchld_handler: entering");

    /*
    以非阻塞方式等待所有子进程状态变化
    waitpid参数说明：
        -1：等待任意子进程
        &status：存储子进程状态信息
        WNOHANG | WUNTRACED：非阻塞，且捕获子进程“终止”和“暂停”状态
    循环处理：确保所有状态已改变的子进程都被处理
    */
    while((pid = waitpid(-1, &status, WNOHANG | WUNTRACED)) > 0){

        // 查找子进程对应的作业，若未找到则提示异常
        if((job = getjobpid(jobs, pid)) == NULL){
            printf("Lost track of (%d)\n", pid);  // 无法识别的子进程
            return;
        }

        jid = job->jid;  // 获取作业ID

        // 情况1：子进程被信号暂停（如Ctrl+Z发送的SIGTSTP）
        if(WIFSTOPPED(status)){
            // 输出暂停信息：作业ID、进程ID、暂停信号
            printf("Job [%d] (%d) stopped by signal %d\n", jid, job->pid, WSTOPSIG(status));
            job->state = ST;  // 更新作业状态为“暂停（ST）”
        }
        // 情况2：子进程正常终止（如调用exit或return）
        else if(WIFEXITED(status)){
            // 从作业列表中删除该作业
            if(deletejob(jobs, pid))
                if(verbose){  // 调试模式：输出删除和正常终止信息
                    printf("sigchld_handler: Job [%d] (%d) deleted\n", jid, pid);
                    printf("sigchld_handler: Job [%d] (%d) terminates OK (status %d)\n", 
                           jid, pid, WEXITSTATUS(status));
                }
        }
        // 情况3：子进程被未捕获的信号终止（如SIGINT、SIGKILL）
        else {
            // 从作业列表中删除该作业
            if(deletejob(jobs, pid)){
                if(verbose)
                    printf("sigchld_handler: Job [%d] (%d) deleted\n", jid, pid);
            }
            // 输出终止信息：作业ID、进程ID、终止信号
            printf("Job [%d] (%d) terminated by signal %d\n", jid, pid, WTERMSIG(status));
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigchld_handler: exiting");

    return;
}
```

**sigint\_handler函数 **

函数功能：捕获SIGINT信号

函数原型：`void sigchld_handler(int sig)`，参数为信号类型

实现思路：

1.  调用函数fgpid返回前台进程pid
1.  如果当前进程pid不为0，那么调用kill函数发送SIGINT信号给前台进程组
1.  在2中调用kill函数如果返回值为-1表示进程不存在。输出error

完整代码：

```java
/**
 * SIGINT信号处理函数：响应Ctrl+C，中断前台作业的执行
 * @param sig 信号值（此处应为SIGINT）
 */
void sigint_handler(int sig)
{
    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigint_handler: entering");

    // 获取当前前台作业的进程ID（PID），无前台作业则返回0
    pid_t pid = fgpid(jobs);

    // 若存在前台作业，向前台进程组发送SIGINT信号
    if(pid){
        /* 
        发送SIGINT给前台进程组内的所有进程：
        - -pid 表示目标是进程组（进程组ID = 前台进程PID）
        - 确保前台作业的主进程及其子进程都收到中断信号
        */
        if(kill(-pid, SIGINT) < 0)
            unix_error("kill (sigint) error");  // 信号发送失败时的错误处理

        // 调试模式：输出前台作业被中断的信息
        if(verbose){
            printf("sigint_handler: Job (%d) killed\n", pid);
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigint_handler: exiting");

    return;
}
```

**sigtstp\_handler函数 **

函数功能：同sigint\_handler差不多，捕获SIGTSTP信号

函数原型：`void sigtstp_handler(int sig)`，参数为信号类型

首先了解一下SIGTSTP的作用：SIGTSPT信号默认行为是停止直到下一个 SIGCONT，是来自终端的停止信号，在键盘上输入 CTR+Z会导致一个 SIGTSPT信号被发送到外壳。外壳捕获该信号，然后发送SIGTSPT信号到这个前台进程组中的每个进程。在默认情况下，结果是停止或挂起前台作业。

实现思路：

1.  用fgpid(jobs)获取前台进程pid，判断当前是否有前台进程，如果没有直接返回。
1.  用kill(-pid,sig)函数发送SIGTSPT信号给前台进程组。

完整代码：

```java
/**
 * SIGTSTP信号处理函数：响应Ctrl+Z，暂停前台作业的执行
 * @param sig 信号值（此处应为SIGTSTP）
 */
void sigtstp_handler(int sig)
{
    // 调试模式：输出进入信号处理函数的信息
    if(verbose)
        puts("sigstp_handler: entering");

    // 获取当前前台作业的进程ID（PID），无前台作业则返回0
    pid_t pid = fgpid(jobs);
    // 根据PID查找对应的作业结构（用于获取作业ID等信息）
    struct job_t *job = getjobpid(jobs, pid);

    // 若存在前台作业，向前台进程组发送SIGTSTP信号
    if(pid){
        /*
        发送SIGTSTP给前台进程组内的所有进程：
        - -pid 表示目标是进程组（进程组ID = 前台进程PID）
        - 确保前台作业的主进程及其子进程都被暂停
        */
        if(kill(-pid, SIGTSTP) < 0)
            unix_error("kill (tstp) error");  // 信号发送失败时的错误处理

        // 调试模式：输出前台作业被暂停的信息（作业ID、进程ID）
        if(verbose){
            printf("sigstp_handler: Job [%d] (%d) stopped\n", job->jid, pid);
        }
    }

    // 调试模式：输出退出信号处理函数的信息
    if(verbose)
        puts("sigstp_handler: exiting");

    return;
}
```

注意：使用kill函数，如果 pid 小于零才会发送信号sig 给进程组中的每个进程，因此这里使用-pid

至此tsh.c文件完成

#### Malloc Lab

#### Link Lab

#### Attack Lab





