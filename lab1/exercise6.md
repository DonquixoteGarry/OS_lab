# 练习6

### 问题1

> 中断描述表（保护模式下的中断向量表）中，一个表项占多少个字节？其中哪几位代表中断处理代码的入口？

在` kern/tarp/trap.c`中找到中断向量表的定义如下

```c
/* *
 * Interrupt descriptor table:
 *
 * Must be built at run time because shifted function addresses can't
 * be represented in relocation records.
 * */
static struct gatedesc idt[256] = {{0}};
```

又在` kern/mm/mmu.h`中找到结构`gatadesc`的定义如下

```c
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```

这个结构加起来是64bits=8bytes，一个表项占8字节

中断处理代码的入口是前32位和后16位决定的，逻辑地址决定处理代码入口，就是段选择子与段偏移这两个字段，看注释知道是 `gd_off_15_0`、`gd_ss`、`gd_off_31_16` 这三个字段，占前32位和后16位

### 问题2

> 请编程完善`kern/trap/trap.c`中对中断向量表进行初始化的函数`idt_init`。在`idt_init`函数中，依次对所有中断入口进行初始化。使用`mmu.h`中的`SETGATE`宏，填充`idt`数组内容。每个中断的入口由`tools/vectors.c`生成，使用`trap.c`中声明的`vectors`数组即可。

根据注释提示，

1. 所有的 ISR 入口地址都存储在 `__vectors` 这个变量中，而这个变量是由 `tools/vector.c` 这个文件生成 `kern/trap/vector.S` 这个汇编代码得来的。

   在代码中引入下面这一行代码声明这个外部变量：

   ```c
   extern uintptr_t __vectors[];
   ```

2. 用 ISR 的入口来初始化 IDT ，也就是 `kern/trap/trap.c` 文件中的 `idt[256]` 

   初始化IDT需要用到宏`SETGATE`

   ```c
   /* *
    * Set up a normal interrupt/trap gate descriptor
    *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
    *   - sel: Code segment selector for interrupt/trap handler
    *   - off: Offset in code segment for interrupt/trap handler
    *   - dpl: Descriptor Privilege Level - the privilege level required
    *          for software to invoke this interrupt/trap gate explicitly
    *          using an int instruction.
    * */
   #define SETGATE(gate, istrap, sel, off, dpl) {            \
       (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
       (gate).gd_ss = (sel);                                \
       (gate).gd_args = 0;                                    \
       (gate).gd_rsv1 = 0;                                    \
       (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
       (gate).gd_s = 0;                                    \
       (gate).gd_dpl = (dpl);                                \
       (gate).gd_p = 1;                                    \
       (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
   }
   ```

   参数注释

   * gate：中断描述符，`gatedesc`结构对象
   * istrap：用来判断是中断还是trap，布尔型变量
   * sel：段选择子

   > 段选择子在 `kern/mm/memlayout.h` 文件中有提到：
   >
   > ```c
   > /* global descriptor numbers */
   > #define GD_KTEXT    ((SEG_KTEXT) << 3)        // kernel text
   > #define GD_KDATA    ((SEG_KDATA) << 3)        // kernel data
   > #define GD_UTEXT    ((SEG_UTEXT) << 3)        // user text
   > #define GD_UDATA    ((SEG_UDATA) << 3)        // user data
   > #define GD_TSS        ((SEG_TSS) << 3)        // task segment selector
   > ```

   * off：段偏移
   * dpl：中断的优先级

   ```c
       int i;
       for(i=0;i<256;i++) {
           SETGATE(idt[i],0,GD_KTEXT,__vectors[i],DPL_KERNEL);//对整个idt数组进行初始化
       }
       SETGATE(idt[T_SWITCH_TOK],0,GD_KTEXT,__vectors[T_SWITCH_TOK],DPL_USER);//把所有的中断都初始化为内核级的中断
       lidt(&idt_pd);//使用lidt指令加载中断描述符表
   }
   ```

3. 最后用 `lidt` 这个指令告诉 CPU 中断向量表的地址了。

```c
lidt(&idt_pd);//使用lidt指令加载中断描述符表
```
### 问题三

> 请编程完善 trap.c 中的中断处理函数 trap，在对时钟中断进行处理的部分填写 trap 函数中处理时钟中断的部分，使操作系统每遇到 100 次时钟中断后，调用 `print_ticks` 子程序，向屏幕上打印一行文字 "100 ticks”。

```c
case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
        ticks ++;
        if (ticks % TICK_NUM == 0) {
            print_ticks();
        }
        break;
```

