参考自[restrict](https://en.cppreference.com/w/c/language/restrict)

## restrict解释

`restrict`关键字出现于`C99`标准，wiki上的解释[restrict from wiki](https://en.wikipedia.org/wiki/Restrict)。

>In the C programming language, as of the C99 standard, restrict is a keyword that can be used in pointer declarations. The restrict keyword is a declaration of intent given by the programmer to the compiler. It says that for the lifetime of the pointer, only the pointer itself or a value directly derived from it (such as pointer + 1) will be used to access the object to which it points. This limits the effects of pointer aliasing, aiding optimizations. If the declaration of intent is not followed and the object is accessed by an independent pointer, this will result in undefined behavior. The use of the restrict keyword in C, in principle, allows non-obtuse C to achieve the same performance as the same program written in Fortran.[1]

>在C编程语言中，从`C99`标准开始，`restrict`是一个可以在指针声明中使用的关键字。`restrict`关键字是由程序员给编译器的一种意向声明。它表示在指针的生命周期内，只有指针本身或直接从指针派生的值(如指针+ 1)将用于访问指针指向的对象。这限制了指针别名的效果，有助于优化。如果没遵循指针声明的意图，使用不受约束的指针对其指向对象存取，将导致结果是未定义的。

## 实例

上面这堆翻译好像我只看懂了：**restrict关键字用于指针的声明**。那到底什么意思呢？在进一步了解restrict之前，需要了解什么是[pointer aliasing](https://en.wikipedia.org/wiki/Pointer_aliasing)。
pointer aliasing是指多个指针指向同一块内存区域，例如：

```c
int a = 10;
int* ptr1 = &a;
int* ptr2 = &a;
```

则ptr1、ptr2互称对方为自己的pointer aliasing。考虑下面这个函数：

```c
int foo(int* a, int* b)
{
	*a = 2；
	*b = 3;
	return *a + *b;
}
```
一般情况下,编译器翻译出来的汇编代码会类似（以ARM汇编为例）：
```asm
01    str     r0, [fp, #-8]			@传递参数指针a
02    str     r1, [fp, #-12]			@传递参数指针b
03    ldr     r3, [fp, #-8]			@得到指针a
04    mov     r2, #2
05    str     r2, [r3]				@*a = 2;
06    ldr     r3, [fp, #-12]			@得到指针b
07    mov     r2, #3
08    str     r2, [r3]				@*b = 3;
09    ldr     r3, [fp, #-8]			@得到指针a		
10    ldr     r2, [r3]				@r2 = *a,     重读*a,因为如果a==b,则08行会覆盖05行的赋值！！！！！
11    ldr     r3, [fp, #-12]			@得到指针b
12    ldr     r3, [r3]				@r3 = *b;
13    add     r0, r2, r3			@*a + *b;
```

如果我们可以提前告诉编译器`a != b`，则编译器可以优化出更简洁的汇编(假设a、b指针都是非volatile的)：

```asm
01    str     r0, [fp, #-8]			@传递参数指针a
02    str     r1, [fp, #-12]			@传递参数指针b
03    ldr     r3, [fp, #-8]			@得到指针a
04    mov     r2, #2
05    str     r2, [r3]				@*a = 2;
06    ldr     r3, [fp, #-12]			@得到指针b
07    mov     r1, #3
08    str     r1, [r3]				@*b = 3;
09    add     r0, r1, r2			@*a + *b;
```
比上面少了4条指令。为什么呢？因为上面那段汇编中得考虑一种情况，如果指针`a == b`，那么第`08`行的指针赋值会覆盖`05`行的指针赋值，为了保险起见，`*a + *b`就得重新从内存中取值，才能得到正确的结果。如果我们显式的告诉编译器`a != b`,则编译器就会使用寄存器里面的缓存数据来优化代码。如何显式告诉编译器呢？就是用`restrict`关键字来修饰指针。

## 注意事项
但是有点需要注意，**不是说用`restrict`修饰指针后就保证了指针不会出现pointer aliasing，这种保证由程序员自己编码时确保**。就是说程序员自己保证对同一内存区域的引用只有一个指针指向，然后通过`restrict`修饰此指针变量，在开启编译优化后，编译器将会特别留意被`restrict`修饰的指针，进行优化。

`restrict`关键字只能修饰[object](https://en.cppreference.com/w/c/language/object)类型的指针（例如`int restrict *p`、`float (* restrict foo)(void)`都是错误的用法）。`restrict`仅用于修饰左值(lvalue)。例如使用它强制转换指针或者修饰函数返回值都是错误的用法。

解释下2个名词：
**object：**
在C语言中，对象是执行环境中的数据存储区域，其内容可以表示值(当解释为具有特定类型时,值是对象内容的含义)。每个对象有如下特征：

- 尺寸(能用sizeof计算）
- 有对齐需求
- 存储期（automatic, static, allocated, thread-local）
- 有效类型
- 值（可能是不确定的）
- 可选地，表示此对象的标识符（就是变量名）

`int restrict *p` 修饰的不是指针。
`float (* restrict foo)(void)`，foo指针指向的区域明显不具备存储数据的能力，*foo = 12是非法的，因为如果它是一个函数指针，指向的内存区域是只读的。

**左值（lvalue):**
只能出现在C语言赋值表达式左边的。
显然`ret = fun()、a = (int)b`就说明了函数返回值，强转都是属于右值。
