
练习0：填写已有实验

见代码

练习1：给未被映射的地址映射上物理页（需要编程）

完成do_pgfault（mm/vmm.c）函数，给未被映射的地址映射上物理页。
代码：

```
//页表项非空，可以尝试换入页面     
else{           
    if(swap_init_ok) {              
        struct Page *page=NULL;              
        //根据mm结构和addr地址，尝试将硬盘中的内容换入至page中             
        //此时的page还没有加入到队列中             
    if ((ret = swap_in(mm, addr, &page)) != 0) {                 
        cprintf("swap_in in do_pgfault failed\n");                 
         goto failed;             
    }              
//建立虚拟地址和物理地址之间的对应关系             
        page_insert(mm->pgdir, page, addr, perm);             
        //将此页面设置为可交换的              
        swap_map_swappable(mm, addr, page, 1);        
    }        
    else {             
         cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);            
 	    goto failed; 
    }    
} 
```

请在实验报告中简要说明你的设计实现过程。
do_pgfault()函数从CR2寄存器中获取页错误异常的虚拟地址，根据error code来确认(1)这个虚拟地址是否在某一个VMA的地址范围内(2)是否具有正确的权限。
满足以上要求，就分配一个物理页。  

请描述页目录项（Pag Director Entry）和页表（Page Table Entry）中组成部分对ucore实现页替换算法的潜在用处。

如果ucore的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？
产生页访问异常后，CPU把引起页访问异常的线性地址装到寄存器CR2中，并给出了出错码errorCode，说明了页访问异常的类型。ucore　OS会把这个值保存在struct trapframe 中tf_err成员变量中。而中断服务例程会调用页访问异常处理函数do_pgfault进行具体处理。


练习2：补充完成基于FIFO的页面替换算法（需要编程）

完成vmm.c中的do_pgfault函数。

代码：
```
//页表项非空，可以尝试换入页面     
else{           
    if(swap_init_ok) {              
        struct Page *page=NULL;              
        //根据mm结构和addr地址，尝试将硬盘中的内容换入至page中             
        //此时的page还没有加入到队列中             
        if ((ret = swap_in(mm, addr, &page)) != 0) {                 
        cprintf("swap_in in do_pgfault failed\n");                 
        goto failed;             
    }              
    //建立虚拟地址和物理地址之间的对应关系             
    page_insert(mm->pgdir, page, addr, perm);             
    //将此页面设置为可交换的              
    swap_map_swappable(mm, addr, page, 1);        
    }        
    else {             
 	    cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);            
 	    goto failed; 
    }    
} 
```
在实现FIFO算法的swap_fifo.c中完成map_swappable和swap_out_vistim函数。

代码：
```
static int  
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in) {      
    list_entry_t *head=(list_entry_t*) mm->sm_priv; 
    list_entry_t *entry=&(page->pra_page_link);        
    assert(entry != NULL && head != NULL); 
    //将最近用到的页面添加到次序队尾
    list_add(head, entry);     
    return 0; 
} 
static int  
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick) {       
    list_entry_t *head=(list_entry_t*) mm->sm_priv;          
    assert(head != NULL);      
    assert(in_tick==0);
    //用le指示需要被换出的页         
    list_entry_t *le = head->prev;      
    assert(head!=le);  
    //le2page宏可以根据链表元素获得对应的Page指针p
    struct Page *p = le2page(le, pra_page_link);      
    //将进来最早的页面从队列中删除      
    list_del(le);      
    assert(p !=NULL);       
    //将这一页的地址存储在ptr_page中      
    *ptr_page = p;      
    return 0; 
}
```
请在实验报告中简要说明你的设计实现过程。
FIFO替换算法会维护一个队列，队列按照页面调用的次序排列，越早被加载到内存的页面会越早被换出。
_fifo_map_swappable()函数将最近被用到的页面添加到算法所维护的次序队列，_fifo_swap_out_victim()函数是用来查询哪个页面需要被换出。


扩展练习 Challenge：实现识别dirty bit的 extended clock页替换算法（需要编程）
代码：（swap_fifo.c中）
```
struct swap_manager swap_manager_fifo =
{
     .name            = "fifo swap manager",
     .init            = &_fifo_init,
     .init_mm         = &_fifo_init_mm,
     .tick_event      = &_fifo_tick_event,
     .map_swappable   = &_fifo_map_swappable,
     .set_unswappable = &_fifo_set_unswappable,
     //.swap_out_victim = &_extended_clock_swap_out_victim,
     .swap_out_victim = &_fifo_swap_out_victim,
     .check_swap      = &_fifo_check_swap,
};
```
将
```
     //.swap_out_victim = &_extended_clock_swap_out_victim,
     .swap_out_victim = &_fifo_swap_out_victim,
```
改为
```
     .swap_out_victim = &_extended_clock_swap_out_victim,
     //.swap_out_victim = &_fifo_swap_out_victim,
```
启用。

```
static int
_extended_clock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
	 assert(head != NULL);
	 assert(in_tick==0);
     //将head指针指向最先进入的页面
	 list_entry_t?*le = head->prev;
	 assert(head!=le);
     //查找最先进入并且未被修改的页面
	 while(le!=head)
     {
	     struct Page *p = le2page(le, pra_page_link);
		 //获取页表项
         pte_t *ptep = get_pte(mm->pgdir, p->pra_vaddr, 0);
		 //判断获得的页表项是否正确
		 if(*ptep<0x1000)
		 break;
         //判断dirty bit
		 if(!(*ptep & PTE_D))
		 {
		     //如果dirty bit为0，换出
		     //将页面从队列中删除
		     list_del(le);
		     assert(p !=NULL);
		     //将这一页的地址存储在ptr_page中
             *ptr_page = p;
			 return = 0;
	     }
		 else
		 {
		     //如果为1，赋值为0，并跳过
             *ptep &= 0xffffffbf;
         }
         le = le->prev;
     }
     //如果执行到这里证明找完了一圈，所有页面都不符合换出条件
	 //那么强行换出最先进入的页面
	 le = head->prev;
	 assert(head!=le);
     struct Page *p = le2page(le, pra_page_link);
	 //将进来最早的页面从队列中删除
	 list_del(le);
	 assert(p !=NULL);
     //将这一页的地址存储在ptr_page中
	 *ptr_page = p;
	 return 0;
}
```
