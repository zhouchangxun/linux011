
# 系统启动与初始化
## *BIOS阶段
计算机上电后，x86的CPU自动进入实模式，并从地址0xFFFF0处自动执行程序指令，这个地址通常是ROM-BIOS中的地址。
BIOS执行某些检测，并在物理地址0处初始化中断向量表。
此后他将启动设备的第一个扇区(引导扇区)读入内存绝对地址0x7c00处，并跳转到此处执行。
Linux 0.11的引导扇区就是由bootsect.s汇编而来的，占512字节。

## *bootsect阶段
被BIOS加载并执行后，它进行如下操作：
* 首先将自己移动到0x90000~0x901ff处并接着执行
* 加载setup的4个扇区至0x90200~0x909ff内存处
* 显示 Loading system...
* 加载system模块至内存的0x10000处
* 将根设备号保存在引导扇区的倒数第三第四字节处即0x901fc~0x901fd处
* 跳转至0x90200执行setup的代码  

linux0.11中通过`SYSSIZE=0x3000`限定system最大不能超过`0x3000*16=192KB`,其实限定值可以是512KB`(0x90000-0x10000=0x80000)`
## *setup阶段
setup的工作主要是通过BIOS中断获取硬件参数放在内存的0x90000~0x901ff处，供以后的初始化程序使用。然后移动system模块至0x00000处，并使CPU进入*保护模式* ，随后跳转至0x00000处执行。过程如下：
* int 0x10获取光标位置 --> 0x90000~0x90001
* int 0x15获取系统扩展内存大小(KB)--> 0x90002~0x90003 
   __*扩展内存*__即除去1MB的内存，如物理内存有16MB，则__*扩展内存*__为15MB.
* int 0x10获取显卡参数 --> 0x90004~0x9000d
* 获取(copy)两个硬盘的参数表 -->0x90080~0x9009ff
* 将system模块（0x10000~0x8ffff）移至内存0x0000处
* 准备GDT，加载GDTR。(这里的GDT只是为了进入保护模式的临时GDT)

       gdt:
      .word	0,0,0,0		# dummy

      .word	0x07FF		# 8Mb - limit=2047 (2048*4096=8Mb)
      .word	0x0000		# base address=0
      .word	0x9A00		# code read/exec
      .word	0x00C0		# granularity=4096, 386

      .word	0x07FF		# 8Mb - limit=2047 (2048*4096=8Mb)
      .word	0x0000		# base address=0
      .word	0x9200		# data read/write
      .word	0x00C0		# granularity=4096, 386
* 准备IDT(也是临时的)并加载IDTR
    idt_48:
        .word	0			# idt limit=0
        .word	0,0			# idt base=0L
* 打开A20地址线
* 初始化8259A  
  *硬件中断号设置成0x20开始， 初始状态屏蔽8259A的所有中断请求*
* 将CR0寄存器的PE位置1，开启保护模式
* 跳转至system模块(0x00000)

-----
## *head.s阶段
此时CPU已经刚刚进入保护模式。head.s属于system模块的一部分，所以位于内存地址0x0处。
* 初始化除cs外的各段寄存器为0x10.(cs是跳转时CPU自己设置的)
* __初始化堆栈指针__：  
      lss _stack_start,%esp //_stack_start在sched.c中定义
      
      long user_stack[PAGE_SIZE >> 2];	// 定义堆栈指针，4K。指针指在最后一项。
      struct{  // 该结构用于设置堆栈ss:esp（数据段选择符，指针）
        long *a; //esp值设为user_stack数组末尾地址
        short b; //ss值设为0x10
      }stack_start ={&user_stack[PAGE_SIZE >> 2], 0x10};
* __设置idt__,将256个描述符都设成一样的:类型为中断们,DPL=0,选择符0x0008,  
处理函数为ignore_int,描述符内容如下：

|31~16          |            15~0|
|---------------|----------------|
|**ignore_int高16位**|property:**0x8e00**|
|selector:**0x0008** |**ignore_int低16位**|
* __设置gdt__,与setup中的gdt区别只是段限长变为16MB  
      gdt:	
      .quad 0x0000000000000000	/* NULL descriptor */
      .quad 0x00c09a0000000fff	/* 16Mb */
      .quad 0x00c0920000000fff	/* 16Mb */
      .quad 0x0000000000000000	/* TEMPORARY - don't use */
      .fill 252,8,0			/* space for LDT's and TSS's etc */

* 重新加载段寄存器ds,es,fs,gs,ss（因为gdt变了）
* 检测A20地址线是否开启成功
* 检查数学协处理器
* 制造进入main函数的现场：  
       after_page_tables:
        pushl $0 # These are the parameters to main :-)
        pushl $0 # 这些是调用main 程序的参数（指init/main.c）。
        pushl $0
        pushl $L6 # return address for main, if it decides to.
        pushl $_main # main函数入口。
        jmp setup_paging # 跳转至第198 行。
       L6:
        jmp L6 # main should never return here, but
        
* 设置分页  
 一个页目录和四个页表（共20KB）放在0x0000~0x4fff处，对0~16MB的内存进行恒等映射。页面属性为0x07
 0x5000~0x53ff处预留了1024字节给软盘驱动程序使用。
* 执行ret指令，返回到main函数执行。

## *main.c阶段

#### 划分内存空间:
      static long memory_end = 0;	// 机器具有的内存（字节数）。
      static long buffer_memory_end = 0;	// 高速缓冲区末端地址。
      static long main_memory_start = 0;	// 主内存（将用于分页）开始的位置。
#### mem_init内存管理初始化
用字节数组mem_map来记录1MB以上物理内存页的状态,其中的值表示该页被占用的次数，0表示该页空闲，当申请一页物理内存时该字节值增加1.
初始化时将mem_map[]所有项设为100(表示已占用)，然后将主内存区的mem_map[]设为0（空闲）。

    //输入参数start_mem表示主内存区开始地址
    // end_mem表示整个物理内存大小
    void mem_init(long start_mem, long end_mem)
    {
        int i;

        HIGH_MEMORY = end_mem;
        for (i=0 ; i<PAGING_PAGES ; i++)
            mem_map[i] = USED;
        i = MAP_NR(start_mem);
        end_mem -= start_mem;
        end_mem >>= 12;
        while (end_mem-->0)
            mem_map[i++]=0;
    }
#### trap_init 中断初始化
设置IDT，注册各种异常处理程序：  

    void trap_init(void)
    {
        int i;

        set_trap_gate(0,&divide_error);
        set_trap_gate(1,&debug);
        set_trap_gate(2,&nmi);
        set_system_gate(3,&int3);	/* int3-5 can be called from all */
        set_system_gate(4,&overflow);
        set_system_gate(5,&bounds);
        set_trap_gate(6,&invalid_op);
        set_trap_gate(7,&device_not_available);
        set_trap_gate(8,&double_fault);
        set_trap_gate(9,&coprocessor_segment_overrun);
        set_trap_gate(10,&invalid_TSS);
        set_trap_gate(11,&segment_not_present);
        set_trap_gate(12,&stack_segment);
        set_trap_gate(13,&general_protection);
        set_trap_gate(14,&page_fault);
        set_trap_gate(15,&reserved);
        set_trap_gate(16,&coprocessor_error);
        for (i=17;i<48;i++)
            set_trap_gate(i,&reserved);
        set_trap_gate(45,&irq13);
        outb_p(inb_p(0x21)&0xfb,0x21);
        outb(inb_p(0xA1)&0xdf,0xA1);
        set_trap_gate(39,&parallel_interrupt);
    }
#### blk_dev_init
将请求项数组中所有项设为空闲:

    void blk_dev_init(void) {
        int i;
        for (i=0 ; i<NR_REQUEST ; i++) {
            request[i].dev = -1;
            request[i].next = NULL;
        }
    }
#### chr_dev_init
该函数没有做任何事：(tty_io.c) 

      void chr_dev_init(void)
      {
      }
#### tty_init 
    // tty_io.c
    void tty_init(void){
        rs_init(); // 初始化串口终端 serial.c 
        con_init(); // 初始化控制台终端 console.c
    }
#### time_init
#### sched_init
准备task0的运行环境，设置时钟中断。  
 * 在GDT中设置task0的tss和ldt描述符项
        set_tss_desc(gdt+FIRST_TSS_ENTRY,&(init_task.task.tss));
        set_ldt_desc(gdt+FIRST_LDT_ENTRY,&(init_task.task.ldt));
 * 清空除task0以外进程的进程表和tss、ldt描述符
        p = gdt+2+FIRST_TSS_ENTRY;
        for(i=1;i<NR_TASKS;i++) {
            task[i] = NULL;
            p->a=p->b=0;
            p++;
            p->a=p->b=0;
            p++;
        }
 * 清除标志寄存器的NT标志位
 * 加载task0的tss段选择符到tr寄存器
 * 加载task0的ldt选择符到ldtr寄存器
 * 配置8253定时器，使它每10ms发出一个IRQ0
        outb_p(0x36,0x43);		/* binary, mode 3, LSB/MSB, ch 0 */
        outb_p(LATCH & 0xff , 0x40);	/* LSB */
        outb(LATCH >> 8 , 0x40);	/* MSB */
 * 设置时钟中断处理程序为timer_interrupt
 * __使能时钟中断__
 * 设置系统调用中断处理程序为system_call


#### buffer_init 高速缓冲初始化
#### hd_init 硬盘初始化  
    void hd_init(void) {
        // 设置块设备表中硬盘的request_fn为do_hd_request()
        blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;
        // 设置硬盘中断入口
        set_intr_gate(0x2E,&hd_interrupt);
        outb_p(inb_p(0x21)&0xfb,0x21);
        outb(inb_p(0xA1)&0xbf,0xA1); // 使能硬盘中断
    }
#### floppy_init  
软盘初始化,设置块设备表中软盘项的request_fn，打开软盘中断：    
    
    void floppy_init(void){
        blk_dev[MAJOR_NR].request_fn = DEVICE_REQUEST;
        set_trap_gate(0x26,&floppy_interrupt);
        outb(inb_p(0x21)&~0x40,0x21);
    }
#### sti 开中断
允许CPU接收中断。经过前面的初始化过程，必要的中断入口程序都设置好了，可以打开中断，使task0可以通过系统调用创建进程，使CPU能产生时钟中断，从而能够将创建好的task1调度运行，此后操作系统就运转起来了。
#### move_to_user_mode
进入用户态，开始运行task0(进程0)    
    
    #define move_to_user_mode() \
    __asm__ ("movl %%esp,%%eax\n\t" \  //保存esp到eax
        "pushl $0x17\n\t" \            // 压入ss
        "pushl %%eax\n\t" \            // 压入保存在eax中的esp
        "pushfl\n\t" \                 // 压入eflags
        "pushl $0x0f\n\t" \            // 压入task0的cs
        "pushl $1f\n\t" \          // 压入task0的eip,即下面的标号1处
        "iret\n" \                 // 开始进入task0
        "1:\tmovl $0x17,%%eax\n\t" \
        "movw %%ax,%%ds\n\t" \     // 初始化段寄存器
        "movw %%ax,%%es\n\t" \
        "movw %%ax,%%fs\n\t" \
        "movw %%ax,%%gs" \
        :::"ax")

#### fork
#### init
该函数首先调用setup系统调用,对应的处理函数sys_setup()位于hd.c文件。  
sys_setup()做了如下工作：  
  * 1.用硬盘参数信息填充`struct hd_i_struct hd_info[]`数组,包括磁头数,每磁道扇区数,柱面数等  
  * 2.根据`hd_info[1].cyl`是否为0判断是否只有一个硬盘还是有两个硬盘(最多支持两个硬盘),给硬盘数NR_HD赋值(1或2)  
  * 3.设置硬盘分区结构数组hd[],(只填充hd[0]和hd[5])  
  * 4.根据CMOS信息真正确定硬盘个数(0或1或2)  
  * 5.读取硬盘第一扇区确定分区结构，填充hd[]其他项  
  * 6.打印"Partition table ok"(如果有硬盘的话)  
  * 7.rd_load()尝试创建并加载虚拟磁盘  
  * 8.mount_root()安装根文件系统
  
---

      



