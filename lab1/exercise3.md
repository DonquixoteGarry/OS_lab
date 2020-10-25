

# EXERCISE3

> 问题1:为何开启 A20,以及如何开启 A20

>> 1.什么是A20

``A20``,即`A20 Gate`,即控制第21根地址总线的开关.当A20打开时,对第21地址线的读写无效,自动返回0.

在`8086/8088`中,使用16位地址模式与20根地址总线,访存时使用16位段基址(左移4位)加16位偏移量的方式,得到20位的地址,理论最高地址为(0xFFFF<<4)+0xFFFF=0x10FFEF但实际上,8086/8088中只有20条地址线,因此实际上最高可访问地址为0x100000,故地址区间0x100000到0x10FFEF之间的内存区域无法访问.

因此,当给出该区间内的地址时,将自动进行取模运算,将地址区间0x100000到0x10FFEF映射到地址区间0x000000到0x00FFEF之间,即地址的回卷(wrap-around).

而在`80286`中,使用16位地址模式与24根地址总线.因此,不需要地址的回卷,`80286`也能正常表示地址区间0x100000到0x10FFEF之间的内存地址,但这与使用回卷的`8086/8088`不兼容.为实现`8086/8088`与`80286`之间的兼容,需要控制与地址区间0x100000到0x10FFEF之间的内存地址访问方式相关的第21条地址总线.而`80286`实现这个功能的开关,便是`A20 Gate`.

在80286之后的使用位数更多的地址总线的处理器,如使用32位地址线的`i386`,也普遍使用`A20`控制第21位地址线的读写.

> > 2.实模式和保护模式

实模式,是一种内存寻址方式,实际上就是无视`A20`的开闭,在`8086/8088`和`80286`中均使用取模运算处理地址区间0x100000到0x10FFEF.当然,由于`8086/8088`中没有`A20`,因此其也只有实模式.因此,`80286`在实模式下的行为和`8086/8088`相同,实现了向前兼容的特性.

而保护模式,则是`80286`在`A20`打开的情况下,正常的读写第21位地址线,不采用取模方式处理地址区间0x100000到0x10FFEF.

> > 3.为什么需要开启A20,进入保护模式

`80286`及其之后的满足向前兼容的处理器中,如果不能对第21位地址总线的数据正常处理(恒取0),则不能完全利用所有的内存空间,仅能访问段基址为偶数的地址.为利用所有的内存空间,需要进入保护模式,开启`A20`,实现对第21位地址总线的数据的正常处理.

> > 4.流程代码解析

```assembly
seta20.1:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
    
seta20.2:
    inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60
    outb %al, $0x60                                 # 0xdf = 11011111, means set P2's `A20` bit(the 1 bit) to 1
```

~~因为IBM程序员鬼迷心整了个PS/2电脑和PS/2接口~~

因为历史原因,键盘控制器端口加了一个启动后值自动置0的引脚，在第21根地址总线前加一个与门.并将键盘控制器引出的该引脚作为此与门的输入端,以使用键盘控制器的输入控制`A20`地址线.

但为防止在键盘控制器端口忙时向其发送指令(可能引起错误),需要确认其状态为不忙.从端口`0x64`(键盘控制器状态端口)中读到一个8bit的状态值放入AL寄存器,使用`test`指令和参数0x2访问这个状态值的第2位(即`AL`中的1号bit位),而这个bit位表示的正是`0x64`端口输入缓冲区的状态.因此,当`test`指令判断`0x64`端口输入缓冲区忙时,自动跳回读`0x64`端口的代码再次执行,直到`0x64`端口出现空闲.

为了能够对输出端口进行写操作,当先后两次读到`0x64`端口空闲时,先向`0x64`端口发送指令0xd1,再向`0x60`端口写入数据0xdf.由于`A20 Gate`定义在写输入端口的1号bit位上,因此`A20`成功开启.

> 问题2:如何初始化GDT表

> > 1.SEG(段)定义

```c
#define SEG_NULLASM                                             \
    .word 0, 0;                                                 \
    .byte 0, 0, 0, 0

#define SEG_ASM(type,base,lim)                                  \
    .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
    .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
        (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
```

以上代码出现在c语言头文件`asm.h`中,规定了空段和一般段的定义.其中一般段的定义需要`type`,`base`,`lim`三个参数.

> > 2.GDT(全局描述符表)定义

```assembly
gdt:
    SEG_NULLASM                                     # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
#初始化空段，代码段和数据段
gdtdesc:
    .word 0x17                                      # sizeof(gdt) - 1
    .long gdt                                       # address gdt
```

以上的代码,分为标签为`gdt`和标签为`gdtdesc`的两段汇编代码.

第一段代码,根据空段和一般段的定义,规定了`GDT`表的格式,用给定参数实例化具体的段.`GDT`表中包含了一个空段`SEG_NULLASM`,不同`type`参数的一个代码段和一个数据段.

第二段代码,定义一个`gdtdesc`表,包含一个`GDT`大小的值和`GDT`的起始地址,共8字节长.

> > 3.GDT初始化

```assembly
lgdt gdtdesc
```

以上的代码,将`gdtdesc`表的值装入全局描述符表寄存器`GDTR`中,初始化完成.

>问题3:如何使能和进入保护模式

> > 1.什么是CR0寄存器和PE位

`CR0`寄存器,是一种控制寄存器,起到控制实模式和保护模式切换的作用.

`PE`位,全称为`Protection Enable(保护位)`,是`CR0`寄存器的最低位,但被置1时直接进入保护模式.

> > 2.如何进入保护模式

```assembly
	.set CR0_PE_ON,             0x1 
	
	
	movl %cr0, %eax
    orl $CR0_PE_ON, %eax
    movl %eax, %cr0
```

以上代码是实现进入保护模式的功能的代码,分为两段.

第一段代码将`CR0_PE_ON`变量置为1.第二段代码,用`OR`指令只修改`CR0`中的`PE`位为1,进入保护模式.

> > 3.进入保护模式之后的工作流程

```assembly
 	# Jump to next instruction, but in 32-bit code segment.
    # Switches processor into 32-bit mode.
    ljmp $PROT_MODE_CSEG, $protcseg
```

以上的代码,通过长转移指令LJMP(跳转空间达到64KB的寻址空间),使得`cs`的基地址被重新修改.

```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment

    # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
    movl $0x0, %ebp
    movl $start, %esp
    call bootmain
```

以上代码,设置数据段寄存器`DS`,扩展段寄存器`ES`,标志段寄存器`FS`,全局段寄存器`GS`.

而后,建立堆栈段寄存器`SS`,并初始化`esp`和`ebp`的值.

最终,调用`bootmain`函数,进入`bootloader`的主体功能函数.