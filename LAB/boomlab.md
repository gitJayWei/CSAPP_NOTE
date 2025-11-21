## Boom Lab

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

