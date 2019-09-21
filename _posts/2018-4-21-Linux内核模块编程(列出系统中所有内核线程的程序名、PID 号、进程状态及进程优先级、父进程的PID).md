###(1) 设计一个模块,要求列出系统中所有内核线程的程序名、PID 号、进程状态及进程优先级、父进程的PID。

1.首先，我们开始编写模块代码，这是Linux内核编程的核心代码，代码如下：

```
#include <linux/init.h>
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/sched.h>
#include <linux/init_task.h>
// 初始化函数
static int hello_init(void)
{
    struct task_struct *p;
    p = NULL;
    p = &init_task;
    printk(KERN_ALERT"名称\t进程号\t状态\t优先级\t父进程号\t");
    for_each_process(p)
    {
        if(p->mm == NULL){ //内核线程的mm成员为空
            printk(KERN_ALERT"%s\t%d\t%ld\t%d\n",p->comm,p->pid, p->state,p->normal_prio,p->parent->pid);
        }
    }
    return 0;
}
// 清理函数
static void hello_exit(void)
{
    printk(KERN_ALERT"goodbye!\n");
}

// 函数注册
module_init(hello_init);  
module_exit(hello_exit);  

// 模块许可申明
MODULE_LICENSE("GPL");
```

2.头文件的声明，module.h包含了大量加载模块所需要的函数和符号的定义；init.h包含了模块初始化和清理函数的定义。如果模块加载时允许用户传递参数，模块还应该包含moduleparam.h头文件。

3.接下来我们需要初始化函数，task_struct 结构定义在/usr/src/linux4.4.19/include/linux/sched.h 中，所以也需要在头文件加入这个库，而这个结构也包含了我们需要使用的PID之类的值，可以直接调用。当然了，这两篇文章可以更好理解这个结构：

> https://blog.csdn.net/npy_lp/article/details/7292563
>
> https://blog.csdn.net/npy_lp/article/details/7335187

接下来，我们需要对我们定义的结构p初始化指针，否则会产生也指针，文件也会报错，因此，我们需要用到init_task，并且在头文件加入init_task.h库。

4.然后，我们用for_each_process遍历我们的每一个进程，并输出我们需要得到的进程信息。

5.最后是函数注册已经模块许可声明，这是必须加入的。

6.编译Makefile

```
obj-m:=module1.o    
KDIR:= /lib/modules/$(shell uname -r)/build
PWD:= $(shell pwd) 

default:
	$(MAKE) -C $(KDIR) M=$(PWD) modules  
clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

7.make后使用insmod加入模块

8.dmesg显示使用情况，我的如图所示。

![VirtualBox_Ubuntu_21_04_2018_10_16_25](/img/VirtualBox_Ubuntu_21_04_2018_10_16_25.png)

## (2) 设计一个带参数的模块,其参数为某个进程的 PID 号,该模块的功能是列出该进程的家族信息,包括父进程、兄弟进程和子进程的程序名、PID 号。

1.我的代码如下：

```
#include<linux/init.h>
#include<linux/module.h>
#include<linux/kernel.h>
#include <linux/sched.h>
#include <linux/moduleparam.h>

static pid_t pid=1;
module_param(pid,int,0644);

static int hello_init(void)
{
    struct task_struct *p;
    struct list_head *pp;
    struct task_struct *psibling;

    // 当前进程的 PID
    p = pid_task(find_vpid(pid), PIDTYPE_PID);
    printk("me: %d %s\n", p->pid, p->comm);

    // 父进程
    if(p->parent == NULL) {
        printk("No Parent\n");
    }
    else {
        printk("Parent: %d %s\n", p->parent->pid, p->parent->comm);
    }

    // 兄弟进程
    list_for_each(pp, &p->parent->children)
    {
        psibling = list_entry(pp, struct task_struct, sibling);
        printk("sibling %d %s \n", psibling->pid, psibling->comm);
    }

    // 子进程
    list_for_each(pp, &p->children)
    {
        psibling = list_entry(pp, struct task_struct, sibling);
        printk("children %d %s \n", psibling->pid, psibling->comm);
    }
    
    return 0;
}

static void hello_exit(void)
{
    printk(KERN_ALERT"goodbye!\n");
}

module_init(hello_init);
module_exit(hello_exit);
                
MODULE_LICENSE("GPL");
```

2.查看当前进程的父进程

```
if(p->parent == NULL) {
    printk("No Parent\n");
}
else {
    printk("Parent: %d %s\n", p->parent->pid, p->parent->comm);
}
```

3.查找当前进程的兄弟进程

```
list_for_each(pp, &p->parent->children)
{
     psibling = list_entry(pp, struct task_struct, sibling);
     printk("sibling %d %s \n", psibling->pid, psibling->comm);
}
```

4.查找当前进程的子进程

```
list_for_each(pp, &p->children)
{
    psibling = list_entry(pp, struct task_struct, sibling);
    printk("children %d %s \n", psibling->pid, psibling->comm);
}
```

5.接下来的过程与第一个实验基本一致，下图是我的结果：

![VirtualBox_Ubuntu_21_04_2018_10_24_17](/img/VirtualBox_Ubuntu_21_04_2018_10_24_17.png)

剑魄未改
