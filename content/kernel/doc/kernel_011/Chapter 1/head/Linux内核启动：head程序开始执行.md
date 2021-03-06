# Linux内核启动：head程序执行过程

<h2 id = 'm'> 目录 </h2>

[教学视频](#t)

[1. 整体过程描述](#1)

[2. HEAD 程序设置栈寄存器](#2)

[3. HEAD设置IDT和GDT](#3)

[5. 重建GDT表和调整段寄存器](#5)

[6. 检测A20](#6)

[7. 检测并开启协处理器](#7)

[8. main函数入栈](#8)

[9. 设定内核页表](#9)

[10. 返回执行main函数](#10)

[直达底部](#e)

<h2 id = 't'> 教学视频 </h2>

[Linux内核启动：head程序开始执行(一)](http://toutiao.com/item/6656049952554222094/ "Linux内核启动：head程序开始执行(一)")

[Linux内核启动：head程序开始执行(二)](http://toutiao.com/item/6656056858475758084/ "Linux内核启动：head程序开始执行(二)")

[Linux内核启动：head程序开始执行(三)--分析setup_idt代码实现](http://toutiao.com/item/6656222683673395716/ "Linux内核启动：head程序开始执行(三)--分析setup_idt代码实现")

<h2 id = '1'> 1. 整体过程描述 </h2>
  
  在执行main函数之前，先要执行三个由汇编代码生成的程序，即bootsect、setup和head。之后，才 执行由main函数开始的用C语言编写的操作系统内核程序。

- 第一步，加载bootsect到0x07C00，然后复制到0x90000； 
- 第二步，加载setup到0x90200。 值得注意的是，这两段程序是分别加载、分别执行的。 
- head 程序与它们的加载方式有所不同。大致的过程是


	1. 先将head.s汇编成目标代码，将用C语言编写 的内核程序编译成目标代码，然后链接成system模块。也就是说，system模块里面既有内核程序，又有 head程序。两者是紧挨着的。要点是，head程序在前，内核程序在后，所以head程序名字为“head”。 head程序在内存中占有25KB+184B的空间。前面讲解过，system模块加载到内存后，setup将system 模块复制到0x00000位置，由于head程序在system的前面，所以实际上，head程序就在0x00000这个位置。
	2. head 程序除了做一些调用 main 的准备工作之外，还用程序自身的代码在程序自身所在的内存空间 创建了内核分页机制，即在 0x000000 的位置创建了页目录表、页表、缓冲区、GDT、IDT，并将head 程序已经执行过的代码所占内存空间覆盖。这意味着head程序自己将自己废弃，main函数即将开始执行。 

![](https://i.imgur.com/B2qG3py.png)

- 以上就是head程序执行过程的整体策略。

这里我们先来关注一下页表的标号： **\_pg_dir**

![](https://i.imgur.com/eOgdhuX.png)

  标号\_pg_dir标识内核分页机制完成后的内核页表起始位置，也就是物理内存的起始位置 0x000000。head程序马上就要在此处建立页目录表，为分页机制做准备。 

![](https://i.imgur.com/hQcd8zM.png)


[返回目录](#m)

<h2 id = '2'> 2. HEAD 程序设置栈寄存器 </h2>

  现在head程序正式开始执行，一切都是为适应保护模式做准备。jmpi 0,8这一句已经将CS的段选择子与GDT第二个表项关联，CS段指向了0x000000。从现在开始，要将DS、ES、FS 和 GS 等其他寄存器 从实模式转变到保护模式，与GDT第三个表项--内核代码段描述符相关联

![](https://i.imgur.com/0fs2m4i.png)

  执行完毕后，DS、ES、FS 和 GS 中的值都成为0x10。0x10 也应看成二进制的00010000，最后两位（00）表示内核特权级，从后数第3位（0）表示选择GDT，第 4、5 两位（10）是 GDT 的 3 项（index = 2）。 也就是说，4个段寄存器用的是同一个全局描述符，它们的段基址、段限长、特权 级都是相同的。特别要注意的是，影响段限长的关键字段的值是 0x7FF，此时段限长就是8MB。

![](https://i.imgur.com/HJ5ankU.png)

  DS,ES,FS,GS 都要参考GDT中的内容。代码中的 movl$ 0x10，%eax 中的 0x10 是GDT中的偏移 值（ 用二进制表示就是10000），即要参考 GDT 中第 2 项的信息来设置这些段寄存器，这一项就是 内核数据段描述符。

  SS现在也要转变为栈段选择符，栈顶指针也成为 32 位的esp。

![](https://i.imgur.com/M8YZKg0.png)

  在 kernel/sched.c中，stack_start={＆user_ stack[PAGE_ SIZE＞＞2]，0x10} 这行代码 将栈顶指针指向user_stack数据结构的最末位置。这个数据结构是在 kernel/sched.c中定义 的，其起始位置为 0x1E25C。
![](https://i.imgur.com/ZARVVbe.png)

> 设置段寄存器指令（Load Segment Instruction）： 该组指令的功能是把内存单元的一个“低字”传送给指令中指定的16位寄存器，把随后的一个“高字”传给相应的段寄存器（DS、ES、FS、GS 和 SS），指令格式如下
> 
	LDS/LES/LFS/LGS/LSS Mem Reg
>
> 指令LDS（Load Data Segment Register）和LES（Load Extra Segment Register）在 8086 CPU 中就存在，而LFS和LGS、LSS（Load Stack Segment Register）是 80386 及其以后 CPU 中 才有的指令。若 Reg 是16位寄存器，则Mem必须是32位指针；若Reg是32位寄存器，则 Mem 必须是48 位指针，其低32位给指令中指定的寄存器，高16位给指令中的段寄存器。

0x10 将SS设置为与前面4个段选择符的值相同。这样SS与前面的4个段选择符相同，段基址都是指向0x000000，段限长都是8MB，特权级都是内核特权级，后面的压栈动作就要在这里进行。

![](https://i.imgur.com/nYGpR1i.png)

[返回目录](#m)

<h2 id = '3'> 3. HEAD设置IDT和GDT </h2>

  head程序接下来对IDT进行设置

![](https://i.imgur.com/GEXZ2cZ.png)

![](https://i.imgur.com/pR5cyle.png)

![](https://i.imgur.com/sXgy9Gn.png)

  setup idt 代码流程说明
![](https://i.imgur.com/8XY87xV.png)

一 拼接中断描述符的内容

1. lea ignore_int,%edx 		将ignore\_int的段内偏移量 0x0000 5428 存储到 edx中，dx为低16位，即5428
2. movl $0x00080000,%eax	将0x00080000存储到eax内，ax为低16位，即 0000
3. movw %dx,%ax				将edx低16位5428，移动到ax，此时 eax的内容为 0x0008 5428
4. movw $0x8E00,%dx			将0x8E00, 放到dx，此时 edx 内容为 0000 8E00

中断描述符拼接完的效果如下：

![](https://i.imgur.com/zTMwbNq.png)


二 反复将拼接好的中断描述符存储到_idt,存储256次

5.  lea _idt,%edi  			取得_idt基地址为目的地址
6.  mov $256,%ecx			循环256次，初始化256个中断描述符
7.  rp_sidt：				for 循环开始标号
8.  movl %eax,(%edi)		把 eax 内容存储到描述符的低4个字节，即 0x 0008 5428
9.  movl %edx,4(%edi)		把 edx 内容存储到描述符的高4个字节，即 0x 0000 8E00，此时已经填写完一个中断描述符
10. dec %ecx				填写完一个描述符，循环变量减1
11. jne rp_sidt				循环变量不为0，继续回到for循环开始处，继续初始化填写描述符
12. lidt idt_descr			for循环完成，将_idt的基地址存放到idtr, 限长256项

> LEA是微机8086/8088系列的一条指令，取自英语Load effect address——取有效地址，也就是取偏移地址。在微机8086/8088中有20位物理地址，由16位段基址向左偏移4位再与偏移地址之和得到。　
　取偏移地址指令,指令格式如下：

	LEA reg16,mem 

> LEA指令将存储器操作数mem的4位16进制偏移地址送到指定的寄存器。这里，源操作数必须是存储器操作数，目标操作数必须是16位通用寄存器。因该寄存器常用来作为地址指针，故在此最好选用四个间址寄存器BX,BP,SI,DI之一。

> movl: mov long ： 字长传送 ： 32位
> movw: mov word：字传送 ：16位
> movb: mov byte：字节传送 ：8位

中断描述符结构如下

![](https://i.imgur.com/Wm3mWkp.png)

  中断描述符为64位，包含了其对应中断服务程序的段内偏移地址（OFFSET）、所在段选择符（SELECTOR）、描述符特权级（DPL）、段存在标志（P）、段描述符类型（TYPE）等信息，供CPU在程序中需要进行中断服务时找到相应的中断服务程序。 

> 其中，第0～15位和第48～63 位组合成32位的中断服务程序的段内偏移地址（OFFSET)； 第16～31位 为段选择符（SELECTOR），定位中断服务程序所在段；第47位为段存在标志（P），用于标识此段是否 存在于内存中，为虚拟存储提供支持； 第45～46位为特权级标志（DPL），特权级范围为 0～3； 第 40～43位为段描述符类型标志（TPYE），中断描述符对应的类型标志为0111（0xE），即将此段描述符 标记为“386中断门”。

  这是重建保护模式下中断服务体系的开始。程序先让所有的中断描述符默认指向ignore_int这个位置作为默认的中断处理函数（将来 main 函数里面还要让中断描述符对应具体的中断服务程序），之后还要对IDT寄存器的值进行设置。
  
  也就是说，<font color = red> **idt表的基地址为 \_idt, 默认处理函数地址为 ignore_int 地址** </font>

![](https://i.imgur.com/KconM72.png)

  构造IDT，使中断机制的整体架构先搭建起来（实际的中断服务程序挂接则在main函数中完成），并使 所有中断服务程序指向同一段只显示一行提示信息就返回的服务程序。IDT有256个表项，实际只使用了几十个，对于误用未使用的中断描述符，这样的提示信息可以提醒开发人员注意错误。

[返回目录](#m)


<h2 id = '5'> 5. 重建GDT表和调整段寄存器 </h2>

  现在，head 程序要废除已有的GDT，并在内核中的新位置重新创建GDT，其中第2项和第3项分别为内核代码段描述符和内核数据段描述符，其段限长均被设置为16MB，并设置GDTR的值。

![](https://i.imgur.com/2cRYik6.png)

 代码实现如下
![](https://i.imgur.com/4hCS5fI.png)

![](https://i.imgur.com/vu7vgWK.png)

![](https://i.imgur.com/WhnoOvZ.png)

  原来GDT所在的位置是设计代码时在setup.s里面设置的数据，将来这个setup模块所在的内存位置会在设计缓冲区时被覆盖。如果不改变位置，将来GDT的内容肯定会被缓冲区覆盖掉，从而影响系统的运行。这样一来，将来整个内存中唯一安全的地方就是现在head.s所在的位置了。

  GDT的位置和内容发生了变化，特别要注意最后的三位是FFF，说明段限长不是原来的8MB，而是现在 的16MB。如果后面的代码第一次使用这几个段选择符，就是访问8 MB以后的地址空间，将会产生段 限长超限报警。为了防止这类可能发生的情况，这里再次对一些段选择符进行重新设置，包括DS、ES、FS、GS及SS。

  内存分布如下：

![](https://i.imgur.com/9iI2BMF.png)

  各段寄存器调整代码如下：

![](https://i.imgur.com/HAqn9PK.png)

  现在，栈顶指针esp指向user_stack数据结构的外边缘，也就是内核栈的栈底(user_start的最后一个地址,user_stack[1024]的末尾)。这样，当后面的程序需要压栈时，就可以最大限度地使用栈空间。栈顶的增长方向是从高地址向低地址的

![](https://i.imgur.com/yrYCZMT.png)

[返回目录](#m)


<h2 id = '6'> 6. 检测A20 </h2>

  因为A20地址线是否打开影响保护模式是否有效，所以，要检验A20地址线是否确实打开了。

![](https://i.imgur.com/nHxcuHj.png)

  实现代码如下

![](https://i.imgur.com/Q0379fz.png)

  A20如果没打开，则计算机处于20位的寻址模式，超过0xFFFFF寻址必然“回滚”。一个特例是0x100000会回滚到0x000000，也就是说，地址 0x100000 处存储的值必然和地址0x000000处存储 的值完全相同。通过在内存 0x000000 位置写入一个数据，然后比较此处和1 MB（0x100000，注意，已超过实模式寻址范围）处数据是否一致，就可以检验A20地址线是否已打开。

[返回目录](#m)

<h2 id = '7'> 7. 检测并开启协处理器 </h2>

  确定 A20 地址线已经打开之后， head程序如果检测到数学协处理器存在，则将其设置为保护模式工作状态

![](https://i.imgur.com/Rnw0nvC.png)

  代码实现如下：

![](https://i.imgur.com/MhTMkdr.png)

![](https://i.imgur.com/2FqhXSg.png)

[返回目录](#m)

<h2 id = '8'> 8. main函数入栈 </h2>

  head程序将为调用main函数做最后的准备。这是head程序执行的最后阶段，也是main函数执行前的最后阶段。

![](https://i.imgur.com/UE3Ksmp.png)

  head程序将 L6 标号和 main 函数入口地址压栈，栈顶为main函数地址，目的是使 head 程序执行完后通过 ret 指令就可以直接执行 main 函数。

![](https://i.imgur.com/JLerxaL.png)

  main函数在正常情况下是不应该退出的。如果main函数异常退出，就会返回这里的标号L6处继续执行

  代码实现如下：

![](https://i.imgur.com/XOFW5td.png)

![](https://i.imgur.com/YPUvTAz.png)

[返回目录](#m)

<h2 id = '9'> 9. 设定内核页表 </h2>

  压栈main完成后，head程序将跳转至setup_ paging： 去执行，开始创建分页机制。先要将页目录 表和4个页表放在物理内存的起始位置，从内存起始位置开始的5页空间内容全部清零（每页4 KB），为初始化页目录和页表做准备。注意，这个动作起到了用1个页目录表和4个页表覆盖 head 程序自身所占内存空间的作用。

![](https://i.imgur.com/RKWupwT.png)

  head程序将页目录表和4个页表所占物理内存空间清零后，设置页目录表的前4项，使之分别指向4个 页表

![](https://i.imgur.com/bOJUZCS.png)

  head 程序设置完页目录表后，Linux 0.11 在保护模式下支持的最大寻址地址为0xFFFFFF（16MB），此处将第4个页表（由pg3指向的位置）的最后一个页表项（pg3+4902 指向的位置）指向 寻址范围的最后一个页面，即0xFFF000开始的4KB字节大小的内存空间。

![](https://i.imgur.com/JKegWAP.png)

  然后开始从高地址向低地址方向填写4个页表，依次指向内存从高地址向低地址方向的各个页面。

  继续设置页表。将第4个页表（由 pg3 指向的位置）的倒数第二个页表项（pg3-4+ 4902指向的位置）指向倒数第二个页面，即0xFFF000-0x1000（0x1000即4KB，一个页面的大小）开始的4KB字节内存空间。

![](https://i.imgur.com/RloCYn6.png)

  最终，从高地址向低地址方向完成4个页表的填写，页表中的每一个页表项分别指向内存从高地址向低地址方向的各个页面

![](https://i.imgur.com/LVplWT3.png)

  这4个页表都是内核专属的页表，将来每个用户进程都会有它们专属的页表。

代码实现如下：

![](https://i.imgur.com/PLy428x.png)

![](https://i.imgur.com/vE35bEH.png)

> STOSL指令相当于将EAX中的值保存到ES:EDI指向的地址中，若设置了EFLAGS中的方向位置位(即在STOSL指令前使用STD指令)则EDI自减4，否则(使用CLD指令)EDI自增4；

> 位置代码设置伪操作 .org <br/>
> 语法格式： 
> 
	.org  offset {,exp}
> 其中
> offset 是一个数值表达式，表示地址偏移量<br/>
> expr 用来指定填充的数据 <br/>   

内存分布示意图

![](https://i.imgur.com/qUW1Y3S.png)

  这些工作完成后，只有184字节的剩余代码。 
  
  head 程序已将页表设置完毕了，但分页机制的建立还没有完成，还需要设置页目录表基址寄存器CR3，使之指向页目录表，再将CR0寄存器设置的最高位（31 位）置为1

> PG（Paging）标志： CR0寄存器的第31位，分页机制控制位。当CPU的控制寄存器CR0第0位PE（保护 模式）置为1时，可设置PG位为开启。当开启后，地址映射模式采取分页机制。当CPU的控制寄存器CR0第 0位PE（保护模式）置为0时，设置PG位将引起CPU发生异常。 
> 
> CR3寄存器：3号32位控制寄存器，其高20位存放页目录表的基地址。当CR0中的PG标志置位时，CPU使用CR3指向的页目录表和页表进行虚拟地址到物理地址的映射。

  代码实现如下：

![](https://i.imgur.com/ZZQPTDu.png)

  前两行代码的动作是将CR3指向页目录表，意味着操作系统认定0x0000这个位置就是页目录表的起始位置；后3行代码的动作是启动分页机制开关PG标志置位，以启用分页寻址模式。 

  到这里为止，内核的分页机制构建完毕。

  在内存的起始位置建立内核分页机制，最后就认定页目录表在内存的起始位置。 这个位置是内核通过分页机制能够实现线性地址等于物理地址的唯一起始位置。

[返回目录](#m)

<h2 id = '10'> 10. 返回执行main函数 </h2>

  head程序执行最后一步：ret。 这要通过跳入main函数程序执行。

![](https://i.imgur.com/yxXhH9l.png)


  原理图：

![](https://i.imgur.com/4RYQx6J.png)

  将压入的main函数的执行入口地址弹出给CS：EIP，这句话等价于CPU开始执行main函数程序。

![](https://i.imgur.com/UX87CwL.png)


[返回目录](#m)

<p id = 'e'> </p>