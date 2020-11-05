# 练习1

> 1.注释翻译

> >  综述		

```
first-fit算法(简称FFMA)中,分配器保存了一个空闲块组成的表(俗称空闲表).当收到分配内存请求时,它依序遍历以找到第一个够大的能满足请求的块.
如果这个块比需求的大的多,那么它会被分割,没被选中的会分割出来的那块会成为表中的新空闲块.
应重写 `default_init`, `default_init_memmap`,`default_alloc_pages`, `default_free_pages`四个函数.
```

> > 实现细节		

````
(1)准备

为执行该算法,应按表处理空闲块,函数`free_area_t`正是为此而用.

头文件`list.h`之中的`list`是双链表的一个简单实现.我们应该知道如何使用`list_init`, `list_add`(`list_add_after`), `list_add_before`, `list_del`, `list_next`, `list_prev`.函数.

这里有一个将普通表结构转化为特殊表结构(如`页`结构)的方法,存储在头文件`memlayout.h`中的宏`le2page`中.
````

```
(2)函数 `default_init`

可复用演示中的 `default_init`函数初始化空闲表`free_list`和给`nr_free`置零.`nr_free`指空闲块页数总和.
```

```
(3)函数 `default_init_memmap`

调用关系:`kern_init` --> `pmm_init` --> `page_init` --> `init_memmap` -->`pmm_manager` --> `init_memmap`

此函数用于初始化空闲块(使用基地址`addr_base`和页数`page_number`作为参数).

为初始化空闲块,先得初始化空闲块的每一页(按照头文件`memlayout.h`).此过程需要(p为`page`型指针)

- 将`p->flag`置为`PG_property`(表示此页合法).
	ps.在文件`pmm.c`中的`pmm_init`函数中,`p->flag`被初始化为`PG_reserved`
- 如果此页空闲,且不为此块的第一页,那么`p->flag`置为 0
- 如果此页空闲,且为此块的第一页,那么`p->property`置为该块中的总页数
- 若p为空闲且未被引用,`p->ref`应该置为 0

然后使用`p->page_link`将此页其连接至空闲表`free_list`中.
	例 :`list_add_before(&free_list, &(p->page_link));` )

最后,更新`nr_free`的值
	例 :`nr_free += n`
```

```
(4)函数 `default_alloc_pages`

先找到空闲表中第一个够大的空闲块(块大小即页数大于`n`)并调整其大小,将其地址作为函数`malloc`的返回值.

一.遍历空闲表过程
list_entry_t le = &free_list;
while((le=list_next(le)) != &free_list) {...
	1.while循环中,创建一个page型结构体(*p指向该块首页)并检查其所在块大小是否大于n
 		struct Page *p = le2page(le, page_link);
        if(p->property >= n){ ...
    2.如果找到了这个块(*p),那意味着块大小符合要求,malloc函数能以此申请n页的空间.malloc函数请求其空间后,我们应将`PG_reserved`置为 1, `PG_property`置为 0, 并且断开其与空闲表的连接
    	ps.如果块大小大于n(`p->property > n`),我们应重新计算此空闲块中的空闲页数
    		例 :`le2page(le,page_link))->property = p->property - n;`
    3.重新计算剩余空闲页数总和`nr_free`
    4.返回这个块(*p)

二.如果不能找到够大的块,那么返回空值`NULL`
```

```
(5)函数 `default_free_pages`

将某页连接回空闲表(释放该页),并将某些小空闲块合并形成大空闲块.

1.根据将要释放的该页的基地址,找到空闲表中的适当位置以插入,因为空闲表是按地址从低到高排的.

2.重置该页的状态,例如给`p->ref`和`p->flags`置值.

3.尝试合并小块,但这将改变某些页的`p->property`值
```

>2.

​		

​		