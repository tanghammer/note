# 目录
* [1.什么是SPI](#1什么是spi)

* [2.SPI接口](#2SPI接口)
	* [2.1.SPI模式：极性和时钟相位](#21spi模式极性和时钟相位)
	* [2.2.SPI三线总线和多IO配置](#22spi三线总线和多io配置)
	
* [3.SPI总线事务](#3spi总线事务)
	* [3.1.简单SPI写事务](#31简单spi写事务)
	* [3.2.简单SPI读事务](#32简单spi读事务)
	* [3.3.四线IO事务](#33四线io事务)

# 1.什么是SPI
**SPI**(Serial Peripheral Interface)[串行外围接口]**是一种接口总线，通常用于与闪存、传感器、实时时钟(RTCs)、模数转换器等进行通信。** 串行外围接口(SPI)总线是由摩托罗拉公司开发的，用于在主设备和从设备之间提供全双工同步串行通信。

[SPI教程白皮书](https://www.corelis.com/whitepapers/request-whitepaper-spi-tutorial/)

# 2.SPI接口
如图1所示，一个标准的SPI连接涉及到一个主机`master`使用串行时钟(SCK)、主输出从输入(MOSI)、主输出从输出(MISO)和从选择(SS)线连接到一个或几个从机`slave`。SCK、MOSI和MISO信号可以由从机`slave`共享，而每个从机`slave`都有一条惟一的SS线。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-11-1.jpg)

**Figure 1. 4-wire SPI bus configuration with multiple slaves**

## 2.1.SPI模式：极性和时钟相位
SPI接口没有定义数据交换协议，限制了开销并允许高速数据流。时钟极性(CPOL)和时钟相位(CPHA)可以指定为“0”或“1”，形成四种独特的模式，以提供主从通信的灵活性，如图2所示。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-21.jpg)

**Figure 2. SPI bus timing**

如果`CPOL`和`CPHA`都为' 0 '(定义为模式0)，则在时钟的前上升沿采样数据。目前,模式0是SPI总线通信最常见的模式。如果`CPOL`为' 1 '，`CPHA`为' 0 '(模式2)，则在时钟的前降边缘采样数据。同样，`CPOL` = ' 0 '和`CPHA` = ' 1 ' (Mode 1)在尾降边缘采样，`CPOL` = ' 1 '和`CPHA` = ' 1 ' (Mode 3)在尾升边缘采样。下面的表1总结了可用的模式
>CPOL：时钟极性，表示时钟线空闲时是高电平`1`还是低电平`0`。
>CPHA：时钟相位，表示是在时钟的前沿`0`还是尾沿`1`采样数据。

Mode|CPOL|CPHA
:-:|:-:|:-:
0|0|0
1|0|1
2|1|0
3|1|1

**Table 1. SPI mode definitions**

## 2.2.SPI三线总线和多IO配置
除了标准的4线配置外，SPI接口还扩展到包括各种IO标准，包括用于减少引脚数的3线和用于更高吞吐量的双或四I/O。

在3线模式下，MOSI和MISO线路组合成单个双向数据线，如图3所示。事务是半双工的，以允许双向通信。减少数据线的数量并以半双工模式运行也会降低最大可能的吞吐量; 许多3线设备具有低性能要求，而设计时考虑到低引脚数。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-31.jpg)

**Figure 3. 3-wire SPI configuration with one slave**

多I/O变体（如双I/O和四I/O）在标准外添加了额外的数据线，以提高吞吐量。利用多I/O模式的组件可以与并行器件的读取速度相媲美，同时仍然可以减少引脚数量。这种性能提升使得能够从闪存中随机访问和直接执行程序（XIP）。

例如，四路I/O设备在与高速设备通信时可提供四倍于标准4线SPI接口的性能。图4显示了单个四通道IO从站配置的示例。

![](https://www.corelis.com/wp-content/uploads/2017/05/2-41.jpg)

**Figure 4. Quad IO SPI configuration with one slave**

# 3.SPI总线事务
SPI协议没有定义数据流的结构; 数据的组成完全取决于组件设计者。但是，许多设备遵循相同的基本格式来发送和接收数据，从而允许来自不同供应商的部件之间的互操作性。

## 3.1.简单SPI写事务
大多数SPI闪存都有一个写状态寄存器命令，用于写入一个或两个字节的数据，如图5所示。要写入状态寄存器，SPI主机首先启用当前器件的从选择线。然后，主设备输出适当的指令，后跟两个数据字节，用于定义预期的状态寄存器内容。由于事务不需要返回任何数据，因此从设备将MISO线保持在高阻抗状态，并且主设备屏蔽任何输入数据。最后，从机选择信号被取消以结束事务。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-51.jpg)

**Figure 5. Write command using a single-byte instruction and two-byte data word**

## 3.2.简单SPI读事务
状态寄存器读取事务与写入事务类似，但现在利用从器件返回的数据，如图6所示。在发送读取状态寄存器指令后，从器件开始以MISO线路传输数据，数率为每八个时钟周期一个字节。主机接收比特流并通过取消SS信号来完成事务。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-61.jpg)

**Figure 6. Read command using a single-byte instruction and two-byte data word**

## 3.3.四线IO事务
由于其性能的提高，四线IO在闪存中越来越受欢迎。四线IO没有使用单输出和单输入接口，而是使用4条独立的半双工数据线来传输和接收数据，其性能是标准四线SPI的四倍。

图7显示了Spansion S25FL016K串行NorFLASH器件的读取示例命令。要从器件读取，主器件首先在第一个IO线上发送快速读取命令（EBh），而其他所有命令都处于三态。接下来，主机发送地址; 由于接口现在有4条双向数据线，因此它可以利用它们在8个时钟周期内发送一个完整的24位地址和8个模式位。然后，该地址跟随2个虚拟字节（4个时钟周期），以允许器件有额外的时间来设置初始地址。

![](https://www.corelis.com/wp-content/uploads/2017/05/1-71.jpg)

**Figure 7. Quad mode fast read sequence for Spansion S25FL016K or equivalent**

在主机发送地址周期和虚拟字节之后，组件开始发送数据字节; 每个时钟周期由分布在4个IO线上的数据半字节组成，每个数据字节总共有两个时钟周期。将此与我们简单读取事务所需的16个时钟周期进行比较，很容易看出为什么四模式在高速闪存应用中越来越受欢迎！