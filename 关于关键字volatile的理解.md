做嵌入式C开发的相信都使用过一个关键字volatile,特别是做底层开发的。假设一个GPIO的数据寄存器地址是0x50000004,我们一般会定义一个这样的宏：
```c
#define GDATA	*((volatile unsigned int*)0x50000004)
```
在面试的时候也会被问到过volatile关键字起什么作用？

>网络上的回答一般是防止被编译器优化，或者还会加一点就是访问被volatile修饰的变量时，强制访问内存中的值，而不是缓存中的。

我对上面的回答一直存在误解，以为是：因为被编译器优化，所以导致存取变量时是存取变量在cache中的缓存。

## volatile官方说明

[点我](http://tigcc.ticalc.org/doc/keywords.html#volatile)

**volatile**

**Indicates that a variable can be changed by a background routine.**

Keyword volatile is an extreme opposite of const. It indicates that a variable may be changed in a way which is absolutely unpredictable by analysing the normal program flow (for example, a variable which may be changed by an interrupt handler). This keyword uses the following syntax:

```c
volatile data-definition;
```

Every reference to the variable will reload the contents from memory rather than take advantage of situations where a copy can be in a register.

----------

**volatile**
**表明变量能被后台程序修改**

关键字volatile和const是完全相反的。它表明变量可能会通过某种方式发生改变，而这种方式是你通过分析正常的程序流程完全预测不出来的。（例如，一个变量可能被中断处理程序修改）。关键字使用语法如下：

```c
volatile data-definition;
```

每次对变量内容的引用会重新从内存中加载而不是从变量在寄存器里面的拷贝加载。

>我的理解：以中断处理程序修改变量解释可能不太合适，以GPIO为例最合适。首先什么是变量？变量就是一块编了地址的内存区域。GPIO的数据寄存器有一个地址，大小一般为32bit，所以这个数据寄存器可以认为就是一个变量，我们可以读写它。如果GPIO设置为输入，修改GPIO数据寄存器这个变量的就是这个GPIO的引脚，不管你如何分析你的程序，你不可能知道这个GPIO数据寄存器里面值是多少，你得读它。你此刻读到数据和下一刻读到的完全可能是不一样的。简单的说就是你要的数据不同步。使用volatile修饰后，会强制你每次引用GPIO寄存器对应的变量时都会去它的寄存器里面读。
----------

为了搞清楚volatile到底是怎么影响编译器行为的，需要以具体的例子来说明：

## 防止编译器优化掉操作变量的语句

下面是a.c文件：

```c
int main(void)
{
	volatile char a;

	a = 5;
	a = 7;
	
	return 0;
}
```

下面是b.c文件：

```c
int main(void)
{
	char a;

	a = 5;
	a = 7;
	
	return 0;
}
```

进行编译，分别得到它们对应的汇编文件：

```shell
$ make
arm-linux-gcc -S a.c -o a.s
arm-linux-gcc -S b.c -o b.s
```

对比a.s、b.s：

```shell
vimdiff a.s b.s
```
![](https://i.imgur.com/trqKr9t.png)

可以看到无任何差异，为什么呢？因为我们gcc的优化等级是默认等级，现在把优化等级调至-O3。再看对比
```shell
$ make
arm-linux-gcc -O3 -S a.c -o a.s
arm-linux-gcc -O3 -S b.c -o b.s
```
![](https://i.imgur.com/lRg80NB.png)

可以看到未加volatile修饰的文件b.c，在优化后，汇编对应的`a=5;a=7;`这两个语句直接优化没了。`a=1；a=0;`假设a是控制GPIO的语句，原来打算是让GPIO先拉高，再拉低，实现某种时序,结果优化一开，这两句直接废了。这让你在调试硬件的时候会感到莫名其妙。所以这种情况得像a.c那样用volatile来修饰。

这里是防止编译器优化掉相关语句，而不是优化变量的存取对象(memory or register)。

## 防止编译器优化变量的存取对象(memory or register)

**a.c**

```c
int main(void)
{
	int b;
	int c;
	volatile int* a = (int*)0x30000000;
	
	b = *a;
	c = *a;
	
	return c + b;
}
```
**b.c**

```c
int main(void)
{
	int b;
	int c;
	int* a = (int*)0x30000000;
	
	b = *a;
	c = *a;
	
	return c + b;
}
```

![](https://i.imgur.com/ct84fGz.png)

**a.s**

```asm
mov     r3, #805306368
ldr     r2, [r3]		@ b = *a;
ldr     r0, [r3]		@ c = *a;
add     r0, r0, r2		@ b + c;
bx      lr
```

**b.s**

```asm
mov     r3, #805306368
ldr     r0, [r3]		@ b = *a;
mov     r0, r0, asl #1  	@ b << 2; 也就是 b * 2;也就是 b + b;也就是 add r0, r0, r0(可能这句汇编不合法)
bx      lr
```
可以看到b.s被优化后，第一次取`*a`的值时，是从地址0x30000000取出来的（`ldr     r0, [r3]`），第二次就直接没取了，是直接使用了r0的值。这里的r0就是`*a`的缓存。我之前在这里的理解存在很大的错误：

>访问被volatile修饰的变量时，强制访问内存中的值，而不是**缓存**中的。

**以为这里的缓存指cache。**

毕竟从汇编指令优化着手怎么可能控制变量的一定从cache里面存取。

你不能假定一个共享（GPIO引脚和你的程序共享了GPIO数据寄存器）的变量只会被你修改。就好比你读一个文件的内容，过10秒后，你不能假定文件内容没变，必须要重新读取文件内容。

## 注意事项

**volatile关键词影响编译器编译的结果，用volatile声明的变量表示该变量随时可能发生变化，与该变量有关的运算，不要进行编译优化，以免出错**

一个例子：
```c
int main(void)
{
	int b;
	volatile int* a = (int*)0x30000000;
	
	b = (*a) * (*a);
	
	return b;
}
```

汇编后

```asm
mov     r3, #805306368
ldr     r2, [r3]	①
ldr     r0, [r3]	②
mul     r0, r2, r0
bx      lr
```

程序本意是要计算平方。如果这段代码在运行至①这行汇编时，被调度开了，过了一阵调度回来继续运行②行，此时完全有可能 R2 != R0。那么计算出来的结果R0必然不等于那个平方值。