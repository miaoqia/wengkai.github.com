# 100C8的startup.S  

ba5ag.kai at gmail.com 2014-04-24

100C8上电后的第一条代码在startup.S里，准确地说，我打算用

	STM32F10x_StdPeriph_Lib_V3.5.0/Libraries/CMSIS/CM3/DeviceSupport/ST/STM32F10x/startup/TrueSTUDIO/startup_stm32f10x_md_vl.s

为了方便，我在Makefile里是这样写的：

	AS = $(GNUARM)/bin/arm-none-eabi-as
	FLAG = -mcpu=cortex-m3 -mthumb -Wall

	startup.o: $(STM)/Libraries/CMSIS/CM3/DeviceSupport/ST/STM32F10x/startup/TrueSTUDIO/startup_stm32f10x_md_vl.s
	        $(AS) $(FLAGS) -c $< -o $@

就是说，不管具体的文件名叫什么，编译出来的.o都是startup.o。

我们来看看这个startup.s。

	.syntax unified
	.cpu cortex-m3
	.fpu softvfp
	.thumb

首先，Cortex-m3 为了兼容 thumb 指令和 thumb2 指令，使这两种指令可以使用统一的格式，引入了一种叫做"统一汇编语言 UAL "的语法机制，第一行说的就是这个。后面的应该不需要解释了，其义自明。

	.global	g_pfnVectors
	.global	Default_Handler

.global 使得连接程序（ld）能够识别后面的符号，所以这两行就是说明这两个符号是全局的，ld可以找过来的，一般也就意味着下面要出现这两个符号的定义了。

	.word	_sidata
	.word	_sdata
	.word	_edata
	.word	_sbss
	.word	_ebss

.word就是在这个地方放一个值。相当于在这里定义一个数据变量。这5个变量依次是：

* 1: .data段的变量的初始值所在的区域的开始地址；
* 2、3: .data段的开始和结束地址；
* 4、5: .bss段的开始和结束地址。

这些段是由链接器确定后，将相应的地址填入这些变量。

	.equ  BootRAM, 0xF108F85F

这句定义一个符号BootRAM的值，在后面会用到。

	.section	.text.Reset_Handler
	.weak	Reset_Handler
	.type	Reset_Handler, %function

接下去的段是代码的`Reset_Handle`，这就是重启（启动）时进入的第一段程序。.weak说明这个代码是弱链接的，如果另外地方有同样名字的非弱链接的函数，那么这个代码会被链接器丢掉。.type说明这个名字“Reset_Handle”是一个函数，可以被外部调用。

	Reset_Handler:
	/* Copy the data segment initializers from flash to SRAM */
	  movs	r1, #0
	  b	LoopCopyDataInit
	CopyDataInit:
		ldr	r3, =_sidata
		ldr	r3, [r3, r1]
		str	r3, [r0, r1]
		adds	r1, r1, #4
	LoopCopyDataInit:
		ldr	r0, =_sdata
		ldr	r3, =_edata
		adds	r2, r0, r1
		cmp	r2, r3
		bcc	CopyDataInit

上电启动的第一步，是把全局变量的初始值拷贝到SRAM中的全局变量中去。这段代码可能是从C语言编译过来的，所以用了一点优化小tricky，把循环的一部分代码前后颠倒了，所以看着怪怪的。基本上，它做的事情就是把`_sidata`开始的flash中的内容，逐字拷贝到`_sdata`开始的地址中，直到`_edata`为止（`_edata`是最后一个要拷贝的地址）。这里r1是索引，从0开始，每次加4；r3是源地址，r0是目的地址。但是在比较是否循环结束的地方，r3又用来表示结束地址，所以循环的每一轮都需要重新装载r3。

		ldr	r2, =_sbss
		b	LoopFillZerobss
	/* Zero fill the bss segment. */
	FillZerobss:
		movs	r3, #0
		str	r3, [r2], #4
	LoopFillZerobss:
		ldr	r3, = _ebss
		cmp	r2, r3
		bcc	FillZerobss

这段和前面的类似，一段循环把bss段全部填充为0，就相当于给没有初始值的全局变量填充了0值。

	/* Call the clock system intitialization function.*/
	  bl  SystemInit 
	/* Call static constructors */
	  bl __libc_init_array  
	/* Call the application's entry point.*/
		bl	main
		bx	lr

依次调用三个函数：`SystemInit`、`__libc_init_array`和`main`。也就是说，在调用应用程序的main之前，还有两个函数要先跑一下的。我们后面再展开来看那两个函数。还要注意的一个细节时，在调用SystemInit之前，堆栈指针还没有初始化，时钟频率（PLL）也还没有设置过。堆栈指针没有初始化就可以调用其他子程序，是因为ARM的BL指令是把返回地址保存在RL寄存器里而不是推堆栈的。另外中断也没有全局关闭，那是因为上电默认全局关闭。

最后的这句bx lr，会使得这段代码回到调用它的地方。但是这是一个中断服务程序，上电就到这里来了，所以实际上会重新回到这段代码开头的地方...所以main不能结束啊。

	.size	Reset_Handler, .-Reset_Handler

接下来：

	.section	.text.Default_Handler,"ax",%progbits
	Default_Handler:
	Infinite_Loop:
		b	Infinite_Loop
		.size	Default_Handler, .-Default_Handler

这是一个默认中断处理程序，所有应用程序不管的中断向量都填入了这个程序，它只是一个死循环。

	.section	.isr_vector,"a",%progbits
		.type	g_pfnVectors, %object
		.size	g_pfnVectors, .-g_pfnVectors
	g_pfnVectors:
		.word	_estack
		.word	Reset_Handler
		.word	NMI_Handler

这是Cortex-M3的最小的中断向量表，这个表从`g_pfnVectores`开始放，第一个是保留的，第二个就是前面的那个`Reset_Handler`了。接下去每个中断都已经定义了一个中断处理程序（少数直接填0）。最后一个是：

	.word BootRAM          /* @0x01CC. This is for boot in RAM mode for 
                         STM32F10x Medium Value Line Density devices. */

就是前面定义的那个符号BootRAM，它的值要填在这里。之所以不在这里直接写一个数值，是因为这样定义了符号之后，一容易让人理解这个是什么，二容易让人找到在哪里修改它。

接下来，是一排这样的代码：

	  .weak  NMI_Handler
	  .thumb_set 
	NMI_Handler,Default_Handler

这是用弱符号的方式，把这些中断向量都定义为那个死循环的默认处理程序。因为是弱符号，所以将来你自己写的中断处理程序就会直接替换它们。

把它编译之后，得到了startup.o。为了验证，再用`arm-none-eabi-objdump -d startup.o `把它dump 出来看看，和.s对照一下。