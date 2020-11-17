### 练习三

> 释放某虚地址所在的页并取消对应二级页表项的映射
>
> 当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。为此，需要补全在kern/mm/pmm.c中的page_remove_pte函数。

先读注释进行分析：

使用pte2page宏可以得到从页表项得到对用的物理页面对应的Page结构体，得到结构体之后，判断此页被引用的次数，如果只被引用一次，则对页面引用进行更改，降为0这个页可以被释放；如果被多次引用，则释放页表入口，清除二级页表项。

```c
if (*ptep & PTE_P) {//判断页表是否存在
        struct Page *page = pte2page(*ptep);//从页表项得到对应的物理页对应的Page结构体
        if (page_ref_dec(page) == 0) {//判断此页是否被多次引用
            free_page(page);
        }
        *ptep = 0;
        tlb_invalidate(pgdir, la);//使TLB无效，清除二级页表项
    }
```

> 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

有对应关系

在mmu.h中找到pte2page函数的定义，从页表项找到对应物理页的Page结构体，Page结构体和对应的物理页的起始地址是一一对应的

```c
static inline struct Page *
pte2page(pte_t pte) {
    if (!(pte & PTE_P)) {
        panic("pte2page called with invalid pte");
    }
    return pa2page(PTE_ADDR(pte));
}
```

> 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？鼓励通过编程来具体完成这个问题

由ld工具形成的ucore的起始虚拟地址是0xC0100000，bootloader把ucore放在了起始物理地址0x100000

PA（物理地址）=LA（线性地址）=VA（逻辑地址）-KERNBASE

先把KERNBASE从0xC000000改成0x00000000，再把起始虚拟地址从0xC0100000改成0x00100000