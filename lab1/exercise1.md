# 练习1

### Makefile调试

`make` ：Makefile文件告诉make命令怎样去编译和链接程序

`make V=` 查看make执行了哪些命令

`man make` 查看Linux下make命令的详细介绍

### 问题1

> 操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)

#### ucore.img 

```bash
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

```

* 调用totarget函数，将ucore.img作为参数传入，函数结果赋值给变量UCOREING。

* target目标文件UCOREING依赖于kernel和bootblock两个文件

> 在makefile文件第六行`V       := @`表示将@赋值给变量V，所以$(V)代指@。

> `dd`命令：用指定大小的块拷贝一个文件，并在拷贝的同时进行指定的转换。
>
> > 参数注释： 
> >
> > if=文件名：输入文件名，缺省为标准输入。即指定源文件。< if=input file >
> >
> > of=文件名：输出文件名，缺省为标准输出。即指定目的文件。< of=output file >
> >
> > conv=conversion：用指定的参数转换文件。`conv=notrunc`指不截短输出文件
> >
> > count=blocks：仅拷贝blocks个块，块大小等于ibs指定的字节数，ibs默认为512字节。
> >
> > seek=blocks：从输出文件开头跳过blocks个obs byte的块后再开始复制。obs默认为512字节。

* make执行命令的内容：首先从/dev/zero中读了10000*512块的空字节，生成空文件，接着将bootlock中的内容拷贝到目标文件，然后从输文件的512字节后继续写入kernel的内容。

#### kernel

```bash
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)

```

kernel的生成依赖于KOBJS和tools/kernel.ld，

``` $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)```链接各种文件输出给target

` @$(OBJDUMP) -S $@ > $(call asmfile,kernel)` 反汇编target输出到变量asmfile

`@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)` 输出target 的符号表，进行文本替换，写入symfile

#### bootblock

```bash
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
bootblock = $(call totarget,bootblock)
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
    @echo + ld $@
    $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
    @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
    @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
$(call create_target,bootblock)

```

bootblock 依赖于bootasm.o、bootmain.o、sign生成bootblock的编译指令为：

` ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o`

> -m 模拟i386上的链接器
>
> -nostdlib 不使用标准库
>
> -N 代码段和数据段均可读写
>
> -e 指定入口
>
> -Ttext 设定代码段开始位置
>
> -fno-builtin 除了使用_builtin前缀，不进行builtin函数优化

### 问题2

> 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

由外部执行程序sign来生成虚拟的硬盘主导扇区，查看sign.c代码

```C
 printf("'%s' size: %lld bytes\n", argv[1], (long long)st.st_size);
    if (st.st_size > 510) {
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }
    char buf[512];
```

```c
 buf[510] = 0x55;
    buf[511] = 0xAA;
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
```

硬盘主引导扇区有512个字节，并且在第511字节写入0x55，第512字节写入0xAA，扇区的最后两个字节是0x550xAA

