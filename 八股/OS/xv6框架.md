# 程序启动流程

- kernel.ld链接脚本中规定了最终可执行文件的各个段的内存位置和大小，并设置了程序起始点是_entry标签。

- _entry标签位于entry.S汇编文件，这里主要设置每个hart的内核栈地址：事先用全局变量数组申请到了一块连续内存，大小=4KB*NCPU，为当前hart的sp寄存器设置对应的4KB内核栈的起始地址。

  设置完内核栈后，使用j start跳转到start.c中的start()函数。

- start函数中的主要工作：

  - 把mstatus的MPP域置为01，指明进入M-mode之前的模式是S-mode，则调用mret后CPU就会进入S-mode

  - mepc设置成main的地址，则调用mret后就会到mepc指向的地址处执行(即main函数)

  - 设置S-mode下页表的地址为0，因为0地址是不可使用的，所以就表示不使用页表，即之后的使用到的地址都是物理地址

  - medeleg和mideleg寄存器控制着将哪些exception和interrupt交给S-mode来处理(默认是直接进入M-mode)，这里设置所有exception和interrupt都委托给S-mode处理(w_medeleg(0xffff); w_mideleg(0xffff);)。

  - 使能S-mode下的外部中断、定时器中断、软中断(w_sie(r_sie() | SIE_SEIE | SIE_STIE | SIE_SSIE);)

  - 配置S-mode可以访问所有物理内存：w_pmpaddr0(0x3fffffffffffffull); w_pmpcfg0(0xf);

  - 定时器中断的初始化(timerinit())(包括: 处理函数地址设置、使能中断等)：

    - 设置第一次定时器中断的tick点：CLINT_MTIMECMP = CLINT_MTIME + interval

    - 设置M-mode下的trap handler是kernelvec.S中的timervec函数：w_mtvec((uint64)timervec)

      timervec的主要工作：

      - 更新CLINT_MTIMECMP寄存器的值为 当前值 + interval
      - 产生S-mode下的软中断，使得在软中断处理函数中切换进程(时间片调度)：sip(pending interrupts)寄存器设置为二进制10，即SSIP域被置1了，表示S-mode下的软中断产生了
      - mret

    - 使能M-mode下的全局中断、使能M-mode下的定时器中断：w_mstatus(r_mstatus() | MSTATUS_MIE); w_mie(r_mie() | MIE_MTIE);

  - mret： asm volatile("mret"); 回到mepc指向的地址处，即之前设置的main.c中的main()函数

- main()函数中进行各种初始化：

  - 设置内核页表地址(w_satp(MAKE_SATP(kernel_pagetable)))

  - 设置S-mode下的trap handler地址(w_stvec((uint64)kernelvec))

  - 通过PLIC(Platform Level Interrupt Controller)使能若干外部中断(设置优先级(\*(uint32\*)(PLIC + UART0_IRQ*4) = 1)、使能中断(PLIC_SENABLE)、设置阈值=0(PLIC_SPRIORITY))

  - buffer cache初始化：缓存磁盘中的若干block，便于之后快速从内存中读出来

  - inode table初始化：缓存磁盘中的若干inode

  - file table初始化：所有struct file的数组集合

  - 磁盘初始化：用内存来模拟磁盘，创建一块1000KB大小的数组fs_img。磁盘操作时，地址就是 fs_img基地址 + 磁盘偏移地址

    ```c
    uint64 diskaddr = b->blockno * BSIZE; // 磁盘偏移地址
    char *addr = (char *)fs_img + diskaddr; // fs_img基地址 + 磁盘偏移地址
    if(write){
        memmove(addr, b->data, BSIZE);
    } else {
        memmove(b->data, addr, BSIZE);
    }
    ```

  - 进入用户空间的第一个用户进程执行。



# trap流程

- 用户空间发生trap(ecall、中断等)
- 进入uservec函数：保存通用寄存器的值到trapframe空间，sp指向内核栈，satp指向内核页表，进入usertrap函数
- usertrap函数：设置stvec指向kernelvec，由scause寄存器判断trap原因，执行对应处理。处理完后，调用usertrapret函数
- usertrapret函数：设置stvec指向uservec，设置S-mode返回后到U-mode，进入userret函数
- userret函数：satp指向进程的页表，从trapframe空间恢复通用寄存器的值，sret返回到U-mode



# kill实现

- 用户进程调用kill函数，传入pid，ecall陷入内核
- usertrap中判断出是syscall进来的，取出系统调用号，发现是kill，调用sys_kill
- sys_kill中设置进程的killed标志为1，若进程处于sleeping状态，则唤醒
- 返回到usertrap，判断出进程的killed标志==1，则调用exit(-1)
- exit中关闭进程打开的所有文件、进程的工作目录的引用数-1、子进程的爹都换成initproc、唤醒父进程、保存退出状态、更新进程状态为ZOMBIE、调用shed函数让出CPU(不会再被调度，直到进程状态变为UNUSED并被分配)
- exit并没有回收进程的所有资源，只是标记了状态为ZOMBIE，会在父进程调用wait函数时回收子进程资源
- wait函数中不断遍历每个子进程，若找到进程状态为ZOMBIE的子进程，则调用freeproc函数释放其资源，并返回该子进程exit的状态
- freeproc函数释放进程的trapframe空间、释放进程页表和所关联的物理内存、标记进程状态为UNUSED
- 此时，进程才算真正地被kill掉，可以复用了。

