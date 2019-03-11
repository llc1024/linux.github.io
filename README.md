# linux内核工作原理

学号241，姓名刘乐成，原创作品转载请注明出处 + [https://github.com/mengning/linuxkernel/](https://github.com/mengning/linuxkernel/)。

## 实验环境

VMware Workstation 14 Player虚拟机
Ubuntu 64位系统

## 实验目的

完成一个简单的时间片轮转多道程序内核代码，参考代码见[mykernel版本库](https://github.com/mengning/mykernel)。

## 实验步骤

1.在安装QUME前先要更新系统

```
sudo apt-get update
```

2.更新之后安装QUME

```
sudo apt-get install qemu
sudo ln -s /usr/bin/qemu-system-i386 /usr/bin/qemu
```

![install QUME](C:\Users\llc\Pictures\TIM截图20190310144401.png)

3.下载并编译mykernel内核

```
wget https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.9.4.tar.xz # download Linux Kernel 3.9.4 source code
wget https://raw.github.com/mengning/mykernel/master/mykernel_for_linux3.9.4sc.patch # download mykernel_for_linux3.9.4sc.patch
xz -d linux-3.9.4.tar.xz
tar -xvf linux-3.9.4.tar
cd linux-3.9.4
patch -p1 < ../mykernel_for_linux3.9.4sc.patch
make allnoconfig
make
```

4.使用QUME启动内核

```
qemu -kernel arch/x86/boot/bzImage
```

从qemu窗口中我们可以看到my_start_kernel在执行，同时my_timer_handler时钟中断处理程序周期性执行。
所以想要实现时间片轮转程序就应当修改这两个函数。

5.修改mykernel内核代码，完成简单的时间片轮转多道程序内核代码

首先是头文件mypcb.h
```
#define MAX_TASK_NUM 10 // max num of task in system
#define KERNEL_STACK_SIZE 1024*8
#define PRIORITY_MAX 30 //priority range from 0 to 30

/* CPU-specific state of this task */
struct Thread {
    unsigned long	ip;//point to cpu run address
    unsigned long	sp;//point to the thread stack's top address
    //todo add other attrubte of system thread
};
//PCB Struct
typedef struct PCB{
    int pid; // pcb id 
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    char stack[KERNEL_STACK_SIZE];// each pcb stack size is 1024*8
    /* CPU-specific state of this task */
    struct Thread thread;
    unsigned long	task_entry;//the task execute entry memory address
    struct PCB *next;//pcb is a circular linked list
    unsigned long priority;// task priority ////////
    //todo add other attrubte of process control block
}tPCB;

//void my_schedule(int pid);
void my_schedule(void);
```
头文件主要定义了进程跟PCB的结构。

其次是mymain.c
```
#include "mypcb.h"

tPCB task[MAX_TASK_NUM];
tPCB * my_current_task = NULL;
volatile int my_need_sched = 0;

void my_process(void);
unsigned long get_rand(int );

void sand_priority(void)
{
	int i;
	for(i=0;i<MAX_TASK_NUM;i++)
		task[i].priority=get_rand(PRIORITY_MAX);
}
void __init my_start_kernel(void)
{
    int pid = 0;
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    // set task 0 execute entry address to my_process
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(pid=1;pid<MAX_TASK_NUM;pid++)
    {
        memcpy(&task[pid],&task[0],sizeof(tPCB));
        task[pid].pid = pid;
        task[pid].state = -1;
        task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
	task[pid].priority=get_rand(PRIORITY_MAX);//each time all tasks get a random priority
    }
	task[MAX_TASK_NUM-1].next=&task[0];
    printk(KERN_NOTICE "\n\n\n\n\n\n                system begin :>>>process 0 running!!!<<<\n\n");
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
asm volatile(
     "movl %1,%%esp\n\t" /* set task[pid].thread.sp to esp */
     "pushl %1\n\t" /* push ebp */
     "pushl %0\n\t" /* push task[pid].thread.ip */
     "ret\n\t" /* pop task[pid].thread.ip to eip */
     "popl %%ebp\n\t"
     :
     : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
);
}
void my_process(void)
{
    int i = 0;
    while(1)
    {
        i++;
        if(i%10000000 == 0)
        {
	  
            if(my_need_sched == 1)
            {
                my_need_sched = 0;
		sand_priority();
	   	my_schedule();  
		
	   }
        }
    }
}//end of my_process

//produce a random priority to a task
unsigned long get_rand(max)
{
	unsigned long a;
	unsigned long umax;
	umax=(unsigned long)max;
 	get_random_bytes(&a, sizeof(unsigned long ));
	a=(a+umax)%umax;
	return a;
}
```
main.c首先生成了10个PCB表，并为他们生成了随机的优先级，最后使用汇编语言实现了现场保护，保存了当前进程的ip和sp。

最后是myinterrupt.c
```
#include "mypcb.h"

#define CREATE_TRACE_POINTS
#include <trace/events/timer.h>

extern tPCB task[MAX_TASK_NUM];
extern tPCB * my_current_task;
extern volatile int my_need_sched;
volatile int time_count = 0;

/*
* Called by timer interrupt.
* it runs in the name of current running process,
* so it use kernel stack of current running process
*/
void my_timer_handler(void)
{
#if 1
    // make sure need schedule after system circle 2000 times.
    if(time_count%2000 == 0 && my_need_sched != 1)
    {
        my_need_sched = 1;
	//time_count=0;
    }
    time_count ++ ;
#endif
    return;
}

void all_task_print(void);

tPCB * get_next(void)
{
	int pid,i;
	tPCB * point=NULL;
	tPCB * hig_pri=NULL;//points to the the hightest task
	all_task_print();
	hig_pri=my_current_task;
	for(i=0;i<MAX_TASK_NUM;i++)
		if(task[i].priority<hig_pri->priority)	
			hig_pri=&task[i];
	printk("                higst process is:%d priority is:%d\n",hig_pri->pid,hig_pri->priority);
	return hig_pri;

}//end of priority_schedule

void my_schedule(void)
{
    tPCB * next;
    tPCB * prev;
    // if there no task running or only a task ,it shouldn't need schedule
    if(my_current_task == NULL
        || my_current_task->next == NULL)
    {
	printk(KERN_NOTICE "                time out!!!,but no more than 2 task,need not schedule\n");
     return;
    }
    /* schedule */

    next = get_next();
    prev = my_current_task;
    printk(KERN_NOTICE "                the next task is %d priority is %u\n",next->pid,next->priority);
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {//save current scene
     /* switch to next process */
     asm volatile(	
         "pushl %%ebp\n\t" /* save ebp */
         "movl %%esp,%0\n\t" /* save esp */
         "movl %2,%%esp\n\t" /* restore esp */
         "movl $1f,%1\n\t" /* save eip */	
         "pushl %3\n\t"
         "ret\n\t" /* restore eip */
         "1:\t" /* next process start here */
         "popl %%ebp\n\t"
         : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
         : "m" (next->thread.sp),"m" (next->thread.ip)
     );
     my_current_task = next;//switch to the next task
    printk(KERN_NOTICE "                switch from %d process to %d process\n                >>>process %d running!!!<<<\n\n",prev->pid,next->pid,next->pid);

  }
    else
    {
        next->state = 0;
        my_current_task = next;
    printk(KERN_NOTICE "                switch from %d process to %d process\n                >>>process %d running!!!<<<\n\n\n",prev->pid,next->pid,next->pid);

     /* switch to new process */
     asm volatile(	
         "pushl %%ebp\n\t" /* save ebp */
         "movl %%esp,%0\n\t" /* save esp */
         "movl %2,%%esp\n\t" /* restore esp */
         "movl %2,%%ebp\n\t" /* restore ebp */
         "movl $1f,%1\n\t" /* save eip */	
         "pushl %3\n\t"
         "ret\n\t" /* restore eip */
         : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
         : "m" (next->thread.sp),"m" (next->thread.ip)
     );
    }
    return;	
}//end of my_schedule

void all_task_print(void)
{
	int i,cnum=62;//
	printk(KERN_NOTICE "\n                current task is:%d   all task in OS are:\n",my_current_task->pid);

	printk("        ");
	for(i=0;i<cnum;i++)
		printk("-");
	printk("\n        |  process:");
	for(i=0;i< MAX_TASK_NUM;i++)
		printk("| %2d ",i);
	printk("|\n        | priority:");
	for(i=0;i<MAX_TASK_NUM;i++)
		printk("| %2d ",task[i].priority);

	printk("|\n        ");
	for(i=0;i<cnum;i++)
		printk("-");
	printk("\n");
}
```
myinterrupt.c实现了时间片轮转切换进程。

### 实验总结
