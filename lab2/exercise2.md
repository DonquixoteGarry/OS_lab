### 练习二

> 实现寻找虚拟地址和对应的页表项
>
> 请描述页目录项和页表中每个组成部分的含义和以及对ucore而言的潜在用处
>
> 如果ucore执行过程中存在访问内存，出现了页访问异常，请问硬件要做哪些事情

```c
pte:Page Table Entry 页表项 (二级页表项)
pde：Page Director Entry 页目录项(一级页表项)
get_pte函数：获取某一页表项，并以线性地址的形式返回其内核虚拟地址
la:需要寻址的内存的线性地址(在x86的分段地址转换下的线性地址)
create: 如果为页表分配一页将要设置的一个逻辑值
return：这个页表项的内核虚拟地址

使用KADDR()访问一个物理地址
阅读pmm.h 了解有用的宏
宏或函数：
PDX(la)   ： 虚拟地址la的页目录项的索引
KADDR(pa) ：  转换一个物理地址为其对应的内核虚拟地址
set_page_ref(page,1) ：页面被引用了一次
page2pa(page): 返回当前页面的物理地址
alloc_page() : 分配地址
memset(void *s, char c, size_t n) :将一片由s指向的内存区域的前n个byte初始化为c
PTE_P           0x001 ：在
PTE_W           0x002 ：可写
PTE_U           0x004 ：用户可访问
```

```c
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
pde_t *pdep = &pgdir[PDX(la)]; //通过线性地址查找到对应的页目录表项的索引并使pde_t指向该页表目录项
    if (!(*pdep & PTE_P)) {		//检查页表项PTE_P位是否为1(页是否在内存中)
        //不在则分配页    
        struct Page *page;		
        if (!create || (page = alloc_page()) == NULL) {//分配页框
            return NULL;
        }
        set_page_ref(page, 1);			//设置页的引用值
        uintptr_t pa = page2pa(page);	 //左移12位得到页面物理地址
        memset(KADDR(pa), 0, PGSIZE);	 //用0初始化(清理)页面
        *pdep = pa | PTE_U | PTE_W | PTE_P; //填写目录表项内容
        
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
    //返回la所在页表项的地址
    //PDE_ADDR(*pdep) ：舍弃*pdep的低12位
    //KADDR(PDE_ADDR(*pdep)) 获取对应虚拟地址
    //(pte_t *)KADDR(PDE_ADDR(*pdep)) 转换为页表指针
    }
    
```

问题一：

页目录项：前20位保存页表的物理地址

页表项：前20位存储虚拟地址对应的物理地址

问题二：

进行换页操作：

\- 首先 CPU 将产生页访问异常的线性地址放到 cr2 寄存器中

\- 然后就是和普通的中断一样保护现场，将寄存器的值压入栈中，然后压入 `error_code` 中断服务例程将外存的数据换到内存中来

\- 最后退出中断，回到进入中断前的状态
