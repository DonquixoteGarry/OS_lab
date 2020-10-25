# EXERCISE4

>问题1: Boot loader 如何读取硬盘扇区的？

> > 1.LBA方式和其参数

在`bootloader`中,使用`LBA`的`PIO`方式来访问硬盘.`LBA`意为Logic Block Address(逻辑区块地址),而`PIO`(Port I/O)指的是通过CPU存储转发硬盘数据的技术.而`LBA`的`PIO`技术指的是使用CPU访问硬盘的I/O地址寄存器以访问硬盘的技术.

这一方式它需要`0x1F2`至`0x1F7`端口给出的多个参数和命令.其中`0x1F2`指定读取扇区的数量,`0x1F3`/`0x1F4`/`0x1F5`指定`LBA`参数的0-23位,`0x1F7`给出读指令`0x20`.

> > 2.insl函数参数和流程分析

```c
static inline void insl(uint32_t port, void *addr, int cnt) __attribute__((always_inline));
```

如上,头文件`x86.h`中定义了`insl`函数的三个参数.其中,第一个参数`port`给定开始读取的扇区,第二个参数`addr`给定读取硬盘扇区后应该存入的内存地址,第三个参数`cnt`指定读取双字(`dword`型变量)的数量.

```c
static inline void
insl(uint32_t port, void *addr, int cnt) {
    asm volatile (
            "cld;"
            "repne; insl;"
            : "=D" (addr), "=c" (cnt)
            : "d" (port), "0" (addr), "1" (cnt)
            : "memory", "cc");
}
```

以上是`insl`函数的具体定义,使用了`asm_volatile`插入内联汇编代码.

`cld`指令将方向寄存器`DF`置0,以设定源变址寄存器`SI`和目的变址寄存器`DI`自增.

`repne`指令设定指令循环,循环执行其后的`insl`指令.

`insl`指令将数据寄存器`DX`指定的 I/O 端口将`dword`类型的值输入拓展段寄存器`ES`和目的变址寄存器`DI/EDI`指定的内存位置(`ES`指定基址,`DI/EDI`指定偏移量).

而接下来两行内联汇编代码,则是由匹配限制符,限定寄存器的使用要求,要求`port`使用数据寄存器`EDX`,`cnt`使用计数寄存器`ECX`(实现计数功能),`addr` 使用源变址寄存器`EDI`(实现地址自增).

最后一行代码,使用`memory`和`cc`中是声明内联汇编代码中存在修改标志寄存器的行为,使得`gcc`编译器在使用内存时直接访问该内存位置而不是其拷贝(防止自增循环被破坏).

>> 3.扇区读取功能实现流程

```c
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

以上的代码,定义了什么是`readsect`函数——实现扇区(sector)读取的函数.`readsect`函数有两个参数`dst`和`secno`,分别指代目标位置(destination)和扇区号(sector number).`

在函数开始时,先等待硬盘空闲.然后,用`out`指令向`0x1F2`和`0x1F7`等6个端口写入`LBA`方式访问硬盘需要的参数和指令.然后等待硬盘空闲,最后调用`insl`函数从0x1F0读取SECTSIZE个字节到`dst`的位置,每次读一个双字(占4字节),读SECTSIZE/4次.

另外,将`readsect`函数进行简单包装,就能构成实现段读取的`readseg`函数.

>Boot loader 是如何加载 ELF 格式的 OS？

> > 1.什么是ELF格式

`ELF`格式,全称`Executable and linking format`,是`Linux`系统下常用的目标文件格式.

```c
struct elfhdr {
    uint32_t e_magic;     // must equal ELF_MAGIC
    uint8_t e_elf[12];
    uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    uint16_t e_machine;   // 3=x86, 4=68K, etc.
    uint32_t e_version;   // file version, always 1
    uint32_t e_entry;     // entry point if executable
    uint32_t e_phoff;     // file position of program header or 0
    uint32_t e_shoff;     // file position of section header or 0
    uint32_t e_flags;     // architecture-specific flags, usually 0
    uint16_t e_ehsize;    // size of this elf header
    uint16_t e_phentsize; // size of an entry in program header
    uint16_t e_phnum;     // number of entries in program header or 0
    uint16_t e_shentsize; // size of an entry in section header
    uint16_t e_shnum;     // number of entries in section header or 0
    uint16_t e_shstrndx;  // section number that contains section name strings
};

```

如上,头文件`elf.h`中定义了`bootloader`使用的`ELF`格式的定义.

变量成员`e_type`指定了`ELF`文件的文件类型,例如重定位类型,可执行类型等.

此外,还有变量成员`e_machine`(机器版本),`e_version`(文件版本),`e_entry`(程序入口地址),`e_phoff`(程序段表头相对elfhdr偏移位置),`e_shoff`(节头表相对elfhdr偏移量),`e_flags`(处理器特定标志),`e_ehsize`(`ELF`文件头长度),`e_phentsize`(程序头部长度),` e_phnum`(段个数),`e_shentsize`(节头部长度),`e_shnum`(节头部个数),`e_shstrndx`(节头部字符索引).还包括数组`e_elf`以指定目标操作系统的数据编码方式,版本和架构.

另外,其中的`elfhdr`结构体的的变量成员`e_magic`被赋予特定值`ELF_MAGIC`,作为判断`ELF`格式的标志.

> > 2.判断ELF格式的流程

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }
    ...
    ...
    ...
bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

以上的代码是`bootloader`的主要工作流程——`bootmain`函数中的格式判断部分.

`bootmain`首先调用了`readseg`函数,将硬盘第一页的内容读取,并强制类型转换为`ELFHDR`类型.

接下来,判断`ELFHDR`的标志——`elf_magic`.若在这个硬盘位置存储的并不是合法的`ELF`文件,则类型强制转换会使得成员变量`elf_magic`不等于`ELF_MAGIC`,以致判定不是ELF格式,跳转到`bad`分支.

> > 3.装载ELF格式文件的过程

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
   ...
   ...
   ...
    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	...
    ...
    ...
}
```

以上代码是`bootmain`函数的`ELF`文件装载部分.结构体`proghdr`定义在头文件`elf.h`中,建立的该类型指针`ph`和`eph`作为记录段信息的段表.

为指针`ph`赋值的语句,通过访问记录程序段表头对`ELF`文件头的偏移量`e_phoff`,将`ELF`表头位置加上偏移量,求得装载`ELF`文件的起始点.

为指针`eph`赋值的语句,通过访问`ELF`段表中表示段个数的成员变量`e_phnum`和 指针`ph`来指定装载`ELF`文件的结束点.

然后,通过循环语句,调用`readseg`函数循环读取装载指针`ph`和`eph`之间的程序段.

读取结束后,通过调用`ELF`的成员变量`e_entry`(记录入口地址),调用`ELF`的入口函数.