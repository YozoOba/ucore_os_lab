# Lab2:物理内存管理

## 实验内容

本次实验包含三个部分。首先了解如何发现系统中的物理内存；然后了解如何建立对物理内存的初步管理，即了解连续物理内存管理；最后了解页表相关的操作，即如何建立页表来实现虚拟内存到物理内存之间的映射，对段页式内存管理机制有一个比较全面的了解。本实验里面实现的内存管理还是非常基本的，并没有涉及到对实际机器的优化，比如针对 cache 的优化等。如果大家有余力，尝试完成扩展练习。

## 练习0：填写已有实验

本实验依赖实验1。请把你做的实验1的代码填入本实验中代码中有“LAB1”的注释相应部分。提示：可采用`diff`和`patch`工具进行半自动的合并（merge），也可用一些图形化的比较`/merge`工具来手动合并，比如`meld`，`eclipse`中的`diff/merge`工具，`understand`中的`diff/merge`工具等。

---

利用`meld diff`实现lab1和lab2的合并，新建一个比较

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8k22do6ttj30h9084q42.jpg)

选择lab1中修改过的如下三个文件比较，然后修改lab2中对应的内容

```
kern/debug/kdebug.c
kern/init/init.c
kern/trap/trap.c
```

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8k259fq7mj30lk0enmzt.jpg)

## 练习1：实现 first-fit 连续物理内存分配算法（需要编程）

在实现`first fit`内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改`default_pmm.c`中的`default_init`，`default_init_memmap`，`default_alloc_pages`， `default_free_pages`等相关函数。请仔细查看和理解`default_pmm.c`中的注释。

---

### 1.1 数据结构

kern/mm/memlayout.h

#### 物理页Page

```
struct Page {
    int ref;                        // page frame's reference counter
    uint32_t flags;                 // array of flags that describe the status of the page frame
    unsigned int property;          // the num of free block, used in first fit pm manager
    list_entry_t page_link;         // free list link
};
```

* `ref`是映射此物理页的虚拟页个数
* `flags`是物理页的状态标记
* `property`记录在此块内的空闲页的个数
* `page_link`链接比它地址小和大的其他连续内存空闲块

用到后两个成员变量的都是这个连续内存空闲块地址最小的一页（Head Page）。

根据接下去几行对flag的描述可知：bit 0表示此页是否被保留（reserved），如果是被保留的页，则bit 0会设置为1，且不能放到空闲页链表中，即这样的页不是空闲页，不能动态分配与释放。bit 1表示此页是否是free的，如果设置为1，表示这页是free的，可以被分配；如果设置为0，表示这页已经被分配出去了，不能被再二次分配。

```
/* Flags describing the status of a page frame */
#define PG_reserved                 0       // if this bit=1: the Page is reserved for kernel, cannot be used in alloc/free_pages; otherwise, this bit=0 
#define PG_property                 1       // if this bit=1: the Page is the head page of a free memory block(contains some continuous_addrress pages), and can be used in alloc_pages; if this bit=0: if the Page is the the head page of a free memory block, then this Page and the memory block is alloced. Or this Page isn't the head page.
```

#### `free_area_t`数据结构

```
typedef struct {
    list_entry_t free_list;         // the list header
    unsigned int nr_free;           // # of free pages in this free list
} free_area_t;
```

包含了一个`list_entry`结构的双向链表指针和记录当前空闲页的个数的无符号整型变量`nr_free`，其中的链表指针指向了空闲的物理页。

### 1.2 `default_pmm.c`

#### `default_init`函数

```
//初始化free_area
static void
default_init(void) {
    list_init(&free_list);//列表的头指针
    nr_free = 0;//空闲页的数目
}
```

这个函数用于初始化`free_list`并将`nr_free`置0

#### `default_init_memmap`函数

```
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;//p为块基址
    for (; p != base + n; p ++) {
        assert(PageReserved(p));//检查是否为保留页
        p->flags = p->property = 0;设置标记位，flag为0表示可以分配，property为0表示不是base
        set_page_ref(p, 0);//清空引用
    }
    base->property = n;//增加全局总页数
    SetPageProperty(base);
    nr_free += n;//增加空闲页的数量
    list_add_before(&free_list, &(base->page_link));//按地址序，依次往后排列。
}
```

这个函数的功能是根据每个物理页帧的情况来建立空闲页链表（按照地址高低），循环把空闲物理页对应的`Page`结构中的`flags`和引用计数`ref`清零，`PageReserved(p)`用来判断该页是否为保留页，循环结束后把`Head Page`的property设为n，来记录在此块内的空闲页的个数，最后`list_add_before`用于把这个连续内存空闲块加到`free_area.free_list`指向的双向列表中，为将来的空闲页管理做好初始化准备工作。

#### `default_alloc_pages`函数

`first-fit`是最先匹配算法，思路是：从空闲块链表头开始找，找到一个满足大小的空闲块就分配出来。

* 优点：更多的使用低地址部分的空闲块，高地址部分有大空闲块被保留。
* 缺点：低地址部分不断被划分，留下很多小空闲块，使查询变慢。

```
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    // 需要页数大于剩余free页，return null
    if (n > nr_free) {
        return NULL;
    }
    struct Page *page = NULL;
    list_entry_t *le = &free_list;
    // 查找n个或以上空闲页块，若找到，则判断是否大过n，大于n的话则将其拆分 
    //  并将拆分后的剩下的空闲页块加回到链表中
    while ((le = list_next(le)) != &free_list)
    // 如果list_next(le)) == &free_list说明已经遍历完了整个双向链表
	{
    	//此处le2page就是将le的地址-page_link 在Page的偏移，从而找到 Page 的地址 
        struct Page *p = le2page(le, page_link);
        // 找到了一个满足的，就把这个空间（的首页）拿出来
        if (p->property >= n) {
            page = p;
            break;
        }
    }
    //如果找到了可行区域
    if (page != NULL) {
        // 这个可行区域的空间大于需求空间，拆分，将剩下的一段放到list中
        if (page->property > n) {
            struct Page *p = page + n;
            p->property = page->property - n;
            SetPageProperty(p);
            // 加入后来的，p
            list_add_after(&(page->page_link), &(p->page_link));
            // list_add(&free_list, &(p->page_link));
        }
        // 删除刚才被分配的空闲页
        list_del(&(page->page_link));
        // 更新空余空间的状态
        nr_free -= n;
        //page被使用了，所以把它的属性clear掉
        ClearPageProperty(page);
    }
    // 返回page
    return page;
}
```

这个函数功能是根据`first-fit`从空闲块链表表头开始查找找到第一块大小不小于n的块，然后分配出n个页。

首先如果所有空闲块的空闲页总数都没有n个，那么无法分配，直接return。通过`list_next`遍历空闲块链表。通过`le2page`宏（定义在`memlayout.h`中）可以获得对应的指向Page的指针p。然后通过`p->property`得到此空闲块的大小，如果`≥n`，就开始重新组织空闲块。循环把这个空闲块中的每一个页初始化，`SetPageReserved`把对应的`Page`结构中的`flags`标志设置为`PG_reserved` ，表示这些页已经被使用了，将来不能被用于分配。如果选中的块大于n，那么只取n个页，就需要修改剩下的块对应的`property`。最后在空闲块链表中删除掉分配出去的块。

#### `default_free_pages`函数

```
//释放掉n个页块，释放后也要考虑释放的块是否和已有的空闲块是紧挨着的，也就是可以合并的
//如果可以合并，则合并，否则直接加入双向链表
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    // 先更改被释放的这几页的标记位
    for (; p != base + n; p ++) {
        assert(!PageReserved(p) && !PageProperty(p));
        p->flags = 0;
        set_page_ref(p, 0);
    }
    // 将这几块视为一个连续的内存空间
    base->property = n;
    SetPageProperty(base);

    list_entry_t *next_entry = list_next(&free_list);
    // 找到base的前一块空块的后一块
    while (next_entry != &free_list && le2page(next_entry, page_link) < base)
        next_entry = list_next(next_entry);
    // 找到前面那块
    list_entry_t *prev_entry = list_prev(next_entry);
    // 找到insert的位置
    list_entry_t *insert_entry = prev_entry;
    // 如果和前一块挨在一起，就和前一块合并
    if (prev_entry != &free_list) {
        p = le2page(prev_entry, page_link);
        if (p + p->property == base) {
            p->property += base->property;
            ClearPageProperty(base);
            base = p;
            insert_entry = list_prev(prev_entry);
            list_del(prev_entry);
        }
    }
	// 后一块
    if (next_entry != &free_list) {
        p = le2page(next_entry, page_link);
        if (base + base->property == p) {
            base->property += p->property;
            ClearPageProperty(p);
            list_del(next_entry);
        }
    }
    // 加一下
    nr_free += n;
    list_add(insert_entry, &(base->page_link));
}
```

`default_free_pages`函数的实现是`default_alloc_pages`的逆过程，将释放掉的空闲块放回空闲块链表中，如果按照地址从小到大插入后和旁边的空闲块地址连续，就需要考虑空闲块的合并问题。

首先检查需要释放的块是否是被分配的，然后遍历按地址从小到大的顺序寻找空闲块要插入链表中的位置，循环将要被释放块的所有页插入空闲链表中，然后修改页的各个属性。`base+n == p`用来判断如果和下一个（高位方向）内存块的地址连续，那么就向高位地址合并，如果不是的话把le指针指向前一个内存块，`le!=&free_list && p==base-1`判断新加入的块是否能和低位的内存块合并，最后更新空闲页总数`nr_free`。

### 1.3 first-fit的改进&其他内存分配算法

#### first-fit优化

可以从数据结构的角度考虑优化`first-fit`，利用二叉搜索树中的线段树可以使`alloc`和`free`的复杂度由O(n)变为O(logn)。

#### next-fit

`next-fit`是循环首次匹配算法，思路是：不从链表头开始找，找到哪里下次就从那开始。

* 优点：内存中的空闲块分布更均匀，查询效率稳定
* 缺点：没有保留大空闲块
* 适用情况：不需要大空闲块的情况

#### best-fit

`best-fit`是最佳匹配算法，思路是：所有空闲块从小到大排序，这样每次分配出去的内存块都是当前能达到的最优的。

* 优点：每次都最优
* 缺点：留下许多小空闲块，排序耗时
* 适用情况：需要的大都为小空闲块的情况

#### worse-fit

`worse-fit`是最差匹配算法，思路是：所有空闲块从大到小排列。

* 优点：不容易留下许多小空闲块
* 缺点：没有保留大空闲块
* 适用情况：不需要大空闲块的情况

## 练习2：实现寻找虚拟地址对应的页表项（需要编程）

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的`get_pte`函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全`get_pte`函数`kern/mm/pmm.c`，实现其功能。请仔细查看和理解`get_pte`函数中的注释。`get_pte`函数的调用关系图如下所示：(`get_pte`函数的调用关系图)

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8l6vuc5b5j30aq050wew.jpg)

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

* 请描述页目录项`（Pag Director Entry）`和页表`（Page Table Entry）`中每个组成部分的含义和以及对`ucore`而言的潜在用处。
* 如果`ucore`执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

---

### 2.1 `get_pte`调用关系图

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8mdmr39d3j30cx07swf1.jpg)

`get_pte`给定一个虚拟地址，找出这个虚拟地址在二级页表中对应的项，如果此二级页表项不存在，则分配一个包含此项的二级页表。

`pte_t  *get_pte (pde_t *pgdir,  uintptr_t la, bool  create)`涉及到下面三个类型。

`pde_t`：一级页表的表项。而`pgdir`是一级页表本身，给出页表起始地址。
`pte_t`：二级页表的表项。
`uintptr_t`：线性地址，由于段式管理只做直接映射，所以它也是逻辑地址。

目前只有`boot_pgdir`一个页表，引入进程的概念之后每个进程都会有自己的页表。

### 2.2 `get_pte`函数

`PDX(la)`通过虚拟地址la得到一级页表项的入口地址。

`KADDR(pa)`由物理地址得到虚拟地址。

通过`alloc_page`函数获得一个空闲物理页作为页目录表（Page Directory Table，PDT），页目录表占4KB空间。宏定义在`kern/mm/pmm.h`中：

```
struct Page *alloc_pages(size_t n);
#define alloc_page() alloc_pages(1)
```

`alloc_page()`分配的页的地址并不是真正的页分配的地址，而是`Page`这个结构体所在的地址，需要通过`page2pa()`将`Page`结构体的地址转换为物理页地址的线性地址

PTE 页表

```
填写页目录项的内容为：页目录项内容 = (页表起始物理地址 &0x0FFF) | PTE_U | PTE_W | PTE_P
PTE_U：位3，表示用户态的软件可以读取对应地址的物理内存页内容
PTE_W：位2，表示物理内存页内容可写
PTE_P：位1，表示物理内存页存在
```

`kern/mm/pmm.c`

```
//get_pte - get pte and return the kernel virtual address of this pte for la
//        - if the PT contians this pte didn't exist, alloc a page for PT
// parameter:
//  pgdir:  the kernel virtual base address of PDT
//  la:     the linear address need to map
//  create: a logical value to decide if alloc a page for PT
// return vaule: the kernel virtual address of this pte
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
    // 段基址后得到的地址是linear_addr, ucore中目前va = la
    pde_t *pdep = &pgdir[PDX(la)]; // 找到它的一级页表项（指针），PDX，线性地址的前十位，page dir index
    if(!(*pdep & PTE_P)) // 看一级页表项，其实就是二级页表的物理地址，如果存在（证明二级页表存在），在二级页表中找到，并直接返回
    {   
        if(!create) // 不要求create，直接返回
            return NULL;
        // 否则alloc a page，建立二级页表，（成功的话）并设置这个page的ref为1，将内存也清空。
        struct Page* page = alloc_page(); 
        if(page == NULL)
            return NULL;
        set_page_ref(page, 1); //引用次数加一
        uintptr_t pa = page2pa(page);  // 页清空
        memset(KADDR(pa), 0, PGSIZE);
        // 在一级页表中，设置该二级页表入口
        *pdep = (pa & ~0xFFF) | PTE_P | PTE_W | PTE_U;
    }
    // PDE_ADDR 就是取了个 &，因为设置的时候取了 |。 得到的是二级页表真正的物理地址。
    // (pte_t *)KADDR(PDE_ADDR(*pdep)): 将物理地址转换为 二级页表的核虚拟地址
    // [PTX(la)] 加上la中相对二级页表的偏移
    // 取地址，返回
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];  // (*pdep)是物理地址
}
```

首先获取一级页表项，`PDX(la)`可以得到一级页表项对应的入口地址（`kern/mm/mmu.h`）。

如果`!(*pdep & PTE_P)`表示物理页不存在，那么需要根据`create`参数的值来处理是否创建新的二级页表。如果`create`参数为0，则`get_pte`返回`NULL`；如果`create`参数不为0，则`get_pte`需要申请一个新的物理页（通过`alloc_page`来实现）。

`set_page_ref`这个页被页表引用，引用次数加一。然后通过`page2pa`得到`page`的物理地址。`memset`把新申请的`PGSIZE`个页全部设定为零，因为这个页所代表的虚拟地址都没有被映射。设置`PTE_U 0x001| PTE_W 0x002| PTE_P 0x004`三个控制位，（对应的宏定义在`mmu.h`中），分别代表：物理页内存存在/物理内存页内容可写/用户态的软件可以读取对应地址的物理内存页内容。

最后返回页表地址。

### 2.3 请描述页目录项PDE和页表PTE中每个组成部分的含义和以及对`ucore`而言的潜在用处

查看`mmu.h`中相关的宏定义：

对应物理页面是否存在/对应物理页面是否可写/对应物理页面用户态是否可以访问/对应物理页面在写入时是否写透(可以直写回内存)/对应物理页面是否能被放入高速缓存/对应物理页面是否被访问/对应物理页面是否被写入/对应物理页面的页面大小/必须为零的部分/用户可自定义的部分。

```
/* page table/directory entry flags */
#define PTE_P           0x001                   // Present
#define PTE_W           0x002                   // Writeable
#define PTE_U           0x004                   // User
#define PTE_PWT         0x008                   // Write-Through
#define PTE_PCD         0x010                   // Cache-Disable
#define PTE_A           0x020                   // Accessed
#define PTE_D           0x040                   // Dirty
#define PTE_PS          0x080                   // Page Size
#define PTE_MBZ         0x180                   // Bits must be zero
#define PTE_AVAIL       0xE00                   // Available for software use
```

PDE高20位是页表地址，剩下的12位都为标志位，其中PDE的0，1，2，...，8，9-11位分别对应`PTE_P,PTE_W,PTE_U,PTE_PWT,PTE_PCD,PTE_A,PTE_MBZ=0,PTE_PS,PTE_AVAIL`。

PDE的组成示意图：

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8mju99f1dj30c0073dgf.jpg)

PTE高20位是物理页地址，PTE的0，1，2，...，8，9-11位分别对应`PTE_P,PTE_W,PTE_U,PTE_PWT,PTE_PCD,PTE_A,PTE_D,PTE_MBZ=0,Global,PTE_AVAIL`。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8mjvxzceoj30c0074mxo.jpg)

| bit  | PDE                                                          | PTE                                                        |
| ---- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| 0    | Present位，0不存在，1存在下级页表                            | 同                                                         |
| 1    | Read/Write位，0只读，1可写                                   | 同                                                         |
| 2    | User/Supervisor位，0则其下页表/物理页用户无法访问，1可以访问 | 同                                                         |
| 3    | Page level Write Through，1则开启页层次的写回机制，0不开启   | 同                                                         |
| 4    | Page level Cache Disable， 1则禁止页层次缓存，0不禁止        | 同                                                         |
| 5    | Accessed位，1代表在地址翻译过程中曾被访问，0没有             | 同                                                         |
| 6    | 忽略                                                         | 脏位，判断是否有写入                                       |
| 7    | PS，当且仅当PS=1且CR4.PSE=1，页大小为4M，否则为4K            | 如果支持 PAT 分页，间接决定这项访问的页的内存类型，否则为0 |
| 8    | 忽略                                                         | Global 位。当 CR4.PGE 位为 1 时,该位为1则全局翻译          |
| 9    | 忽略                                                         | 忽略                                                       |
| 10   | 忽略                                                         | 忽略                                                       |
| 11   | 忽略                                                         | 忽略                                                       |

### 2.4 如果`ucore`执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

1. 把引起页访问异常的线性地址放到CR2寄存器中
2. 之后需要往中断时的栈中压入EFLAGS,CS,EIP,ERROR CODE，如果这页访问异常很不巧发生在用户态，还需要先压入SS,ESP并切换到内核态
3. 最后根据IDT表查询到对应的也访问异常的ISR，跳转过去并将剩下的部分交给相关软件处理。

## 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解page_remove_pte函数中的注释。为此，需要补全在 kern/mm/pmm.c中的page_remove_pte函数。page_remove_pte函数的调用关系图如下所示：

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8mkb63r1aj309o023q34.jpg)

（page_remove_pte函数的调用关系图）

请在实验报告中简要说明你的设计实现过程。请回答如下问题：

* 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？
* 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题

---

### 3.1 `page_remove_pte`函数

`page_remove_pte`函数用于释放某虚地址所在的页并取消对应二级页表项的映射。

`pmm.c`413-447

```
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
    if (*ptep & PTE_P) { // 如果二级页表项存在
        struct Page *page = pte2page(*ptep); // 找到这个二级页表项对应的page
        if (page_ref_dec(page) == 0) // 自减该page的ref，如果为0，则free该page
            free_page(page);
        *ptep = 0; //将该page table entry置0
        //刷新tlb
        tlb_invalidate(pgdir, la); 
    }
}
```

首先判断页表中该表项是否存在，`pte2page`获得相应的`page`，如果引用次数减一后为0，（即该页表只被引用了一次），就释放改物理页，否则不能释放，`PTE置零`，刷新`TLB`。

### 3.2 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

数组中每一个Page对应物理内存中的一个页，由`mmu.h`的描述可知线性地址的高20位是页目录项索引PDX与页表项索引PTX的组合PPN。所以高20位可以对应page中的一项。有两种对应关系


- 可以通过 PTE 的地址计算其所在的页表的Page结构：

 将虚拟地址向下对齐到页大小，换算成物理地址(减 KERNBASE), 再将其右移 PGSHIFT(12)位获得在pages数组中的索引PPN，&pages[PPN]就是所求的Page结构地址。



- 可以通过 PTE 指向的物理地址计算出该物理页对应的Page结构：

 PTE 按位与 0xFFF获得其指向页的物理地址，再右移 PGSHIFT(12)位获得在pages数组中的索引PPN，&pages[PPN]就 PTE 指向的地址对应的Page结构。
```
// A linear address 'la' has a three-part structure as follows:
//
// +--------10------+-------10-------+---------12----------+
// | Page Directory |   Page Table   | Offset within Page  |
// |      Index     |     Index      |                     |
// +----------------+----------------+---------------------+
//  \--- PDX(la) --/ \--- PTX(la) --/ \---- PGOFF(la) ----/
//  \----------- PPN(la) -----------/
//
// The PDX, PTX, PGOFF, and PPN macros decompose linear addresses as shown.
// To construct a linear address la from PDX(la), PTX(la), and PGOFF(la),
// use PGADDR(PDX(la), PTX(la), PGOFF(la)).
```


### 3.3 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题


修改链接脚本，将内核起始虚拟地址修改为0x100000。

`tools/kernel.ld`

```
SECTIONS {
    /* Load the kernel at this address: "." means the current address */
    . = 0x100000;
...
```

物理地址由虚拟地址加上一个基地址得到，把基地址修改为0。

`kern/mm/memlayout.h`

```
#define KERNBASE            0xC0000000
修改为：
#define KERNBASE            0x00000000
```

`boot_pgdir[0] = boot_pgdir[PDX(KERNBASE)]`用来建立物理地址在0~4MB之内的三个地址间的临时映射关系，第四阶段从`gdt_init`函数开始，第三次更新了段映射，形成了新的段页式映射机制。

`pmm.c`

因此注释掉对应语句，原先功能为取消了0~4MB之内的临时映射关系。

```
//disable the map of virtual_addr 0~4M
//boot_pgdir[0] = 0;
//now the basic virtual memory map(see memalyout.h) is established.
//check the correctness of the basic virtual memory map.
//check_boot_pgdir();
```

这样虚拟地址就都等于物理地址了。

