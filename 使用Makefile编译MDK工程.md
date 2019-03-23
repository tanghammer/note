## 使用Makefile编译MDK工程
## 1.安装Windows上的gnu环境msys
使用makefile进行编译，那么就必须需要make这个工具，并且还要有个shell环境，还有一些其他gnu工具例如awk,sed，grep，diff,patch等等。msys(Minimal GNU（POSIX）system on Windows)是一个轻量级的gnu环境。[https://www.msys2.org/](https://www.msys2.org/ "msys")在此处可以下载安装。

### 2.准备必须的文件
在linux上交叉编译裸机程序需要：交叉编译工具链、源码文件、链接文件、Makefile。

**交叉编译工具链**，在windows下交叉编译工具链使用MDK自动的，一般在C:\Keil_v5\ARM\ARMCC\bin类似的这个目录下：
![](https://i.imgur.com/SFqooJm.png)

**源码文件**，为了将各个功能区分开，将一个文件拆分为三个文件：vector.s、handler.s、test.s,他们分别包含：中断向量表，中断服务函数、用户程序。
vector.s:
```asm
;=========================================================================
; 中断向量表
		AREA    VECTOR_TABLE, DATA, READONLY

            	EXPORT  __Vectors
            	EXPORT  __Vectors_End
            	EXPORT  __Vectors_Size

            	IMPORT Reset_Handler

__Vectors       DCD     0x20000000                ; 栈顶
            	DCD     Reset_Handler             ; 复位异常向量

__Vectors_End

__Vectors_Size  EQU     __Vectors_End - __Vectors

		END

```
handler.s:

```asm
;=========================================================================
;代码段，异常处理函数
	AREA    HANDLER, CODE, READONLY
	ENTRY
	
	IMPORT Start
	
Reset_Handler   PROC
	EXPORT  Reset_Handler             [WEAK]
	LDR     R0, =Start
	BX      R0 
	ENDP
	
	END
```

test.s:
```asm
;=========================================================================
;代码段 
	AREA    TEST, CODE, READONLY
	EXPORT Start
Start
	MOV R0, #0xFF
	B Start
	NOP
	
	END
```

**分散加载描述文件**，分散加载文件用于描述CPU启动后，如何从非易失性介质中加载程序和数据，在加载完后又如何运行。其实就和linux下的连接脚本一个作用。

    LR_IROM1 0x00000000 0x00000100
    {
      ER_IROM1 0x00000000 0x00000100  
      {
       vector.o (VECTOR_TABLE, +First)
       handler.o
       test.o
      }
    }

**Makefile**,指导如何构建工程

```makefile
#################################################
# 指定交叉编译工具链
#################################################
KEIL_PATH = c:/Keil_v5/ARM/ARMCC/bin

ARMCC = $(KEIL_PATH)/armcc
ARMASM = $(KEIL_PATH)/armasm
ARMAR = $(KEIL_PATH)/armar
ARMLINK = $(KEIL_PATH)/armlink
FROMELF = $(KEIL_PATH)/fromelf

#################################################
# 编译选项
#################################################
CFLAGS := -c --cpu Cortex-M3 -g --li -O0 --apcs=interwork --split_sections
ASMFLAGS := --cpu Cortex-M3 -g --li --apcs=interwork
LINKFLAGS := --cpu Cortex-M3 --strict
MAP := --autoat --summary_stderr --info summarysizes --map --xref --callgraph --symbols 
INFO := --info sizes --info totals --info unused --info veneers


TARGET = cm3_assembly

OBJS := $(patsubst %.s,%.o,$(wildcard *.s))

%.o:%.c
	$(ARMCC) $(CFLAGS) $< -o $@
	
%.o:%.s
	$(ARMASM) $(ASMFLAGS) $< -o $@	

all:$(OBJS)
	$(ARMLINK) $(LINKFLAGS)  --scatter $(TARGET).sct $(MAP) $(INFO) --list $(TARGET).map $^ -o $(TARGET).axf

bin:$(TARGET).axf
	$(FROMELF) --bin $^ -o $(TARGET).bin
	
hex:$(TARGET).axf
	$(FROMELF) --i32 $^ -o $(TARGET).hex

.PHONY : clean

clean:
	rm -rf $(OBJS) *.map *.htm *.axf *.hex *.bin
```

### 3.构建工程
打开msys，进入工程目录，执行make，生成.axf文件。还可以通过make bin；make hex来生成bin文件和hex文件。
![](https://i.imgur.com/c00xqyY.png)


### 4.分析源码
首先，cortex-m3上电相当于是进行了一个复位，它复位会做以下几步：

1. 从内部rom上0地址开始读取第一4字节（复位后的栈地址）加载到MSP。
2. 从内部rom上0地址开始读取第二个4字节（复位中断复位程序入口地址）加载至PC(R15)。

对应到源文件vector.s:
```asm
DCD     0x20000000                ; 栈顶
DCD     Reset_Handler             ; 复位异常向量
```
指令DCD将一个4字节的数值放在此处。
PC取到Reset_Handler这个地址后，进入复位异常处理函数里面
```asm
Reset_Handler   PROC
	EXPORT  Reset_Handler             [WEAK]
	LDR     R0, =Start			;伪指令，加载Start标号的地址到R0
	BX      R0 				    ;无条件跳转至Start
	ENDP
```
test.s：
```asm
Start
	MOV R0, #0xFF
	B Start			;死循环
	NOP
```
整个程序做的事很简单，从复位异常处理函数里面跳转到Start函数这个死循环里面。

### 5.剖析整个构建过程
首先如果源文件是C文件，那么构建的过程大致是：预处理--->编译--->汇编--->连接
![](https://i.imgur.com/NfNrT98.png)

我这里是直接从汇编文件开始处理的，a.c、a.i、a.s这三个文件都是可阅读的，但是之后的a.o、a.axf都是不可阅读的，使用file命令可以看到：
![](https://i.imgur.com/ehUMybJ.png)

.o文件是

	ELF 32-bit LSB relocatable
.axf文件是

	ELF 32-bit LSB executable

它们都属于ELF文件，何为ELF文件?
ELF, Executable and Linking Format, 是一种用于可执行文件、目标文件、共享库和核心转储的标准文件格式。  ELF格式是是UNIX系统实验室作为ABI（Application Binary Interface）而开发和发布的。

ELF有四种不同的类型:  
1. 可重定位文件(Relocatable): 编译器和汇编器产生的.o文件，需要被Linker进一步处理  
2. 可执行文件(Executable): Have all relocation done and all symbol resolved except perhaps shared library symbols that must be resolved at run time  
3. 共享对象文件(Shared Object): 即动态库文件(.so)  
4. 核心转储文件(Core File): 

这里只涉及到了前两种，并且是用可重定位文件经过armld处理后生成了可执行文件。ELF文件具体的格式是怎么样的?armld又是如何处理的？首先看看ELF文件的格式
![](https://upload.wikimedia.org/wikipedia/commons/thumb/7/77/Elf-layout--en.svg/800px-Elf-layout--en.svg.png)

最开头它有一个ELF header，描述了整个文件的组织，占用 52-bytes。用一个结构体描述就是：
```c
#define EI_NIDENT (16)
typedef struct
{
	unsigned char e_ident[EI_NIDENT];   /* Magic number and other info */
	Elf32_Half    e_type;               /* Object file type */
	Elf32_Half    e_machine;            /* Architecture */
	Elf32_Word    e_version;            /* Object file version */
	Elf32_Addr    e_entry;              /* Entry point virtual address */
	Elf32_Off     e_phoff;              /* Program header table file offset */
	Elf32_Off     e_shoff;              /* Section header table file offset */
	Elf32_Word    e_flags;              /* Processor-specific flags */
	Elf32_Half    e_ehsize;             /* ELF header size in bytes */
	Elf32_Half    e_phentsize;          /* Program header table entry size */
	Elf32_Half    e_phnum;              /* Program header table entry count */
	Elf32_Half    e_shentsize;          /* Section header table entry size */
	Elf32_Half    e_shnum;              /* Section header table entry count */
	Elf32_Half    e_shstrndx;           /* Section header string table index */
} Elf32_Ehdr;
```