
��ϰ0����д����ʵ��

������

��ϰ1����δ��ӳ��ĵ�ַӳ��������ҳ����Ҫ��̣�

���do_pgfault��mm/vmm.c����������δ��ӳ��ĵ�ַӳ��������ҳ��
���룺
//ҳ����ǿգ����Գ��Ի���ҳ��     
else{           
    if(swap_init_ok) {              
        struct Page *page=NULL;              
        //����mm�ṹ��addr��ַ�����Խ�Ӳ���е����ݻ�����page��             
        //��ʱ��page��û�м��뵽������             
    if ((ret = swap_in(mm, addr, &page)) != 0) {                 
        cprintf("swap_in in do_pgfault failed\n");                 
         goto failed;             
    }              
//���������ַ�������ַ֮��Ķ�Ӧ��ϵ             
        page_insert(mm->pgdir, page, addr, perm);             
        //����ҳ������Ϊ�ɽ�����              
        swap_map_swappable(mm, addr, page, 1);        
    }        
    else {             
 	    cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);            
 	    goto failed; 
    }    
} 

����ʵ�鱨���м�Ҫ˵��������ʵ�ֹ��̡�
do_pgfault()������CR2�Ĵ����л�ȡҳ�����쳣�������ַ������error code��ȷ��(1)��������ַ�Ƿ���ĳһ��VMA�ĵ�ַ��Χ��(2)�Ƿ������ȷ��Ȩ�ޡ�
��������Ҫ�󣬾ͷ���һ������ҳ��  

������ҳĿ¼�Pag Director Entry����ҳ��Page Table Entry������ɲ��ֶ�ucoreʵ��ҳ�滻�㷨��Ǳ���ô���

���ucore��ȱҳ����������ִ�й����з����ڴ棬������ҳ�����쳣������Ӳ��Ҫ����Щ���飿
����ҳ�����쳣��CPU������ҳ�����쳣�����Ե�ַװ���Ĵ���CR2�У��������˳�����errorCode��˵����ҳ�����쳣�����͡�ucore��OS������ֵ������struct trapframe ��tf_err��Ա�����С����жϷ������̻����ҳ�����쳣������do_pgfault���о��崦��


��ϰ2��������ɻ���FIFO��ҳ���滻�㷨����Ҫ��̣�

���vmm.c�е�do_pgfault������

���룺
//ҳ����ǿգ����Գ��Ի���ҳ��     
else{           
    if(swap_init_ok) {              
        struct Page *page=NULL;              
        //����mm�ṹ��addr��ַ�����Խ�Ӳ���е����ݻ�����page��             
        //��ʱ��page��û�м��뵽������             
        if ((ret = swap_in(mm, addr, &page)) != 0) {                 
        cprintf("swap_in in do_pgfault failed\n");                 
        goto failed;             
    }              
    //���������ַ�������ַ֮��Ķ�Ӧ��ϵ             
    page_insert(mm->pgdir, page, addr, perm);             
    //����ҳ������Ϊ�ɽ�����              
    swap_map_swappable(mm, addr, page, 1);        
    }        
    else {             
 	    cprintf("no swap_init_ok but ptep is %x, failed\n",*ptep);            
 	    goto failed; 
    }    
} 

��ʵ��FIFO�㷨��swap_fifo.c�����map_swappable��swap_out_vistim������

���룺
static int  
_fifo_map_swappable(struct mm_struct *mm, uintptr_t addr, struct Page *page, int swap_in) {      
    list_entry_t *head=(list_entry_t*) mm->sm_priv; 
    list_entry_t *entry=&(page->pra_page_link);        
    assert(entry != NULL && head != NULL); 
    //������õ���ҳ����ӵ������β
    list_add(head, entry);     
    return 0; 
} 
static int  
_fifo_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick) {       
    list_entry_t *head=(list_entry_t*) mm->sm_priv;          
    assert(head != NULL);      
    assert(in_tick==0);
    //��leָʾ��Ҫ��������ҳ         
    list_entry_t *le = head->prev;      
    assert(head!=le);  
    //le2page����Ը�������Ԫ�ػ�ö�Ӧ��Pageָ��p
    struct Page *p = le2page(le, pra_page_link);      
    //�����������ҳ��Ӷ�����ɾ��      
    list_del(le);      
    assert(p !=NULL);       
    //����һҳ�ĵ�ַ�洢��ptr_page��      
    *ptr_page = p;      
    return 0; 
}

����ʵ�鱨���м�Ҫ˵��������ʵ�ֹ��̡�
FIFO�滻�㷨��ά��һ�����У����а���ҳ����õĴ������У�Խ�类���ص��ڴ��ҳ���Խ�类������
_fifo_map_swappable()������������õ���ҳ����ӵ��㷨��ά���Ĵ�����У�_fifo_swap_out_victim()������������ѯ�ĸ�ҳ����Ҫ��������


��չ��ϰ Challenge��ʵ��ʶ��dirty bit�� extended clockҳ�滻�㷨����Ҫ��̣�
���룺��swap_fifo.c�У�
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
��
     //.swap_out_victim = &_extended_clock_swap_out_victim,
     .swap_out_victim = &_fifo_swap_out_victim,
��Ϊ
     .swap_out_victim = &_extended_clock_swap_out_victim,
     //.swap_out_victim = &_fifo_swap_out_victim,
���á�
	 
static int
_extended_clock_swap_out_victim(struct mm_struct *mm, struct Page ** ptr_page, int in_tick)
{
     list_entry_t *head=(list_entry_t*) mm->sm_priv;
	 assert(head != NULL);
	 assert(in_tick==0);
     //��headָ��ָ�����Ƚ����ҳ��
	 list_entry_t?*le = head->prev;
	 assert(head!=le);
     //�������Ƚ��벢��δ���޸ĵ�ҳ��
	 while(le!=head)
     {
	     struct Page *p = le2page(le, pra_page_link);
		 //��ȡҳ����
         pte_t *ptep = get_pte(mm->pgdir, p->pra_vaddr, 0);
		 //�жϻ�õ�ҳ�����Ƿ���ȷ
		 if(*ptep<0x1000)
		 break;
         //�ж�dirty bit
		 if(!(*ptep & PTE_D))
		 {
		     //���dirty bitΪ0������
		     //��ҳ��Ӷ�����ɾ��
		     list_del(le);
		     assert(p !=NULL);
		     //����һҳ�ĵ�ַ�洢��ptr_page��
             *ptr_page = p;
			 return = 0;
	     }
		 else
		 {
		     //���Ϊ1����ֵΪ0��������
             *ptep &= 0xffffffbf;
         }
         le = le->prev;
     }
     //���ִ�е�����֤��������һȦ������ҳ�涼�����ϻ�������
	 //��ôǿ�л������Ƚ����ҳ��
	 le = head->prev;
	 assert(head!=le);
     struct Page *p = le2page(le, pra_page_link);
	 //�����������ҳ��Ӷ�����ɾ��
	 list_del(le);
	 assert(p !=NULL);
     //����һҳ�ĵ�ַ�洢��ptr_page��
	 *ptr_page = p;
	 return 0;
}	 