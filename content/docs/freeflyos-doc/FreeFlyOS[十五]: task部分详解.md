---
title: Chapter-15 task部分详解
description: Chapter 15 of FreeFlyOS
toc: true
authors:
tags:
weight: 15
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## task.c

```
#include "task.h"
#include "../interrupt/syscall.h"
#include "../stl/elf.h"
#include "../mem/pmm.h"
#include "../mem/vmm.h"
#include "../mem/memlayout.h"
#include "../debug/debug.h"
#include "../sync/sync.h"
#include "../file/file.h"
#include "../file/inode.h"
#define HASH_SHIFT          10
#define HASH_LIST_SIZE      (1 << HASH_SHIFT)
#define pid_hashfn(x)       (hash32(x, HASH_SHIFT))


/*
#define __KERNEL_EXECVE(name, binary, size) ({                          \
            printk("kernel_execve: pid = %d, name = \"%s\".\n",        \
                    current->pid, name);                                \
            kernel_execve(name, binary, (unsigned int)(size));                \
        })
//#x 表示字符串操作符，即"x"
//##x##表示连接符  
#define KERNEL_EXECVE(x) ({                                             \
            extern unsigned char _binary_##x##_out_start[],  \
                _binary_##x##_out_size[];                    \
            __KERNEL_EXECVE(#x, _binary_##x##_out_start,     \
                            _binary_##x##_out_size);         \
        })

#define __KERNEL_EXECVE2(x, xstart, xsize) ({                           \
            extern unsigned char xstart[], xsize[];                     \
            __KERNEL_EXECVE(#x, xstart, (unsigned int)xsize);                 \
        })

#define KERNEL_EXECVE2(x, xstart, xsize)       __KERNEL_EXECVE2(x, xstart, xsize)
*/
// 绑定PID的哈希表
static list_entry_t hash_list[HASH_LIST_SIZE];

//PID 位图初始化
pidmap_t task_pidmap={pid_max,{0}}; 
static unsigned int volatile last_pid=0;

//就绪进程链表
static list_entry_t ready_task_list;
//所有进程链表
static list_entry_t all_task_list;
struct task_struct *task0;  //祖先进程，即进程0
struct task_struct *user_task; //第一个用户进程
//struct task_struct *task1;  //由进程0 do_fork出来的进程1
struct task_struct *current;  //指向当前进程

extern struct semaphore user_sema;

//MACOS下容易出现BUG
//静态全局变量设置为0值时，在运行的时候容易跑飞，所以为了避免出现BUG，在使用的时候应先定义0值
static unsigned int volatile nr_task=0; //当前所有进程数量  

void forkrets(struct trapframe *tf);

extern unsigned int __trapret;
extern struct file file_table[MAX_FILE_OPEN];
extern struct segdesc gdt[];
extern unsigned int new_pdt[PAGE_DIR_SIZE] __attribute__( (aligned(VMM_PAGE_SIZE) ) );
extern unsigned int user_pt_highmem[(unsigned int)0xC0000000/
((unsigned int)PAGE_TABLE_SIZE*(unsigned int)VMM_PAGE_SIZE)]
[PAGE_TABLE_SIZE]__attribute__( (aligned(VMM_PAGE_SIZE) ) );
struct task_struct *task0,*task1;
/*
*   kernel_task_init:创建第一个内核进程
*/
void kernel_task_init(void *function){

    /*就绪进程链表初始化*/
    list_init(&ready_task_list);
    /*所有进程链表初始化*/
    list_init(&all_task_list);
    /*哈希链表初始化*/
    for(int i=0;i<HASH_LIST_SIZE;i++){
        list_init(&hash_list[i]);
    }
    //分配task_struct结构体
    if((task0=alloc_task(KERNEL_TASK))==NULL){
        printk("alloc task error!\n");
    }
    
    /* 设置task0属性 */
    task0->state=STOPPED;   
    task0->counter=5;
    task0->priority=1; 
    last_pid=task0->pid=0;  //初始化task0的PID和last_pid
    set_task_name(task0,"kernel_task");
    task0->kernel_stack=(unsigned int)task0+VMM_PAGE_SIZE;
    task0->cr3=LA_PA((unsigned int)new_pdt);
    task0->cwd_inode_nr=0;

        
    task0->tf = (struct trapframe *)(task0->kernel_stack)- 1;
    task0->tf->tf_regs.reg_eax=0;
    task0->tf->tf_regs.reg_ebp=0;
    task0->tf->tf_regs.reg_ebx=0;
    task0->tf->tf_regs.reg_ecx=0;
    task0->tf->tf_regs.reg_edi=0;
    task0->tf->tf_regs.reg_edx=0;
    task0->tf->tf_regs.reg_esi=0;
    task0->tf->tf_regs.reg_oesp=0;

    task0->tf->tf_cs=KERNEL_CS;
    task0->tf->tf_ds=task0->tf->tf_es=task0->tf->tf_fs=task0->tf->tf_ss=KERNEL_DS;
    task0->tf->tf_gs=0;
    task0->tf->tf_eip=0;//function; //user_space1
    task0->tf->tf_eflags=(EFLGAS_IOPL_0|EFLAGS_MBS|EFLAGS_IF_1);

    task0->tf->tf_esp=task0->kernel_stack-sizeof(struct trapframe);
    
    /*设置用户的上下文*/
    task0->context.eip=function;//task0->tf->tf_eip;
    task0->context.esp= (unsigned int)(task0->tf); 
    task0->context.ebx=task0->tf->tf_regs.reg_ebx;
    task0->context.edx=task0->tf->tf_regs.reg_edx;

    /* 进程链表指向task0 */
    //ask_list=task0->link;   //待调试
    //memcpy(&(task_list),&(task0->link),sizeof(list_entry_t));
    //list_init(&task0->link);
    //插入就绪任务链表
    list_init(&task0->link);
    add_link(&task0->link);
    //插入所有任务链表
    list_init(&task0->all_link);
    add_all_link(&task0->all_link);
    task0->fd_table[0]=0;
    task0->fd_table[1]=1;
    task0->fd_table[2]=2;
    for(int i=3;i<MAX_FILE_OPEN;i++){
        task0->fd_table[i]=-1;
    }
    /* 当前进程指向task0 */
    current=task0;
    
    //clear();
    //printk("task0->counter:%08d!\n",task0->counter);
    //printk("current:%08X!\n",current);
    //printk("In task_init,current->counter=%08d\n",current->counter);
    /* 根据PID加入哈希链表 */
    add_pid_hash(task0);
    
    wakeup_task(task0);
    //这时候直接赋值，以免静态全局变量在不同编译器下跑飞
    //nr_task++;
    nr_task=1;
}
//设置PID位
static int set_pid_bit(int pid){
    //获取PID所在的字节号
    int chars=pid/8;
    //获取PID所在字节的偏移位
    int offset=pid%8;
    //置位
    set_char_bit(task_pidmap.bits[chars],offset,1);
}
//清除PID位
static int clear_pid_bit(int pid){
    //获取PID所在的字节号
    int chars=pid/8;
    //获取PID所在字节的偏移位
    int offset=pid%8;
    //清零
    set_char_bit(task_pidmap.bits[chars],offset,0);
}
//返回未被占用的PID号，若没有则返回-1
static int find_free_pid(){
    int i=0,k=0;
    while(k!=32768){
        for(int j=0;j<8;j++,k++){
            if((task_pidmap.bits[i]>>j)&1==0){
                return k;
            }
        }
        i++;
    }
    return -1;
}
//分配一个可用的PID
static int alloc_pid(){

    //若无多余的PID号
    if(!task_pidmap.nr_free){
        return -1;
    }
    /*
    ** 一般将PID号设置为上个进程PID号+1
    **  若PID号已设置到末尾，则查询位图中是否还有未被占用的PID号
    */
    int pid=(pid==32767)?find_free_pid():last_pid+1;
    
    //有效PID
    if(pid>=0){
        set_pid_bit(pid);
        task_pidmap.nr_free--;
    }
    last_pid=pid;
    return pid;
}
//释放一个PID
static void free_pid(int pid){
    clear_pid_bit(pid);
    task_pidmap.nr_free++;
}

// 设置进程名称
char *set_task_name(struct task_struct *task, const char *name) {
    memset(task->name, 0, sizeof(task->name));
    return memcpy(task->name, name, task_name_max);
}

// 获取进程名称
char *get_task_name(struct task_struct *task) {
    static char name[task_name_max + 1];
    memset(name, 0, sizeof(name));
    return memcpy(name, task->name, task_name_max);
}

//将新进程插入就绪进程链表队尾
static void add_link(list_entry_t *new){
    list_add_before(&ready_task_list,new);
}
//将新进程插入所有进程链表队尾
static void add_all_link(list_entry_t *new){
    list_add_before(&all_task_list,new);
}

//在进程链表中删除某个进程
static void remove_link(list_entry_t *node){
    list_del(node);
}

//根据PID加入到PID哈希表中
static void add_pid_hash(struct task_struct *task){
    list_add(hash_list+pid_hashfn(task->pid),&(task->hash_link));
}

//根据PID删除哈希表中的节点
static void remove_pid_hash(int pid){
    struct task_struct *task=find_task(pid);
    list_del(&(task->hash_link)); 
}

//给定PID，在哈希表查找进程
static struct task_struct* find_task(int pid){
    
    if(pid<0){
        return NULL;
    }
    //找到哈希链表头
    list_entry_t *head=&hash_list[pid_hashfn(pid)];
    list_entry_t *ite=head;
    
    //在哈希表头下的双向循环链表(不包含哈希表头)中查找PID对应的进程
    while((ite=list_next(ite))!=head){
        struct  task_struct *task=list_to_task(ite,hash_link);
        if(task->pid==pid){
            return task;
        }
    } 
    //未找到
    return NULL;
}

//给进程分配task_struct结构体,判断是用户进程还是内核进程
static struct task_struct* alloc_task(enum task_kind kind){
    struct task_struct *task;
    if(kind==KERNEL_TASK)
        task=vmm_malloc(VMM_PAGE_SIZE*2,1);
    else
        task=vmm_malloc(VMM_PAGE_SIZE*2,2);
    
    if(task!=NULL){
        task->state=UNRUNNABLE;
        task->counter=5;
        task->priority=0;
        task->pid=-1;
        memset(&(task->name),0,sizeof(task->name));
        task->kernel_stack=0;
        task->cr3=new_pdt;
        task->tf=NULL;
        memset(&(task->context),0,sizeof(task->context));
        task->magic=TASK_MAGIC;
        task->cwd_inode_nr=0;
        for(int i=0;i<MAX_FILE_OPEN;i++){
        task->fd_table[i]=-1;
    }
    }
    return task;
}

/* 由kernel_thread去执行function(func_arg) */
//static void kernel_thread(thread_func* function, void* func_arg) {
/* 执行function前要开中断,避免后面的时钟中断被屏蔽,而无法调度其它线程 */
//   intr_enable();
//  function(func_arg); 
//}

// forkret -- the first kernel entry point of a new thread/task
// NOTE: the addr of forkret is setted in copy_thread function
//       after switch_to, the current proc will execute here.
static void
forkret(void) {
    forkrets(current->tf);
}

// copy_thread - setup the trapframe on the  task's kernel stack top and
//             - setup the kernel entry point and stack of task
static void
copy_thread(struct task_struct *task, unsigned int esp, struct trapframe *tf) {
    //在内核栈顶分配一个中断帧大小
    task->tf = (struct trapframe *)(task->kernel_stack)- 1;
    //task->kernel_stack-=sizeof(struct trapframe);
    //将trapframe信息放入内核栈中
    *(task->tf) = *tf;
    task->tf->tf_regs.reg_eax = 0;
    //task->tf->tf_esp = esp;
    task->tf->tf_eflags |= FL_IF;

    //拷贝栈信息，即用户态之前的函数调用信息
    memcpy((unsigned int)task+sizeof(struct task_struct),(unsigned int)current+sizeof(struct task_struct),VMM_PAGE_SIZE-sizeof(struct task_struct));
    //更改下栈信息，防止破坏父进程的栈
    unsigned int *start=(unsigned int)task+sizeof(struct task_struct);
    unsigned int i;
    for(i=0;i<((unsigned int)VMM_PAGE_SIZE-sizeof(struct task_struct))/sizeof(unsigned int);i++){
        if((*(start+i)&0xFFFFF000)==((unsigned int)current&0xFFFFF000)){
            *(start+i)=(unsigned int)((unsigned int)task&(unsigned int)0xFFFFF000)+
            +(unsigned int)(*(start+i)&(unsigned int)0x00000FFF);
        }
    }
    //设置相同的栈地址
    task->tf->tf_esp=(task->tf->tf_esp-(unsigned int)current)+(unsigned int)task;
    task->context.eip=forkret;
    task->context.ebx=tf->tf_regs.reg_ebx;
    task->context.edx=tf->tf_regs.reg_edx;
    //task->context.eip = (unsigned int)forkret;//print_task1;
    task->context.esp =task->tf->tf_esp;  
}

/* 更新inode打开数 */
static void update_inode_open_cnts(struct task_struct* task) {
   int local_fd = 3, global_fd = 0;
   while (local_fd < MAX_FILE_OPEN) {
      global_fd = task->fd_table[local_fd];
      ASSERT(global_fd < MAX_FILE_OPEN);
      if (global_fd != -1) {
	 if (is_pipe(local_fd)) {
	    file_table[global_fd].fd_pos++;
	 } else {
	    file_table[global_fd].fd_inode->i_open_cnts++;
	 }
      }
      local_fd++;
   }
}

/* do_fork -     parent task for a new child task
 * @clone_flags: used to guide how to clone the child task
 * @stack:       the parent's user stack pointer. if stack==0, It means to fork a kernel thread.
 * @tf:          the trapframe info, which will be copied to child task's task->tf
 */
int
do_fork(unsigned int clone_flags, unsigned int stack, struct trapframe *tf) {
    struct  task_struct *task;

    //判断进程数是否达到最大值
    if(nr_task>pid_max){
        return -1;
    }
    
    //分配task_struct结构体
    if((task=alloc_task(USER_TASK)) == NULL){
        return -1;
    }
    //设置父进程指针
    task->parent=current;
    task->ppid=current->pid;
    ASSERT(current->state==0);
    
    //设置新进程内核栈
    task->kernel_stack=(unsigned int)task+VMM_PAGE_SIZE*2;
    
    //设置新进程的cr3
    copy_user_cr3(task);
    //在新进程的内核栈中设置中断帧，并设置中断上下文的eip和esp
    copy_thread(task,stack,tf);

    //复制父进程的文件表
    for(int i=0;i<MAX_FILE_OPEN;i++){
        task->fd_table[i]=current->fd_table[i];
    }
    //设置子进程文件表
    update_inode_open_cnts(task);
    //获取PID
    if((task->pid=alloc_pid())<0){
        return -1;
    }
    list_init(&task->link);
    //将新进程加入就绪进程链表中,加入到队尾中
    add_link(&(task->link));
    list_init(&task->all_link);
    //将新进程加入所有进程链表中,加入到队尾中
    add_all_link(&(task->all_link));
    //将新进程的PID加入到哈希表中
    add_pid_hash(task);
    nr_task++;

    wakeup_task(task);
    //schedule();
    return task->pid;
}

static void wakeup_task(struct task_struct *task){
    task->state=RUNNABLE;
}
// 创建内核线程
int kernel_thread(int (*fun)(void *), void *args, unsigned int flags) {
    struct trapframe tf;
    memset(&tf, 0, sizeof(struct trapframe));
    tf.tf_cs = KERNEL_CS;
    tf.tf_ds = tf.tf_es = tf.tf_ss = KERNEL_DS;
    tf.tf_regs.reg_ebx = (unsigned int)fun;
    tf.tf_regs.reg_edx = (unsigned int)args;
    tf.tf_eip = (unsigned int)kernel_thread_entry;

    return do_fork(flags ,0, &tf);
}

static void task_run(struct task_struct *task){
    //enum intr_status flag;
    if(task!=current){
        struct task_struct *prev=current;
        current=task;
        
        if(task==task0)
            intr_enable();
        else
            intr_disable();
        //local_intr_save(flag);
        //{
        set_ts_esp0(task->kernel_stack);
        lcr3(task->cr3);
        switch_to(&(prev->context),&(task->context));
        //printk("task_schedule!\n");
       // }
        //local_intr_restore(flag);
    }
}
/* 调度算法 */
void schedule(){
    //在进程链表中查找可运行的进程
    list_entry_t *head=&ready_task_list;
    list_entry_t *ite=head;
    struct  task_struct *task;

    //首先判断当前进程是不是时间片用完了或者是不是有用户进程响应
    if(current->state==RUNNABLE&&current->counter==0||user_sema.value==1){
        //若该线程只是时间片用完了，重新分配时间片，并将其放入就绪进程链表队尾
        current->counter=5;
        add_link(&current->link);
    }
    

    //在就绪进程链表中查找时间片不为0的可用进程
    while((ite=list_next(ite))!=head){
        task=list_to_task(ite,link);
        //找到一个可运行进程，则弹出就绪链表
        if(task->state==RUNNABLE&&task->counter!=0){
            remove_link(ite);
            break;
        }
    }

    //若无可调度进程，则运行进程0
    //if(task==current||task->state==UNRUNNABLE){
    //    task=task0;
    //}

    //运行进程
    task_run(task);
}
/* 当前线程将自己阻塞,标志其状态为stat. */
void thread_block(enum task_state stat) {
/* stat取值为TASK_BLOCKED,TASK_WAITING,TASK_HANGING,也就是只有这三种状态才不会被调度*/
   //ASSERT(stat == STOPPED);
   enum intr_status flag;
   local_intr_save(flag);
   {
       current->state=stat;
       schedule();
   }
   /* 待当前线程被解除阻塞后才继续运行下面的intr_set_status */
   local_intr_restore(flag);
}
/* 解除线程的阻塞状态 */
void thread_unblock(struct task_struct* task) {
    list_entry_t *head=&ready_task_list;
    list_entry_t *ite=head;
    enum intr_status flag;
    local_intr_save(flag);
    {
        ASSERT(task->state == STOPPED);
        if (task->state != RUNNABLE) {
            //若已堵塞的线程已经在就绪队列中
            while((ite=list_next(ite))!=head){
                if(task==list_to_task(ite,link))
                     PANIC("thread_unblock: blocked thread in ready_task_list\n");
            }
            task->state = RUNNABLE;
            add_link(&task->link);   //插入到就绪队列表尾部   
        }
    }
    local_intr_restore(flag);

}

/* 打印进程信息 */
static int
print_taskinfo(void *arg) {
    printk("this task, pid = %d, name = \"%s\"\n", current->pid, get_task_name(current));
    printk("To U: \"%s\".\n", (const char *)arg);
    printk("To U: \"en.., Bye, Bye. :)\"\n");
    return 0;
}

/* 进程退出 */
void do_exit(){
    printk("task exit!\n");
    //schedule();
    while(1);
}
// do_execve - call exit_mmap(mm)&put_pgdir(mm) to reclaim memory space of current process
//           - call load_icode to setup new memory space accroding binary prog.
int
do_execve(const char *name, unsigned int len, unsigned char *binary, unsigned int size) { 
  /*  struct elfhdr *elf = (struct elfhdr *)binary;
    //(3.2) get the entry of the program section headers of the bianry program (ELF format)
    struct proghdr *ph = (struct proghdr *)(binary + elf->e_phoff);
    struct proghdr *ph_end = ph + elf->e_phnum;
    unsigned int pdt=setup_pgdir();
    //有多少个程序头
    for (; ph < ph_end; ph ++){
        unsigned char *start = binary + ph->p_offset,map_start=round_down_to(start,VMM_PAGE_SIZE);
        unsigned char *end=ph->p_va + ph->p_filesz,map_end=round_up_to(end,VMM_PAGE_SIZE);
        vmm_map(pdt,map_start+(unsigned int)0x80000000,map_end+(unsigned int)0x80000000);
        memcpy(map_start+(unsigned int)0x80000000,map_start,map_end-map_start);
    }
    vmm_map(pdt,(unsigned int)0xA0000000,(unsigned int)0xA0002000);
    unsigned int user_stack=0xA0002000;
    current->cr3=LA_PA(pdt);
    lcr3(LA_PA(pdt));
    // setup trapframe for user environment
    struct trapframe *tf = current->tf;
    memset(tf, 0, sizeof(struct trapframe));
    tf->tf_cs = USER_CS;
    tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
    tf->tf_esp = user_stack;
    tf->tf_eip = elf->e_entry;*/
    return 0;
}
// kernel_execve - do SYS_exec syscall to exec a user program called by user_main kernel_thread
static int
kernel_execve(const char *name, unsigned char *binary, unsigned int size) {
    int ret, len = strlen(name);
    /*asm volatile (
        "int %1;"
        : "=a" (ret)
        : "i" (T_SYSCALL), "0" (SYS_exec), "d" (name), "c" (len), "b" (binary), "D" (size)
        : "memory");*/
    return ret;
}

// user_main - kernel thread used to exec a user program
static int
user_main(void *arg) {
    //KERNEL_EXECVE(exit);
}

//设置用户页表
void set_user_cr3(struct task_struct *task){
    unsigned int cr3_addr=vmm_malloc(VMM_PAGE_SIZE,2);
   // printk("cr3_addr:%08x",cr3_addr);
   // printk("new_pdt[0]:%08x",new_pdt[0]);
    memcpy(cr3_addr,new_pdt,VMM_PAGE_SIZE);
   // printk("new_pdt[0]:%08x",new_pdt[0]);
    unsigned int *pdt=(unsigned int *)cr3_addr;
    //printk("idx(PA_LA(DMA_START)):%08x",idx(PA_LA(DMA_START)));
    //printk("new_pdt[0x300]:%08x",new_pdt[0x300]);
    //printk("pdt[0x300]:%08x",pdt[0x300]);
   /* unsigned int pt_len=(unsigned int)HIGHMEM_START/
((unsigned int)PAGE_TABLE_SIZE*(unsigned int)VMM_PAGE_SIZE);

    for(unsigned int i=0;i<pt_len;i++){
            pdt[i+idx(PA_LA(DMA_START))]|=VMM_PAGE_KERNEL;//VMM_PAGE_USER
        }
    //user_task_print c10021b4 user_print_string c1001aa9 user_syscall c1001a20
    //pdt[idx(PA_LA(DMA_START))+4]|=VMM_PAGE_USER;
    unsigned int *pt=(unsigned int *)PA_LA((new_pdt[idx(PA_LA(DMA_START))]&VMM_PAGE_MASK));
    printk("pt[0]:%08x",pt[0]);
    for(unsigned int i=0,k=0,n=0;n<pt_len*(unsigned int)PAGE_TABLE_SIZE;i+=(unsigned int)VMM_PAGE_SIZE,n++){
        pt[k++]|=VMM_PAGE_KERNEL;//VMM_PAGE_USER
        //printk("i:%08ux\nj:%08ux\nk:%08ux\n",i,j,k);
    }*/
    unsigned int *cr3_ph_addr=(unsigned int *)cr3_addr;
    unsigned int *zh1=PA_LA((cr3_ph_addr[(unsigned int)((cr3_addr>>22)&0x3FF)]&VMM_PAGE_MASK));
    unsigned int phaddr=zh1[((cr3_addr&0x003FF000)>>12)]&VMM_PAGE_MASK;
    //unsigned int ph_addr=(unsigned int)(*cr3_ph_addr[((cr3_addr>>22)&0x3FF)/4]&VMM_PAGE_MASK)[((cr3_addr&0x003FF000)>>12)/4]&VMM_PAGE_MASK;
    //printk("phaddr:%08ux\n",phaddr);
   // pt[0x1001]|=VMM_PAGE_USER;
   // pt[0x1002]|=VMM_PAGE_USER;
    task->cr3=phaddr;
    task->cr3_va=cr3_addr;
}
/* 拷贝用户页表 */
void copy_user_cr3(struct task_struct *task){
    unsigned int cr3_addr=vmm_malloc(VMM_PAGE_SIZE,2);
    //先拷贝内核页表，因为该页信息在内核页表中
    memcpy(cr3_addr,new_pdt,VMM_PAGE_SIZE);
    //然后拷贝进程独有的页目录项,实际上还是和内核共用页表项，但页目录项是分开的。
    unsigned int *task_pdt=(unsigned int *)current->cr3_va;
    unsigned int *pdt=(unsigned int *)cr3_addr;
    for(int i=0;i<0x400;i++){
        pdt[i]=(task_pdt[i]&VMM_PAGE_PRESENT)?task_pdt[i]:pdt[i];
    }
    //获取该页物理地址
    unsigned int *cr3_ph_addr=(unsigned int *)cr3_addr;
    unsigned int *zh1=PA_LA((cr3_ph_addr[(unsigned int)((cr3_addr>>22)&0x3FF)]&VMM_PAGE_MASK));
    unsigned int phaddr=zh1[((cr3_addr&0x003FF000)>>12)]&VMM_PAGE_MASK;
    task->cr3_va=cr3_addr;
    task->cr3=phaddr;
}
/* 把列表plist中的每个元素elem和arg传给回调函数func,
 * arg给func用来判断elem是否符合条件.
 * 本函数的功能是遍历列表内所有元素,逐个判断是否有符合条件的元素。
 * 找到符合条件的元素返回元素指针,否则返回NULL. */
struct list_entry_t* list_traversal(struct list_entry_t* list, function func, int arg) {
   struct list_entry_t* ite = list;
/* 如果队列为空,就必然没有符合条件的结点,故直接返回NULL */
   if (list_empty(ite)) { 
      return NULL;
   }
    while((ite=list_next(ite))!=list){
        if (func(ite, arg)) {		  // func返回ture则认为该元素在回调函数中符合条件,命中,故停止继续遍历
	        return ite;
        }
    }
   return NULL;
}
/* 进程退出，由父进程释放子进程的资源 */
void task_exit(struct task_struct *task)
{
    list_entry_t *ready_head=&ready_task_list;
    list_entry_t *all_head=&all_task_list;
    list_entry_t *ite=ready_head;
    enum intr_status flag;
    local_intr_save(flag);
    {
        //进程设置为退出状态
        task->state=EXIT;  
        //在就绪进程列表中删除该进程
        while((ite=list_next(ite))!=ready_head){
            if(task==list_to_task(ite,link)){
                list_del_init(ite);
                break;
            }
        }
        //在所有进程列表中删除该进程
        ite=all_head;
        while((ite=list_next(ite))!=all_head){
            if(task==list_to_task(ite,all_link)){
                list_del_init(ite);
                break;
            }
        }
        //释放PID
        free_pid(task->pid);
        //释放进程所属页目录表
        vmm_free(task->cr3_va,VMM_PAGE_SIZE);
        //释放进程内核栈
        vmm_free((unsigned int)task+(unsigned int)VMM_PAGE_SIZE,VMM_PAGE_SIZE);
        //释放进程结构体信息和用户栈
        vmm_free((unsigned int)task,VMM_PAGE_SIZE);
    }
    local_intr_restore(flag);
}
/* 得到虚拟地址vaddr对应的pte指针*/
unsigned int* pte_ptr(unsigned int vaddr) {
   /* 先访问到页表自己 + \
    * 再用页目录项pde(页目录内页表的索引)做为pte的索引访问到页表 + \
    * 再用pte的索引做为页内偏移*/
   unsigned int* pte = (unsigned int*)(0xffc00000 + \
	 ((vaddr & 0xffc00000) >> 10) + \
	 ((vaddr & 0x003ff000) >> 12) * 4);
   return pte;
}
/* 用户进程自己释放资源 */
void release_prog_resource(struct task_struct *task){
   /*
    //查找用户空间占用的页（3G以下），设置页表项
   unsigned int* pgdir_vaddr = task->cr3_va;
   unsigned short user_pde_nr = 768, pde_idx = 0;
   unsigned int pde = 0;
   unsigned int* v_pde_ptr = NULL;	    // v表示var,和函数pde_ptr区分

   unsigned short user_pte_nr = 1024, pte_idx = 0;
   unsigned int pte = 0;
   unsigned int* v_pte_ptr = NULL;	    // 加个v表示var,和函数pte_ptr区分

   unsigned int* first_pte_vaddr_in_pde = NULL;	// 用来记录pde中第0个pte的地址
   unsigned int pg_phy_addr = 0;

   // 回收页表中用户空间的页框 
   while (pde_idx < user_pde_nr) {
      v_pde_ptr = pgdir_vaddr + pde_idx;
      pde = *v_pde_ptr;
      if (pde & 0x00000001) {   // 如果页目录项p位为1,表示该页目录项下可能有页表项
	 first_pte_vaddr_in_pde = pte_ptr(pde_idx * 0x400000);	  // 一个页表表示的内存容量是4M,即0x400000
	 pte_idx = 0;
	 while (pte_idx < user_pte_nr) {
	    v_pte_ptr = first_pte_vaddr_in_pde + pte_idx;
	    pte = *v_pte_ptr;
	    if (pte & 0x00000001) {
	       // 将pte中记录的物理页框直接在相应内存池的位图中清0 
	       pg_phy_addr = pte & 0xfffff000;
           if(pg_phy_addr!=0)
	       pmm_free(pg_phy_addr,PMM_PAGE_SIZE);
	    }
	    pte_idx++;
	 }
        // 将pde中记录的物理页框直接在相应内存池的位图中清0 
        //pg_phy_addr = pde & 0xfffff000;
        //pmm_free(pg_phy_addr,PMM_PAGE_SIZE);
      }
        pde_idx++;
   }*/
   /* 关闭进程打开的文件 */

      /* 关闭进程打开的文件 */
   unsigned char local_fd = 3;
   while(local_fd < MAX_FILE_OPEN) {
      if (task->fd_table[local_fd] != -1) {
	 if (is_pipe(local_fd)) {
	    unsigned int global_fd = fd_local2global(local_fd);  
	    if (--file_table[global_fd].fd_pos == 0) {
           vmm_free(file_table[global_fd].fd_inode,VMM_PAGE_SIZE);
	       file_table[global_fd].fd_inode = NULL;
	    }
	 } else {
	    sys_close(local_fd);
	 }
      }
      local_fd++;
   }
}
/* list_traversal的回调函数,
 * 查找pelem的parent_pid是否是ppid,成功返回true,失败则返回false */
static char find_child(list_entry_t *ite, int ppid) {
    struct task_struct *task=list_to_task(ite,all_link);
    if (task->ppid == ppid) {     // 若该任务的parent_pid为ppid,返回
      return 1;   // list_traversal只有在回调函数返回true时才会停止继续遍历,所以在此返回true
   }
   return 0;     // 让list_traversal继续传递下一个元素
}

/* list_traversal的回调函数,
 * 查找状态为TASK_HANGING的任务 */
static char find_hanging_child(list_entry_t *ite, int ppid) {
   struct task_struct* task = list_to_task(ite,all_link);
   if (task->ppid == ppid && task->state == HANGING) {
      return 1;
   }
   return 0; 
}

/* list_traversal的回调函数,
 * 将一个子进程过继给task0 */
static char task0_adopt_a_child(list_entry_t *ite, int pid) {
   struct task_struct* task = list_to_task(ite,all_link);
   if (task->ppid == pid) {     // 若该进程的parent_pid为pid,返回
      task->ppid = 0;
   }
   return 0;		// 让list_traversal继续传递下一个元素
}

/* 等待子进程调用exit,将子进程的退出状态保存到status指向的变量.
 * 成功则返回子进程的pid,失败则返回-1 */
int sys_wait(int* status) {

   while(1) {
      /* 优先处理已经是挂起状态的任务 */
      struct list_entry_t* child_elem = list_traversal(&all_task_list, find_hanging_child, current->pid);
      /* 若有挂起的子进程 */
      if (child_elem != NULL) {
	 struct task_struct* child_task = list_to_task(child_elem,all_link);
	 *status = child_task->exit_status; 

	 /* thread_exit之后,pcb会被回收,因此提前获取pid */
	 int child_pid = child_task->pid;

	 /* 2 从就绪队列和全部队列中删除进程表项*/
	 task_exit(child_task); 
	 /* 进程表项是进程或线程的最后保留的资源, 至此该进程彻底消失了 */

	    return child_pid;
      } 

      /* 判断是否有子进程 */
      child_elem = list_traversal(&all_task_list, find_child, current->pid);
      if (child_elem == NULL) {	 // 若没有子进程则出错返回
	    return -1;
      } else {
      /* 若子进程还未运行完,即还未调用exit,则将自己挂起,直到子进程在执行exit时将自己唤醒 */
	    thread_block(STOPPED); 
      }
   }
}

/* 子进程用来结束自己时调用 */
void sys_exit(int status) {
   current->exit_status = status; 
   if (current->ppid == -1) {
      PANIC("sys_exit: child_thread->parent_pid is -1\n");
   }

   /* 将进程child_thread的所有子进程都过继给task0 */
   list_traversal(&all_task_list, task0_adopt_a_child, current->pid);

   /* 回收进程child_thread的资源 */
   //release_prog_resource(current); 

   /* 如果父进程正在等待子进程退出,将父进程唤醒 */
   struct task_struct* parent_task = find_task(current->ppid);
   if (parent_task->state == STOPPED) {
      thread_unblock(parent_task);
   }

   /* 将自己挂起,等待父进程获取其status,并回收其pcb */
   thread_block(HANGING);
}


/*用户进程初始化*/
void  user_task_init(void *function){
    
    //分配task_struct结构体
    if((user_task=alloc_task(USER_TASK))==NULL){
        printk("alloc task error!\n");
    }
    /* 设置task0属性 */
    user_task->state=STOPPED;   
    user_task->counter=5;
    user_task->priority=1; 
    user_task->pid=alloc_pid();  //初始化task0的PID和last_pid
    user_task->ppid=0;
    set_task_name(user_task,"user_task");
    user_task->kernel_stack=(unsigned int)user_task+VMM_PAGE_SIZE*2;//后一页为内核栈

    set_user_cr3(user_task);//LA_PA(set_user_cr3());
    
    user_task->tf = (struct trapframe *)((unsigned int)user_task+VMM_PAGE_SIZE*2)- 1;
    user_task->tf->tf_regs.reg_eax=0;
    user_task->tf->tf_regs.reg_ebp=0;
    user_task->tf->tf_regs.reg_ebx=0;
    user_task->tf->tf_regs.reg_ecx=0;
    user_task->tf->tf_regs.reg_edi=0;
    user_task->tf->tf_regs.reg_edx=0;
    user_task->tf->tf_regs.reg_esi=0;
    user_task->tf->tf_regs.reg_oesp=0;

    user_task->tf->tf_cs=USER_CS;
    user_task->tf->tf_ds=user_task->tf->tf_es=user_task->tf->tf_fs=user_task->tf->tf_ss=USER_DS;
    user_task->tf->tf_gs=0;
    user_task->tf->tf_eip=function;//function; //user_space1
    user_task->tf->tf_eflags=(EFLGAS_IOPL_0|EFLAGS_MBS|EFLAGS_IF_1);

    //前一页为用户栈
    user_task->tf->tf_esp=(unsigned int)user_task+VMM_PAGE_SIZE;
    
    /*设置用户的上下文*/
    user_task->context.eip=__trapret;//user_task->tf->tf_eip;
    user_task->context.esp=(unsigned int)user_task+VMM_PAGE_SIZE; 
    user_task->context.ebx=user_task->tf->tf_regs.reg_ebx;
    user_task->context.edx=user_task->tf->tf_regs.reg_edx;

    user_task->fd_table[0]=0;
    user_task->fd_table[1]=1;
    user_task->fd_table[2]=2;
    for(int i=3;i<MAX_FILE_OPEN;i++){
        user_task->fd_table[i]=-1;
    }
    /* 进程链表指向task0 */
    //ask_list=task0->link;   //待调试
    //memcpy(&(task_list),&(task0->link),sizeof(list_entry_t));
    //list_init(&task0->link);
    
    list_init(&user_task->link);
    //add_link(&user_task->link);
    //插入所有任务链表
    list_init(&user_task->all_link);
    add_all_link(&user_task->all_link);
    
    //add_link(&user_task->link);
    //printk("task0->counter:%08d!\n",task0->counter);
    //printk("current:%08X!\n",current);
    //printk("In task_init,current->counter=%08d\n",current->counter);
    /* 根据PID加入哈希链表 */
    add_pid_hash(user_task);

    wakeup_task(user_task);

    
    nr_task++;
    
    //这时候直接赋值，以免静态全局变量在不同编译器下跑飞
    //nr_task++;
    current=user_task;
    set_ts_esp0(user_task->kernel_stack);
    lcr3(user_task->cr3);
    asm volatile ("movl %0, %%esp; jmp __trapret" : : "g" (user_task->tf) : "memory");
    //schedule();
    //task_run(user_task);
}
/* sys_fork系统调用，返回子进程PID号 */
/*int sys_fork(){

}*/

/* sys_print_task 系统调用，打印所有运行进程*/
void sys_print_task(){
    list_entry_t *head=&all_task_list;
    list_entry_t *ite=head;
    struct task_struct *task;
    printk("PID\t\tNAME\t\tSTATE\n");
    while((ite=list_next(ite))!=head){
        task=list_to_task(ite,all_link);
        printk("%d\t\t%s\t",task->pid,task->name);
        if(task->state==-1)
            printk("UNRUNNABLE\n");
        else if(task->state==0)
            printk("RUNNABLE\n");
        else
            printk("STOPPED\n"); 
    }
}
```



## task.h

```
#ifndef _TASK_H_
#define _TASK_H_

#include "../dt/dt.h"
#include "../mem/vmm.h"
#include "../mem/memlayout.h"
#include "../asm/asm.h"
#include "../interrupt/trap.h"
#include "../vga/vga.h"
#include "../stl/list.h"
#include "../stl/hash.h"
#include "../stl/defs.h"
#include "../file/fs.h"
#include "../file/file.h"
//进程魔数
#define TASK_MAGIC 0x19971211
//最大进程数量  pid号从0-32767 
#define pid_max 32768
//进程名最大值
#define task_name_max 20

//将list_entry_t转化为sturct task_struct
#define list_to_task(list_entry_addr,member)         \
    to_struct(list_entry_addr,struct task_struct,member)

#define EFLAGS_MBS (1<<1)
#define EFLAGS_IF_1  (1<<9)
#define EFLAGS_IF_0 0
#define EFLAGS_IOPL_3 (3<<12) 
#define EFLGAS_IOPL_0 (0<<12)

#define FL_IF           0x00000200  // Interrupt Flag
/* 自定义通用函数类型,它将在很多线程函数中做为形参类型 */
typedef void thread_func(void*);
/* 自定义函数类型function,用于在list_traversal中做回调函数 */
typedef char (function)(struct list_entry_t*, int arg);
// Saved registers for kernel context switches.
// Don't need to save all the %fs etc. segment registers,
// because they are constant across kernel contexts.
// Save all the regular registers so we don't need to care
// which are caller save, but not the return register %eax.
// (Not saving %eax just simplifies the switching code.)
// The layout of context must match code in switch.S.
struct context {
    unsigned int eip;
    unsigned int esp;
    unsigned int ebx;
    unsigned int ecx;
    unsigned int edx;
    unsigned int esi;
    unsigned int edi;
    unsigned int ebp;
};
enum task_state{
    UNRUNNABLE=-1,
    RUNNABLE=0,
    STOPPED=1,
    HANGING=2,
    EXIT=3,
};
enum task_kind{
    USER_TASK=0,
    KERNEL_TASK=1,
};
struct task_struct{
    enum task_state state;  //-1 unrunnable , 0 runnable, 1 stopped
    int counter; //计数器，已经运行多长时间
    int priority; //优先级
    int pid,ppid;
    char name[task_name_max]; //进程名
    unsigned int kernel_stack;    //内核栈
    unsigned int cr3;            //cr3基址
    unsigned int cr3_va;         //cr3虚拟地址（访问地址）
    struct task_struct *parent;  //父进程
    struct trapframe *tf;
    struct context context;
    list_entry_t link;                //进程状态链表
    list_entry_t all_link;            //所有进程链表
    list_entry_t hash_link;           //哈希链表
    unsigned int fd_table[MAX_FILE_OPEN]; //文件描述符数组
    unsigned int magic;
    unsigned int cwd_inode_nr;
    char exit_status;  //进程退出状态
};
union task_union
{
    struct task_struct task;
    unsigned char stack[VMM_PAGE_SIZE];
};

//进程号位映射
typedef struct pidmap
{
    unsigned int nr_free; //还剩下多少个进程号没有分配
    char bits[4096]; //0代表未被占用，1代表被占用
}pidmap_t;

// 内核线程入口函数 thread_entry.S
extern int kernel_thread_entry(void *args);
void kernel_task_init(void *function);
//void print1();
//void print2();
int kernel_thread(int (*fun)(void *), void *args, unsigned int flags); 

/*  PID管理函数 */
static int set_pid_bit(int pid);
static int clear_pid_bit(int pid);
static int find_free_pid();
static int alloc_pid();
static void free_pid(int pid);

/* 进程链表管理函数 */
static void add_link(list_entry_t *new);
static void add_all_link(list_entry_t *new);
static void remove_link(list_entry_t *node);

/* PID哈希表管理函数 */
static void add_pid_hash(struct task_struct *task);
static void remove_pid_hash(int pid);
static struct task_struct* find_task(int pid);

/* 进程名管理函数 */
char *set_task_name(struct task_struct *task, const char *name);
char *get_task_name(struct task_struct *task);
/* 分配task_struct结构体 */
static struct task_struct* alloc_task(enum task_kind kind);

/* fork产生新进程 */
static void forkret(void);
static void copy_thread(struct task_struct *task, unsigned int esp, struct trapframe *tf);
int do_fork(unsigned int clone_flags, unsigned int stack, struct trapframe *tf);
int kernel_thread(int (*fun)(void *), void *args, unsigned int flags);

/* 进程调度管理函数 */
static void task_run(struct task_struct *task);
static void wakeup_task(struct task_struct *task);
void schedule(); 
void thread_block(enum task_state stat);
void thread_unblock(struct task_struct* task);

static int print_taskinfo(void *arg); 
static void print_task1();
static void print_task2();
void do_exit();

int do_execve(const char *name, unsigned int len, unsigned char *binary, unsigned int size);
void set_user_cr3();
void copy_user_cr3(struct task_struct *task);
void task_exit(struct task_struct *task);
void user_task_init(void *function);

struct list_entry_t* list_traversal(struct list_entry_t* list, function func, int arg);
unsigned int* pte_ptr(unsigned int vaddr);
void sys_print_task();
int sys_wait(int* status);
void sys_exit(int status);
#endif
```



## switch.S

```
.text
.globl switch_to
switch_to:                      # switch_to(from, to)

    # 参数是结构体指针，所以需要进行两次内存寻址
    # save from's registers
    movl 4(%esp), %eax          # eax points to from
    popl 0(%eax)                # save eip !popl
    movl %esp, 4(%eax)
    movl %ebx, 8(%eax)
    movl %ecx, 12(%eax)
    movl %edx, 16(%eax)
    movl %esi, 20(%eax)
    movl %edi, 24(%eax)
    movl %ebp, 28(%eax)

    # eax=0
    # restore to's registers
    movl 4(%esp), %eax          # not 8(%esp): popped return address already
                                # eax now points to to
    movl 28(%eax), %ebp
    movl 24(%eax), %edi
    movl 20(%eax), %esi
    movl 16(%eax), %edx
    movl 12(%eax), %ecx
    movl 8(%eax), %ebx
    movl 4(%eax), %esp

    #  将寄存器恢复为下一个进程的进程上下文，下一步执行to->context.eip
    pushl 0(%eax)               # push eip

    ret


```



## thread_entry.h

```
.code32

.global kernel_thread_entry
.extern do_exit

kernel_thread_entry: # void kernel_thread(void)
    sti  #open interrrupt
    pushl %edx # push arg
    call *%ebx # call fn
    ret
    #pushl %eax # save the return value of fn(arg)
    #call do_exit # call do_exit to terminate current thread

```



## exec.c

```
#include "exec.h"
#include "../interrupt/syscall.h"
#include "../asm/asm.h"
#include "../mem/vmm.h"
#include "../mem/memlayout.h"
#include "task.h"
#define NULL ((void *)0)

extern unsigned int __trapret;
extern struct task_struct *current;
//extern unsigned int test_end;
typedef unsigned int Elf32_Word, Elf32_Addr, Elf32_Off;
typedef unsigned short Elf32_Half;

/* 32位elf头 */
struct Elf32_Ehdr {
   unsigned char e_ident[16];
   Elf32_Half    e_type;
   Elf32_Half    e_machine;
   Elf32_Word    e_version;
   Elf32_Addr    e_entry;
   Elf32_Off     e_phoff;
   Elf32_Off     e_shoff;
   Elf32_Word    e_flags;
   Elf32_Half    e_ehsize;
   Elf32_Half    e_phentsize;
   Elf32_Half    e_phnum;
   Elf32_Half    e_shentsize;
   Elf32_Half    e_shnum;
   Elf32_Half    e_shstrndx;
};

/* 程序头表Program header.就是段描述头 */
struct Elf32_Phdr {
   Elf32_Word p_type;		 // 见下面的enum segment_type
   Elf32_Off  p_offset;
   Elf32_Addr p_vaddr;
   Elf32_Addr p_paddr;
   Elf32_Word p_filesz;
   Elf32_Word p_memsz;
   Elf32_Word p_flags;
   Elf32_Word p_align;
};

/* 段类型 */
enum segment_type {
   PT_NULL,            // 忽略
   PT_LOAD,            // 可加载程序段
   PT_DYNAMIC,         // 动态加载信息 
   PT_INTERP,          // 动态加载器名称
   PT_NOTE,            // 一些辅助信息
   PT_SHLIB,           // 保留
   PT_PHDR             // 程序头表
};

/* 将文件描述符fd指向的文件中,偏移为offset,大小为filesz的段加载到虚拟地址为vaddr的内存 */
static char segment_load(int fd, unsigned int offset, unsigned int filesz, unsigned int vaddr) {
   unsigned int vaddr_first_page = vaddr & 0xfffff000;    // vaddr地址所在的页框
   unsigned int size_in_first_page = VMM_PAGE_SIZE - (vaddr & 0x00000fff);     // 加载到内存后,文件在第一个页框中占用的字节大小
   unsigned int occupy_pages = 0;
   /* 若一个页框容不下该段 */
   if (filesz > size_in_first_page) {
      unsigned int left_size = filesz - size_in_first_page;
      occupy_pages = (left_size+VMM_PAGE_SIZE)/VMM_PAGE_SIZE + 1;	     // 1是指vaddr_first_page
   } else {
      occupy_pages = 1;
   }

   sys_mmap(vaddr_first_page,vaddr_first_page+occupy_pages*VMM_PAGE_SIZE,EXEC_START);

   /* 为进程分配内存 
   unsigned int page_idx = 0;
   unsigned int vaddr_page = vaddr_first_page;
   while (page_idx < occupy_pages) {


      unsigned int* pde = pde_ptr(vaddr_page);
      unsigned int* pte = pte_ptr(vaddr_page);



      // 如果pde不存在,或者pte不存在就分配内存.
    // pde的判断要在pte之前,否则pde若不存在会导致
    // 判断pte时缺页异常 
      if (!(*pde & 0x00000001) || !(*pte & 0x00000001)) {
	 if (get_a_page(PF_USER, vaddr_page) == NULL) {
	    return false;
	 }
      } // 如果原进程的页表已经分配了,利用现有的物理页,直接覆盖进程体
      vaddr_page += VMM_PAGE_SIZE;
      page_idx++;
   } */

   sys_lseek(fd, offset, SEEK_SET);
   sys_read(fd, (void*)vaddr, filesz);
   return 1;
}

/* 从文件系统上加载用户程序pathname,成功则返回程序的起始地址,否则返回-1 */
static int load(const char* pathname) {
   int ret = -1;
   struct Elf32_Ehdr elf_header;
   struct Elf32_Phdr prog_header;
   memset(&elf_header, 0, sizeof(struct Elf32_Ehdr));

   int fd = sys_open(pathname, O_RDONLY);
   if (fd == -1) {
      return -1;
   }

   if (sys_read(fd, &elf_header, sizeof(struct Elf32_Ehdr)) != sizeof(struct Elf32_Ehdr)) {
      ret = -1;
      goto done;
   }

   /* 校验elf头 */
   if (memcmp(elf_header.e_ident, "\177ELF\1\1\1", 7) \
      || elf_header.e_type != 2 \
      || elf_header.e_machine != 3 \
      || elf_header.e_version != 1 \
      || elf_header.e_phnum > 1024 \
      || elf_header.e_phentsize != sizeof(struct Elf32_Phdr)) {
      ret = -1;
      goto done;
   }

   Elf32_Off prog_header_offset = elf_header.e_phoff; 
   Elf32_Half prog_header_size = elf_header.e_phentsize;

   /* 遍历所有程序头 */
   unsigned int prog_idx = 0;
   while (prog_idx < elf_header.e_phnum) {
      memset(&prog_header, 0, prog_header_size);
      
      /* 将文件的指针定位到程序头 */
      sys_lseek(fd, prog_header_offset, SEEK_SET);

     /* 只获取程序头 */
      if (sys_read(fd, &prog_header, prog_header_size) != prog_header_size) {
	 ret = -1;
	 goto done;
      }

      /* 如果是可加载段就调用segment_load加载到内存 */
      if (PT_LOAD == prog_header.p_type) {
	 if (!segment_load(fd, prog_header.p_offset, prog_header.p_filesz, prog_header.p_vaddr)) {
	    ret = -1;
	    goto done;
	 }
      }

      /* 更新下一个程序头的偏移 */
      prog_header_offset += elf_header.e_phentsize;
      prog_idx++;
   }
   ret = elf_header.e_entry;
done:
   sys_close(fd);
   return ret;
}

/* 用path指向的程序替换当前进程 */
int sys_execv(const char* path, const char* argv[]) {
   unsigned int argc = 0;
   while (argv[argc]) {
      argc++;
   }
   int entry_point = load(path);     
   if (entry_point == -1) {	 // 若加载失败则返回-1
      return -1;
   }

   /* 修改进程名 */
   memcpy(current->name, path, task_name_max);
   current->name[task_name_max-1] = 0;

   struct trapframe *tf=(struct trapframe *)
   ((unsigned int)current+VMM_PAGE_SIZE*2-sizeof(struct trapframe));
   tf->tf_regs.reg_ebx=(int)argv;
   tf->tf_regs.reg_ecx=argc;
   tf->tf_eip=(void*)entry_point;
   tf->tf_esp=(unsigned int)current+VMM_PAGE_SIZE;

   /* exec不同于fork,为使新进程更快被执行,直接从中断返回 */
   asm volatile ("movl %0, %%esp; jmp __trapret" : : "g" ((unsigned int)tf) : "memory");
   return 0;
}

```



## exec.h

```
#ifndef _EXEC_H_
#define _EXEC_H_

int sys_execv(const char* path, const char*  argv[]);
#endif
```

关于进程这一块，还是比较复杂的。

首先看下FreeFlyOS的进程描述符，如下图所示。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-5CpBLDPV-1609681604096)(readme.assets/image-20210103125344968.png)\]](https://img-blog.csdnimg.cn/20210103214720284.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


1、首先我们会创建一个内核进程，这些信息均已写好，该进程主要任务是怠速，也就是当其他进程无法运行时，该任务就会占用CPU。

2、接着手动创建一个用户进程，主要步骤如下，先构造好用户进程的信息，主要是用户进程的内核栈和用户本身使用的栈，一般在创建任务时，会分配2页给进程使用，前一页页尾作为用户自己使用的栈，后一页页尾作为用户内核栈。所以我们只需要在用户内核栈中构造好用户进程信息（USER_CS、USER_DS等），然后切换到用户内核栈，调用__trapret函数，模拟中断返回的后续操作，则该函数会将用户进程信息弹出，故当前状态变为USER_CS，即用户权限下的环境。

3、用户进程会直接调用user目录下的shell，从而实现用户和系统的交互操作。

4、一般情况下，时钟中断会进行进程切换，但由于增加了shell，所以当一个进程等待时，也会进行进程切换。进程切换时需要注意的是，更换内核栈和页表，内核栈是用户在进行系统调用时，内核权限下访问资源的栈，然后进行上下文切换，一般的上下文由以下寄存器组成。如果是线程切换，由于共用一套内核栈和页表，所以只需要进行上下文切换即可。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-cOvwYF4M-1609681604098)(readme.assets/image-20210103132009043.png)\]](https://img-blog.csdnimg.cn/20210103214732334.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


5、简单说下FreeFlyOS中fork的原理，假定有一个用户进程调用了fork代码，首先我们会创建一个子进程信息，关键在于构造子进程的用户栈信息和内核栈信息，首先子进程的用户栈需要拷贝父进程的用户栈，保存之前父进程的函数调用关系，不然等待子进程执行时，函数就会跑飞，但是需要改下栈中和进程地址有关的信息，也就是要构造两套地址不同、功能相同的栈，不然还是用了同样的栈。接着中断栈直接拷贝父进程的中断栈帧，但是需要改下esp的地址，让它指向用户进程栈，同时上下文中设置esp为当前中断栈地址，eip为forkre函数（调用__trapret）。当子进程数据构造完毕时，当时钟中断开始进程调度时，子进程会按照它的上下文信息执行，记住此时栈指向了中断栈帧的启示地址，首先调用forkret,然后forkret调用forkrets,接着把esp设置为中断栈帧的esp（用户栈），要记住内核权限转化为用户权限是软件模拟的过程，没有硬件自动切换栈的操作，这样子进程就会变成和父进程一样的用户进程，并且获取了之前的函数调用信息，开始下一步的操作。

![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-CtdldrSs-1609681604099)(readme.assets/image-20210103134220781.png)\]](https://img-blog.csdnimg.cn/20210103214745878.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


![\[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-8S1rTmpC-1609681604100)(readme.assets/image-20210103134240827.png)\]](https://img-blog.csdnimg.cn/20210103214757602.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


大概就讲这么多了，写文档的过程太枯燥了，去打游戏了，88！若有疑问可以联系295957410@qq.com
