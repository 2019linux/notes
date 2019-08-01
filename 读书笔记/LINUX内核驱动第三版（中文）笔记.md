# LINUX内核驱动第三版（中文）笔记

### 第 2 章 建立和运行模块

#include <linux/sched.h> 

​	最重要的头文件中的一个. 这个文件包含很多驱动使用的内核 API 的定义, 包括睡眠函数和许多变量声明.

```c
struct task_struct *current; 
```

```
current->pid 
current->comm 
```

进程 ID 和 当前进程的命令名.

```c
 printk(KERN_INFO "The process is \"%s\" (pid %i)\n", current->comm, current->pid);
```



```c
#include <linux/moduleparam.h>  

 module_param(variable, type, perm);  
```

 宏定义, 创建模块参数, 可以被用户在模块加载时调整( 或者在启动时间, 对于内嵌代码). 类型可以是 bool, charp, int,  invbool, short, ushort, uint, ulong, 或者 intarray.



```c
EXPORT_SYMBOL (symbol);  

EXPORT_SYMBOL_GPL (symbol);  
```

 宏定义, 用来输出一个符号给内核. 第 2 种形式输出没有版本信息, 第 3 种限制输出给 GPL 许可的模块.

```c
MODULE_AUTHOR(author);  
MODULE_DESCRIPTION(description);  
MODULE_VERSION(version_string);  
MODULE_DEVICE_TABLE(table_info);    
MODULE_ALIAS(alternate_name);  
```

 放置文档在目标文件的模块中.

### 第 3 章 字符驱动

#### 3.2. 主次编号

ls -l 的输出的第一列的"c"标识. 块设备也出现在 /dev 中, 但是它们由"b"标识

传统上, 主编号标识设备相连的驱动.

次编号被内核用来决定引用哪个设备。 内核自己几乎不知道次编号的任何事情, 除了它们指向你的驱动实现的设备.

##### 3.2.1. 设备编号的内部

dev_t 是 32 位的量, 12 位用作主编号, 20 位用作次编号.

```c
MAJOR(dev_t dev); 
MINOR(dev_t dev);
```

相反, 如果你有主次编号, 需要将其转换为一个 dev_t, 使用:

```c
MKDEV(int major, int minor); 
```

##### 3.2.2. 分配和释放设备编号

​	<linux/fs.h>

```c
int register_chrdev_region(dev_t first, unsigned int count, char *name);
```

first 是你要分配的起始设备编号。

count 是你请求的连续设备编号的总数。

name 是应当连接到这个编号范围的设备的名子。

它会出现在 /proc/devices 和 sysfs 中



动态为你分配一个主编号

```c
int alloc_chrdev_region(dev_t *dev, unsigned int firstminor, unsigned int count, char *name);
```

dev 是一个只输出的参数, 它在函数成功完成时持有你的分配范围的第一个数. fisetminor 应当是请求的第一个要用的次编号; 它常常是 0. count 
和 name 参数如同给 request_chrdev_region 的一样.



设备编号的释放使用:

```c
void unregister_chrdev_region(dev_t first, unsigned int count); 
```

##### 3.2.3. 主编号的动态分配

动态分配的缺点是你无法提前创建设备节点, 因为分配给你的模块的主编号会变化. 对于驱动的正常使用, 这不是问题, 因为一旦编号分配了, 你可从 
/proc/devices 中读取它

#### 3.3. 一些重要数据结构

##### 3.3.1. 文件操作

file_operation 结构是一个字符驱动如何建立这个连接. 这个结构, 定义在 <linux/fs.h>,是一个函数指针的集合. 

- struct module *owner  

   第一个 file_operations 成员根本不是一个操作; 它是一个指向拥有这个结构的模块的指针. 这个成员用来在它的操作还在被使用时阻止模块被卸载.  几乎所有时间中, 它被简单初始化为 THIS_MODULE, 一个在 <linux/module.h> 中定义的宏.

- loff_t (*llseek) (struct file *, loff_t,  int);  

   llseek 方法用作改变文件中的当前读/写位置, 并且新位置作为(正的)返回值. loff_t 参数是一个"long offset", 并且就算在  32位平台上也至少 64 位宽. 错误由一个负返回值指示. 如果这个函数指针是 NULL, seek 调用会以潜在地无法预知的方式修改 file  结构中的位置计数器( 在"file 结构" 一节中描述).

- ssize_t (*read) (struct file *, char __user *,  size_t, loff_t *);  

   用来从设备中获取数据. 在这个位置的一个空指针导致 read 系统调用以 -EINVAL("Invalid argument") 失败.  一个非负返回值代表了成功读取的字节数( 返回值是一个 "signed size" 类型, 常常是目标平台本地的整数类型).

- ssize_t (*aio_read)(struct kiocb *, char __user *,  size_t, loff_t);  

   初始化一个异步读 -- 可能在函数返回前不结束的读操作. 如果这个方法是 NULL, 所有的操作会由 read 代替进行(同步地).

- ssize_t (*write) (struct file *, const char __user *,  size_t, loff_t *);  

   发送数据给设备. 如果 NULL, -EINVAL 返回给调用 write 系统调用的程序. 如果非负, 返回值代表成功写的字节数.

- ssize_t (*aio_write)(struct kiocb *, const char  __user *, size_t, loff_t *);  

   初始化设备上的一个异步写.

- int (*readdir) (struct file *, void *,  filldir_t);  

   对于设备文件这个成员应当为 NULL; 它用来读取目录, 并且仅对文件系统有用.

- unsigned int (*poll) (struct file *, struct  poll_table_struct *);  

   poll 方法是 3 个系统调用的后端: poll, epoll, 和 select, 都用作查询对一个或多个文件描述符的读或写是否会阻塞. poll  方法应当返回一个位掩码指示是否非阻塞的读或写是可能的, 并且, 可能地, 提供给内核信息用来使调用进程睡眠直到 I/O 变为可能. 如果一个驱动的 poll  方法为 NULL, 设备假定为不阻塞地可读可写.

- int (*ioctl) (struct inode *, struct file *, unsigned  int, unsigned long);  

   ioctl 系统调用提供了发出设备特定命令的方法(例如格式化软盘的一个磁道, 这不是读也不是写). 另外, 几个 ioctl 命令被内核识别而不必引用  fops 表. 如果设备不提供 ioctl 方法, 对于任何未事先定义的请求(-ENOTTY, "设备无这样的 ioctl"), 系统调用返回一个错误.  

- int (*mmap) (struct file *, struct vm_area_struct  *);  

   mmap 用来请求将设备内存映射到进程的地址空间. 如果这个方法是 NULL, mmap 系统调用返回 -ENODEV.

- int (*open) (struct inode *, struct file  *);  

   尽管这常常是对设备文件进行的第一个操作, 不要求驱动声明一个对应的方法. 如果这个项是 NULL, 设备打开一直成功,  但是你的驱动不会得到通知.

- int (*flush) (struct file *);  

   flush 操作在进程关闭它的设备文件描述符的拷贝时调用; 它应当执行(并且等待)设备的任何未完成的操作. 这个必须不要和用户查询请求的 fsync  操作混淆了. 当前, flush 在很少驱动中使用; SCSI 磁带驱动使用它, 例如, 为确保所有写的数据在设备关闭前写到磁带上. 如果 flush 为  NULL, 内核简单地忽略用户应用程序的请求.

- int (*release) (struct inode *, struct file  *);  

   在文件结构被释放时引用这个操作. 如同 open, release 可以为 NULL.

- int (*fsync) (struct file *, struct dentry *,  int);  

   这个方法是 fsync 系统调用的后端, 用户调用来刷新任何挂着的数据. 如果这个指针是 NULL, 系统调用返回 -EINVAL

- int (*aio_fsync)(struct kiocb *, int);  

   这是 fsync 方法的异步版本.

##### 3.3.2. 文件结构

struct file, 定义于 <linux/fs.h>是一个内核结构, 从不出现在用户程序中.

它由内核在 open 时创建, 并传递给在文件上操作的任何函数, 直到最后的关闭. 在文件的所有实例都关闭后, 内核释放这个数据结构

在内核源码中, struct file 的指针常常称为 file 或者 filp("file pointer"). 我们将一直称这个指针为 filp 
以避免和结构自身混淆. 因此, file 指的是结构, 而 filp 是结构指针.

##### 3.3.3. inode 结构

inode 结构由内核在内部用来表示文件. 因此, 它和代表打开文件描述符的文件结构是不同的. 可能有代表单个文件的多个打开描述符的许多文件结构, 
但是它们都指向一个单个 inode 结构.

- dev_t i_rdev;  

   对于代表设备文件的节点, 这个成员包含实际的设备编号.

- struct cdev *i_cdev;  

   struct cdev 是内核的内部结构, 代表字符设备; 这个成员包含一个指针, 指向这个结构, 当节点指的是一个字符设备文件时.

#### 3.4. 字符设备注册

内核在内部使用类型 struct cdev 的结构来代表字符设备.

 <linux/cdev.h>, 这个结构和它的关联帮助函数定义在这里.

有 2 种方法来分配和初始化一个这些结构. 如果你想在运行时获得一个独立的 cdev 结构, 你可以为此使用这样的代码:

```c
struct cdev *my_cdev = cdev_alloc();
my_cdev->ops = &my_fops;
```

但是, 偶尔你会想将 cdev 结构嵌入一个你自己的设备特定的结构; scull 这样做了. 在这种情况下, 你应当初始化你已经分配的结构, 
使用:

```c
void cdev_init(struct cdev *cdev, struct file_operations *fops);
```

任一方法, 有一个其他的 struct cdev 成员你需要初始化. 象 file_operations 结构, struct cdev  有一个拥有者成员, 应当设置为 THIS_MODULE. 一旦 cdev 结构建立, 最后的步骤是把它告诉内核, 调用:

```c
int cdev_add(struct cdev *dev, dev_t num, unsigned int count);
```

 cdev_add 是有几个重要事情要记住. 第一个是这个调用可能失败. 如果它返回一个负的错误码, 你的设备没有增加到系统中. 它几乎会一直成功.

为从系统去除一个字符设备, 调用:

```c
void cdev_del(struct cdev *dev);
```

#### 3.5. open 和 release

##### 3.5.1. open 方法

open 应当进行下面的工作:

- 检查设备特定的错误(例如设备没准备好, 或者类似的硬件错误
- 如果它第一次打开, 初始化设备
- 如果需要, 更新 f_op 指针.
- 分配并填充要放进 filp->private_data 的任何数据结构

```c
int (*open)(struct inode *inode, struct file *filp);
```

inode 参数有我们需要的信息,以它的 i_cdev 成员的形式, 里面包含我们之前建立的 cdev 结构. 唯一的问题是通常我们不想要 cdev 
结构本身, 我们需要的是包含 cdev 结构的 scull_dev 结构. C 语言使程序员玩弄各种技巧来做这种转换; 但是, 这种技巧编程是易出错的, 
并且导致别人难于阅读和理解代码. 幸运的是, 在这种情况下, 内核 hacker 已经为我们实现了这个技巧, 以 container_of 宏的形式, 在 
<linux/kernel.h> 中定义:

```c
container_of(pointer, container_type, container_field); 
```

这个宏使用一个指向 container_field 类型的成员的指针, 它在一个 container_type 类型的结构中,  并且返回一个指针指向包含结构. 在 scull_open, 这个宏用来找到适当的设备结构:

```c
struct scull_dev *dev; /* device information */ 
dev = container_of(inode->i_cdev, struct scull_dev, cdev);
filp->private_data = dev; /* for other methods */
```

```c
int scull_open(struct inode *inode, struct file *filp)
{
        struct scull_dev *dev; /* device information */
        dev = container_of(inode->i_cdev, struct scull_dev, cdev);
        filp->private_data = dev; /* for other methods */

        /* now trim to 0 the length of the device if open was write-only */
        if ( (filp->f_flags & O_ACCMODE) == O_WRONLY)
        {
                scull_trim(dev); /* ignore errors */
        }
        return 0; /* success */
}
```

##### 3.5.2. release 方法

```c
int scull_release(struct inode *inode, struct file *filp)
{
 return 0;
}
```

你可能想知道当一个设备文件关闭次数超过它被打开的次数会发生什么. 毕竟, dup 和 fork 系统调用不调用 open 来创建打开文件的拷贝; 
每个拷贝接着在程序终止时被关闭. 例如, 大部分程序不打开它们的 stdin 文件(或设备), 但是它们都以关闭它结束. 
当一个打开的设备文件已经真正被关闭时驱动如何知道?

答案简单: 不是每个 close 系统调用引起调用 release 方法. 只有真正释放设备数据结构的调用会调用这个方法 -- 因此得名.  内核维持一个文件结构被使用多少次的计数. fork 和 dup 都不创建新文件(只有 open 这样); 它们只递增正存在的结构中的计数. close  系统调用仅在文件结构计数掉到 0 时执行 release 方法, 这在结构被销毁时发生. release 方法和 close  系统调用之间的这种关系保证了你的驱动一次 open 只看到一次 release.

注意, flush 方法在每次应用程序调用 close 时都被调用. 但是, 很少驱动实现 flush, 因为常常在 close 时没有什么要做,  除非调用 release.

如你会想到的, 前面的讨论即便是应用程序没有明显地关闭它打开的文件也适用: 内核在进程 exit 时自动关闭了任何文件, 通过在内部使用 close  系统调用.

#### 3.6. scull 的内存使用

scull 驱动引入 2 个核心函数来管理 Linux 内核中的内存. 这些函数, 定义在 <linux/slab.h>, 是:

```c
void *kmalloc(size_t size, int flags); 
void kfree(void *ptr);
```

在 scull, 每个设备是一个指针链表, 每个都指向一个 scull_dev 结构. 每个这样的结构, 缺省地, 指向最多 4 兆字节, 
通过一个中间指针数组. 发行代码使用一个 1000 个指针的数组指向每个 4000 字节的区域. 我们称每个内存区域为一个量子, 数组(或者它的长度) 
为一个量子集.

```c
struct scull_qset {
 void **data;
 struct scull_qset *next; 
}; 
```

下一个代码片段展示了实际中 struct scull_dev 和 struct scull_qset 是如何被用来持有数据的. sucll_trim 
函数负责释放整个数据区, 由 scull_open 在文件为写而打开时调用. 它简单地遍历列表并且释放它发现的任何量子和量子集.

```c
int scull_trim(struct scull_dev *dev)
{
        struct scull_qset *next, *dptr;
        int qset = dev->qset; /* "dev" is not-null */
        int i;
        for (dptr = dev->data; dptr; dptr = next)
        { /* all the list items */
                if (dptr->data) {
                        for (i = 0; i < qset; i++)
                                kfree(dptr->data[i]);
                        kfree(dptr->data);
                        dptr->data = NULL;
                }

                next = dptr->next;
                kfree(dptr);
        }
        dev->size = 0;
        dev->quantum = scull_quantum;
        dev->qset = scull_qset;
        dev->data = NULL;
        return 0;
}
```

scull_trim 也用在模块清理函数中, 来归还 scull 使用的内存给系统.

#### 3.7. 读和写

```c
ssize_t read(struct file *filp, char __user *buff, size_t count, loff_t *offp);
ssize_t write(struct file *filp, const char __user *buff, size_t count, loff_t *offp);
```

让我们重复一下, read 和 write 方法的 buff 参数是用户空间指针. 因此, 它不能被内核代码直接解引用. 这个限制有几个理由:

- 依赖于你的驱动运行的体系, 以及内核被如何配置的, 用户空间指针当运行于内核模式可能根本是无效的. 可能没有那个地址的映射,  或者它可能指向一些其他的随机数据.
- 就算这个指针在内核空间是同样的东西, 用户空间内存是分页的, 在做系统调用时这个内存可能没有在 RAM 中. 试图直接引用用户空间内存可能产生一个页面错,  这是内核代码不允许做的事情. 结果可能是一个"oops", 导致进行系统调用的进程死亡.
- 置疑中的指针由一个用户程序提供, 它可能是错误的或者恶意的. 如果你的驱动盲目地解引用一个用户提供的指针,  它提供了一个打开的门路使用户空间程序存取或覆盖系统任何地方的内存. 如果你不想负责你的用户的系统的安全危险,  你就不能直接解引用用户空间指针.

scull 中的读写代码需要拷贝一整段数据到或者从用户地址空间. 这个能力由下列内核函数提供, 它们拷贝一个任意的字节数组, 
并且位于大部分读写实现的核心中.

```c
unsigned long copy_to_user(void __user *to,const void *from,unsigned long count); 
unsigned long copy_from_user(void *to,const void __user *from,unsigned long count); 
```

##### 3.7.1. read 方法

read 的返回值由调用的应用程序解释:

- 如果这个值等于传递给 read 系统调用的 count 参数, 请求的字节数已经被传送. 这是最好的情况.
- 如果是正数, 但是小于 count, 只有部分数据被传送. 这可能由于几个原因, 依赖于设备. 常常, 应用程序重新试着读取. 例如, 如果你使用  fread 函数来读取, 库函数重新发出系统调用直到请求的数据传送完成.
- 如果值为 0, 到达了文件末尾(没有读取数据).
- 一个负值表示有一个错误. 这个值指出了什么错误, 根据 <linux/errno.h>. 出错的典型返回值包括 -EINTR(  被打断的系统调用) 或者 -EFAULT( 坏地址 ).



```c
ssize_t scull_read(struct file *filp, char __user *buf, size_t count, loff_t *f_pos)
{
        struct scull_dev *dev = filp->private_data;
        struct scull_qset *dptr; /* the first listitem */
        int quantum = dev->quantum, qset = dev->qset;
        int itemsize = quantum * qset; /* how many bytes in the listitem */
        int item, s_pos, q_pos, rest;
        ssize_t retval = 0;

        if (down_interruptible(&dev->sem))
                return -ERESTARTSYS;
        if (*f_pos >= dev->size)
                goto out;
        if (*f_pos + count > dev->size)
                count = dev->size - *f_pos;

        /* find listitem, qset index, and offset in the quantum */
        item = (long)*f_pos / itemsize;
        rest = (long)*f_pos % itemsize;
        s_pos = rest / quantum;
        q_pos = rest % quantum;

        /* follow the list up to the right position (defined elsewhere) */
        dptr = scull_follow(dev, item);
        if (dptr == NULL || !dptr->data || ! dptr->data[s_pos])
                goto out; /* don't fill holes */

        /* read only up to the end of this quantum */
        if (count > quantum - q_pos)
                count = quantum - q_pos;

        if (copy_to_user(buf, dptr->data[s_pos] + q_pos, count))
        {
                retval = -EFAULT;
                goto out;

        }
        *f_pos += count;
        retval = count;

out:
        up(&dev->sem);
        return retval;
}
```

##### 3.7.2. write 方法

write, 象 read, 可以传送少于要求的数据, 根据返回值的下列规则:

- 如果值等于 count, 要求的字节数已被传送.
- 如果正值, 但是小于 count, 只有部分数据被传送. 程序最可能重试写入剩下的数据.
- 如果值为 0, 什么没有写. 这个结果不是一个错误, 没有理由返回一个错误码. 再一次, 标准库重试写调用. 我们将在第 6 章查看这种情况的确切含义,  那里介绍了阻塞.
- 一个负值表示发生一个错误; 如同对于读, 有效的错误值是定义于 <linux/errno.h>中.

不幸的是, 仍然可能有发出错误消息的不当行为程序, 它在进行了部分传送时终止. 这是因为一些程序员习惯看写调用要么完全失败要么完全成功,  这实际上是大部分时间的情况, 应当也被设备支持. scull 实现的这个限制可以修改, 但是我们不想使代码不必要地复杂.

write 的 scull 代码一次处理单个量子, 如 read 方法做的:

```c
ssize_t scull_write(struct file *filp, const char __user *buf, size_t count, loff_t *f_pos)
{
        struct scull_dev *dev = filp->private_data;
        struct scull_qset *dptr;
        int quantum = dev->quantum, qset = dev->qset;
        int itemsize = quantum * qset;
        int item, s_pos, q_pos, rest;
        ssize_t retval = -ENOMEM; /* value used in "goto out" statements */
        if (down_interruptible(&dev->sem))
                return -ERESTARTSYS;

        /* find listitem, qset index and offset in the quantum */
        item = (long)*f_pos / itemsize;
        rest = (long)*f_pos % itemsize;
        s_pos = rest / quantum;
        q_pos = rest % quantum;
        /* follow the list up to the right position */
        dptr = scull_follow(dev, item);
        if (dptr == NULL)
                goto out;
        if (!dptr->data)
        {
                dptr->data = kmalloc(qset * sizeof(char *), GFP_KERNEL);
                if (!dptr->data)
                        goto out;
                memset(dptr->data, 0, qset * sizeof(char *));
        }
        if (!dptr->data[s_pos])
        {
                dptr->data[s_pos] = kmalloc(quantum, GFP_KERNEL);
                if (!dptr->data[s_pos])

                        goto out;
        }
        /* write only up to the end of this quantum */
        if (count > quantum - q_pos)

                count = quantum - q_pos;
        if (copy_from_user(dptr->data[s_pos]+q_pos, buf, count))
        {
                retval = -EFAULT;
                goto out;

        }
        *f_pos += count;
        retval = count;

        /* update the size */
        if (dev->size < *f_pos)
                dev->size = *f_pos;

out:
        up(&dev->sem);
        return retval;

}
```

##### 3.7.3. readv 和 writev

矢量操作的原型是:

```c
ssize_t (*readv) (struct file *filp, const struct iovec *iov, unsigned long count, loff_t *ppos);
ssize_t (*writev) (struct file *filp, const struct iovec *iov, unsigned long count, loff_t *ppos);
```

##### 3.8. 使用新设备

一旦你装备好刚刚描述的 4 个方法, 驱动可以编译并测试了; 它保留了你写给它的任何数据, 直到你用新数据覆盖它. 这个设备表现如一个数据缓存器,  它的长度仅仅受限于可用的真实 RAM 的数量. 你可试着使用 cp, dd, 以及 输入/输出重定向来测试这个驱动.

free 命令可用来看空闲内存的数量如何缩短和扩张的, 依据有多少数据写入 scull.

为对一次读写一个量子有更多信心, 你可增加一个 printk 在驱动的适当位置, 并且观察当应用程序读写大块数据中发生了什么. 可选地, 使用  strace 工具来监视程序发出的系统调用以及它们的返回值. 跟踪一个 cp 或者一个 ls -l > /dev/scull0 展示了量子化的读和写.  监视(以及调试)技术在第 4 章详细介绍.

##### 3.9. 快速参考

- #include <linux/types.h>  

   

- dev_t  

   dev_t 是用来在内核里代表设备号的类型

```c
int MAJOR(dev_t dev);  
int MINOR(dev_t dev);
```

从设备编号中抽取主次编号的宏.

```c
dev_t MKDEV(unsigned int major, unsigned int minor); 
```

从主次编号来建立 dev_t 数据项的宏定义.

#include <linux/fs.h> 

"文件系统"头文件是编写设备驱动需要的头文件. 许多重要的函数和数据结构在此定义.

```c
int register_chrdev_region(dev_t first, unsigned int  count, char *name)  
int alloc_chrdev_region(dev_t *dev, unsigned int  firstminor, unsigned int count, char *name)  
void unregister_chrdev_region(dev_t first, unsigned  int count); 
```

struct file_operations;  

struct file;  

struct inode;

大部分设备驱动使用的 3 个重要数据结构. file_operations 结构持有一个字符驱动的方法; struct file 代表一个打开的文件, 
struct inode 代表磁盘上的一个文件

```c
#include <linux/cdev.h>  
struct cdev *cdev_alloc(void);  
void cdev_init(struct cdev *dev, struct  file_operations *fops);  
int cdev_add(struct cdev *dev, dev_t num, unsigned  int count);  
void cdev_del(struct cdev *dev);
```

cdev 结构管理的函数, 它代表内核中的字符设备.

```c
#include <linux/kernel.h>  
container_of(pointer, type, field);  
```

```c
#include <asm/uaccess.h> 
unsigned long copy_from_user (void *to, const void  *from, unsigned long count);  
unsigned long copy_to_user (void *to, const void  *from, unsigned long count); 
```

在用户空间和内核空间拷贝数据.

### 第 4 章 调试技术

#### 4.1. 内核中的调试支持

#### 4.2. 用打印调试

##### 4.2.1. printk

- KERN_EMERG  用于紧急消息, 常常是那些崩溃前的消息.
- KERN_ALERT 需要立刻动作的情形.
- KERN_CRIT 严重情况, 常常与严重的硬件或者软件失效有关.
- KERN_ERR 用来报告错误情况; 设备驱动常常使用 KERN_ERR 来报告硬件故障.
- KERN_WARNING  有问题的情况的警告, 这些情况自己不会引起系统的严重问题.
- KERN_NOTICE 正常情况, 但是仍然值得注意. 在这个级别一些安全相关的情况会报告.
- KERN_INFO 信息型消息. 在这个级别, 很多驱动在启动时打印它们发现的硬件的信息.
- KERN_DEBUG 用作调试消息.

每个字串( 在宏定义扩展里 )代表一个在角括号中的整数. 整数的范围从 0 到 7, 越小的数表示越大的优先级.

```c
 # echo 8 > /proc/sys/kernel/printk 
```

##### 4.2.2. 重定向控制台消息

为了选择一个不同地虚拟终端来接收消息, 你可对任何控制台设备调用 ioctl(TIOCLINUX). 

```c
int main(int argc, char **argv)
{
    char bytes[2] = {11,0}; /* 11 is the TIOCLINUX cmd number */
    if (argc==2) bytes[1] = atoi(argv[1]); /* the chosen console */
    else {

        fprintf(stderr, "%s: need a single arg\n",argv[0]); exit(1); 
    } 
    if (ioctl(STDIN_FILENO, TIOCLINUX, bytes)<0) { /* use stdin */
        fprintf(stderr,"%s: ioctl(stdin, TIOCLINUX): %s\n",
                argv[0], strerror(errno));
        exit(1);
    }
    exit(0);
}
```

##### 4.2.3. 消息是如何记录的

printk 函数将消息写入一个 __LOG_BUF_LEN 字节长的环形缓存, 长度值从 4 KB 到 1 MB, 由配置内核时选择.

##### 4.2.4. 打开和关闭消息

```c
#undef PDEBUG /* undef it, just in case */
#ifdef SCULL_DEBUG
# ifdef __KERNEL__

/* This one if debugging is on, and kernel space */
# define PDEBUG(fmt, args...) printk( KERN_DEBUG "scull: " fmt, ## args)
# else

/* This one for user space */
# define PDEBUG(fmt, args...) fprintf(stderr, fmt, ## args)
# endif
#else
# define PDEBUG(fmt, args...) /* not debugging: nothing */
#endif

#undef PDEBUGG #define PDEBUGG(fmt, args...) /* nothing: it's a placeholder */
```



##### 4.2.5. 速率限制

#### 4.3. 用查询来调试

##### 4.3.1. 使用 /proc 文件系统

###### 4.3.1.1. 在 /proc 里实现文件

所有使用 /proc 的模块应当包含 <linux/proc_fs.h> 来定义正确的函数.

read_proc 的方法:

```c
int (*read_proc)(char *page, char **start, off_t offset, int count, int *eof, void *data);
```

###### 4.3.1.3. 创建你的 /proc 文件

使用一个 creat_proc_read_entry 调用:

```c
struct proc_dir_entry *create_proc_read_entry(const char *name,mode_t mode, struct proc_dir_entry *base, read_proc_t *read_proc, void *data); 
```

```c
create_proc_read_entry("scullmem", 0 /* default mode */,
                       NULL /* parent dir */, scull_read_procmem,
                       NULL /* client data */);
```

```c
remove_proc_entry("scullmem", NULL /* parent dir */); 
```

###### 4.3.1.4. seq_file 接口

#### 4.5. 调试系统故障

##### 4.5.1. oops 消息

#### 4.6. 调试器和相关工具

##### 4.6.1. 使用 gdb

gdb 对于看系统内部是非常有用. 在这个级别精通调试器的使用要求对 gdb 命令有信心, 需要理解目标平台的汇编代码,  以及对应源码和优化的汇编码的能力.

调试器必须把内核作为一个应用程序来调用. 除了指定内核映象的文件名之外, 你需要在命令行提供一个核心文件的名子. 对于一个运行的内核,  核心文件是内核核心映象, /proc/kcore. 一个典型的 gdb 调用看来如下:

```
gdb /usr/src/linux/vmlinux /proc/kcore 
```

第一个参数是非压缩的 ELF 内核可执行文件的名子, 不是 zImage 或者 bzImage 或者给启动环境特别编译的任何东东.

gdb 命令行的第二个参数是核心文件的名子. 如同任何 /proc 中的文件, /proc/kcore 是在被读的时候产生的. 当 read 系统调用在  /proc 文件系统中执行时, 它映射到一个数据产生函数,而不是一个数据获取函数; 我们已经在本章"使用 /proc 文件系统"一节中利用了这个特点.  kcore 用来代表内核"可执行文件", 以一个核心文件的形式; 它是一个巨大的文件, 因为他代表整个的内核地址空间, 对应于所有的物理内存. 从 gdb 中,  你可查看内核变量,通过发出标准 gdb 命令. 例如, p jiffies 打印时钟的从启动到当前时间的嘀哒数.

当你从gdb打印数据, 内核仍然在运行, 各种数据项在不同时间有不同的值; 然而, gdb 通过缓存已经读取的数据来优化对核心文件的存取.  如果你试图再次查看 jiffies 变量, 你会得到和以前相同的答案. 缓存值来避免额外的磁盘存取对传统核心文件是正确的做法,  但是在使用一个"动态"核心映象时就不方便. 解决方法是任何时候你需要刷新 gdb 缓存时发出命令 core-file /proc/kcore;  调试器准备好使用新的核心文件并且丢弃任何旧信息. 然而, 你不会一直需要发出 core-file 在读取一个新数据时; gdb 读取核心以多个几KB的块的方式,  并且只缓存它已经引用的块.

gdb 通常提供的不少功能在你使用内核时不可用. 例如, gdb 不能修改内核数据; 它希望在操作内存前在它自己的控制下运行一个被调试的程序.  也不可能设置断点或观察点, 或者单步过内核函数.

注意, 为了给 gdb 符号信息, 你必须设置 CONFIG_DEBUG_INFO 来编译你的内核. 结果是一个很大的内核映象在磁盘上, 但是,  没有这个信息, 深入内核变量几乎不可能.

有了调试信息, 你可以知道很多内核内部的事情. gdb 愉快地打印出结构, 跟随指针, 等等. 而有一个事情比较难, 然而, 是检查 modules.  因为模块不是传递给gdb 的 vmlinux 映象, 调试器对它们一无所知. 幸运的是, 作为 2.6.7 内核, 有可能教给 gdb  需要如何检查可加载模块.

Linux 可加载模块是 ELF 格式的可执行映象; 这样, 它们被分成几个节. 一个典型的模块可能包含一打或更多节, 但是有 3  个典型的与一次调试会话相关:

- .text  

   这个节包含有模块的可执行代码. 调试器必须知道在哪里以便能够给出回溯或者设置断点.( 这些操作都不相关, 当运行一个调试器在 /proc/kcore 上,  但是它们在使用 kgdb 时可能有用, 下面描述). 

- .bss  

   

- .data  

   这 2 个节持有模块的变量. 在编译时不初始化的任何变量在 .bss 中, 而那些要初始化的在 .data 里.

使 gdb 能够处理可加载模块需要通知调试器一个给定模块的节加载在哪里. 这个信息在 sysfs 中, 在 /sys/module 下. 例如, 在加载  scull 模块后, 目录 /sys/module/scull/sections 包含名子为 .text 的文件; 每个文件的内容是那个节的基地址.

我们现在该发出一个 gdb 命令来告诉它关于我们的模块. 我们需要的命令是 add-symble-flile; 这个命令使用模块目标文件名, .text  基地址作为参数, 以及一系列描述任何其他感兴趣的节安放在哪里的参数. 在深入位于 sysfs 的模块节数据后, 我们可以构建这样一个命令:

```
(gdb) add-symbol-file .../scull.ko 0xd0832000 \
-s .bss 0xd0837100 \
 -s .data 0xd0836be0
```

我们已经包含了一个小脚本在例子代码里( gdbline ), 它为给定的模块可以创建这个命令.

我们现在使用 gdb 检查我们的可加载模块中的变量. 这是一个取自 scull 调试会话的快速例子:

```
(gdb) add-symbol-file scull.ko 0xd0832000 \
-s .bss 0xd0837100 \
 -s .data 0xd0836be0
add symbol table from file "scull.ko" at
 .text_addr = 0xd0832000
 .bss_addr = 0xd0837100
 .data_addr = 0xd0836be0
(y or n) y
Reading symbols from scull.ko...done.
(gdb) p scull_devices[0]
$1 = {data = 0xcfd66c50,
 quantum = 4000,
 qset = 1000,
 size = 20881,
 access_key = 0,
 ...}
```

这里我们看到第一个 scull 设备当前持有 20881 字节. 如果我们想, 我们可以跟随数据链, 或者查看其他任何感兴趣的模块中的东东.

这是另一个值得知道的有用技巧:

```
(gdb) print *(address)
```

这里, 填充 address 指向的一个 16 进制地址; 输出是对应那个地址的代码的文件和行号. 这个技术可能有用, 例如,  来找出一个函数指针真正指向哪里.

我们仍然不能进行典型的调试任务, 如设置断点或者修改数据; 为进行这些操作, 我们需要使用象 kdb( 下面描述 ) 或者 kgdb ( 我们马上就到  )这样的工具.

##### 4.6.2. kdb 内核调试器

许多读者可能奇怪为什么内核没有建立更多高级的调试特性在里面.答案, 非常简单, 是 Linus 不相信交互式的调试器. 他担心它们会导致不好的修改,  这些修改给问题打了补丁而不是找到问题的真正原因. 因此, 没有内嵌的调试器.

其他内核开发者, 但是, 见到了交互式调试工具的一个临时使用. 一个这样的工具是 kdb 内嵌式内核调试器, 作为来自 oss.sgi.com  的一个非官方补丁. 要使用 kdb, 你必须获得这个补丁(确认获得一个匹配你的内核版本的版本), 应用它, 重建并重安装内核. 注意, 直到本书编写时, kdb  只在IA-32(x86)系统中运行(尽管一个给 IA-64 的版本在主线内核版本存在了一阵子, 在被去除之前.)

一旦你运行一个使能了kdb的内核, 有几个方法进入调试器. 在控制台上按下 Pause(或者 Break) 键启动调试器. kdb 在一个内核 oops  发生时或者命中一个断点时也启动, 在任何一种情况下, 你看到象这样的一个消息:

```
Entering kdb (0xc0347b80) on processor 0 due to Keyboard Entry
[0]kdb>
```

注意, 在kdb运行时内核停止任何东西. 在你调用 kdb 的系统中不应当运行其他东西; 特别, 你不应当打开网络 -- 除非, 当然,  你在调试一个网络驱动. 一般地以单用户模式启动系统是一个好主意, 如果你将使用 kdb.

作为一个例子, 考虑一个快速 scull 调试会话. 假设驱动已经加载, 我们可以这样告诉 kdb 在 sucll_read 中设置一个断点:

```
[0]kdb> bp scull_read
Instruction(i) BP #0 at 0xcd087c5dc (scull_read)
 is enabled globally adjust 1
[0]kdb> go
```

bp 命令告诉 kdb 在下一次内核进入 scull_read 时停止. 你接着键入 go 来继续执行. 在将一些东西放入一个 scull 设备后,  我们可以试着通过在另一个终端的外壳下运行 cat 命令来读取它, 产生下面:

```
Instruction(i) breakpoint #0 at 0xd087c5dc (adjusted)
0xd087c5dc scull_read: int3

Entering kdb (current=0xcf09f890, pid 1575) on processor 0 due to
Breakpoint @ 0xd087c5dc
[0]kdb>
```

我们现在位于 scull_read 的开始. 为看到我们任何到那里的, 我们可以获得一个堆栈回溯:

```
[0]kdb> bt
 ESP EIP Function (args)
0xcdbddf74 0xd087c5dc [scull]scull_read
0xcdbddf78 0xc0150718 vfs_read+0xb8
0xcdbddfa4 0xc01509c2 sys_read+0x42
0xcdbddfc4 0xc0103fcf syscall_call+0x7
[0]kdb>
```

kdb 试图打印出调用回溯中每个函数的参数. 然而, 它被编译器的优化技巧搞糊涂了. 因此, 它无法打印 scull_read 的参数.

到时候查看一些数据了. mds 命令操作数据; 我们可以查询 schull_devices 指针的值, 使用这样一个命令:

```
[0]kdb> mds scull_devices 1 
0xd0880de8 cf36ac00 ....
```

这里我们要求一个(4字节)字, 起始于 scull_devices 的位置; 答案告诉我们的设备数组在地址 0xd0880de8; 第一个设备结构自己在  0xcf36ac00. 为查看那个设备结构, 我们需要使用这个地址:

```
[0]kdb> mds cf36ac00
0xcf36ac00 ce137dbc ....
0xcf36ac04 00000fa0 ....
0xcf36ac08 000003e8 ....
0xcf36ac0c 0000009b ....
0xcf36ac10 00000000 ....
0xcf36ac14 00000001 ....
0xcf36ac18 00000000 ....
0xcf36ac1c 00000001 ....
```

这里的 8 行对应于 scull_dev 结构的开始部分. 因此, 我们看到第一个设备的内存位于 0xce137dbc, quantum 是 4000  (16进制 fa0), 量子集大小是 1000 (16进制 3e8 ), 当前有 155( 16进制 9b) 字节存于设备中.

kdb 也可以改变数据. 假想我们要截短一些数据从设备中:

```
[0]kdb> mm cf26ac0c 0x50
0xcf26ac0c = 0x50
```

在设备上一个后续的 cat 会返回比之前少的数据.

kdb 有不少其他功能, 包括单步(指令, 不是 C 源码的一行), 在数据存取上设置断点, 反汇编代码, 步入链表, 存取寄存器数据, 还有更多.  在你应用了 kdb 补丁后, 一个完整的手册页集能够在你的源码树的 documentation/kdb 下发现.

##### 4.6.3. kgdb 补丁

目前为止我们看到的 2 个交互式调试方法( 使用 gdb 于 /proc/kcore 和 kdb) 都缺乏应用程序开发者已经熟悉的那种环境.  如果有一个真正的内核调试器支持改变变量, 断点等特色, 不是很好?

确实, 有这样一个解决方案. 在本书编写时, 2 个分开的补丁在流通中, 它允许 gdb, 具备完全功能, 针对内核运行. 这 2 个补丁都称为  kgdb. 它们通过分开运行测试内核的系统和运行调试器的系统来工作; 这 2 个系统典型地是通过一个串口线连接起来. 因此, 开发者可以在稳定地桌面系统上运行  gdb, 而操作一个运行在专门测试的盒子中的内核. 这种方式建立 gdb 开始需要一些时间, 但是很快会得到回报,当一个难问题出现时.

这些补丁目前处于健壮的状态, 在某些点上可能被合并, 因此我们避免说太多, 除了它们在哪里以及它们的基本特色.  鼓励感兴趣的读者去看这些的当前状态.

第一个 kgdb 补丁当前在 -mm 内核树里 -- 补丁进入 2.6 主线的集结场. 补丁的这个版本支持 x86, SuperH, ia64,  x86_64, 和 32位 PPC 体系. 除了通过串口操作的常用模式, 这个版本的 kgdb 可以通过一个局域网通讯. 使能以太网模式并且使用  kgdboe参数指定发出调试命令的 IP 地址来启动内核. 在 Documentation/i386/kgdb 下的文档描述了如何建立.[[16](#ftn.id423580)]

作为一个选择, 你可使用位于 http://kgdb.sf.net 的kgdb补丁. 这个调试器的版本不支持网络通讯模式(尽管据说在开发中),  但是它确实有内嵌的使用可加载模块的支持. 它支持 x86, x86_64, PowerPC, 和 S/390 体系.

##### 4.6.4. 用户模式 Linux 移植

用户模式 Linux (UML) 是一个有趣的概念. 它被构建为一个分开的 Linux 内核移植, 有它自己的 arch/um 子目录.  它不在一个新的硬件类型上运行, 但是; 相反, 它运行在一个由 Linux 系统调用接口实现的虚拟机上. 如此, UML 使用 Linux 内核来运行,  作为一个Linux 系统上的独立的用户模式进程.

有一个作为用户进程运行的内核拷贝有几个优点. 因为它们运行在一个受限的虚拟的处理器上, 一个错误的内核不能破坏"真实的"系统.  可以在同一台盒子轻易的尝试不同的硬件和软件配置. 并且, 也许对内核开发者而言, 用户模式内核可容易地使用 gdb 和 其他调试器操作.

毕竟, 它只是一个进程. UML 显然有加快内核开发的潜力.

然而, UML 有个大的缺点,从驱动编写者的角度看: 用户模式内核无法存取主机系统的硬件. 因此, 虽然它对于调试大部分本书的例子驱动是有用的, UML  对于不得不处理真实硬件的驱动的调试还是没有用处.

看 http://user-mode-linux.sf.net/ 关于 UML 的更多信息.

##### 4.6.5. Linux 追踪工具

Linux Trace Toolkit (LTT) 是一个内核补丁以及一套相关工具, 允许追踪内核中的事件. 这个追踪包括时间信息,  可以创建一个给定时间段内发生事情的合理的完整图像. 因此, 它不仅用来调试也可以追踪性能问题.

LTT, 同广泛的文档一起, 可以在 http://www.opersys.com/LTT 找到.

##### 4.6.6. 动态探针

Dynamic Probes ( DProbes ) 是由 IBM 发行的(在 GPL 之下)为 IA-32 体系的 Linux 的调试工具.  它允许安放一个"探针"在几乎系统中任何地方, 用户空间和内核空间都可以. 探针由一些代码组成( 有一个特殊的,面向堆栈的语言写成), 当控制命中给定的点时执行.  这个代码可以报告信息给用户空间, 改变寄存器, 或者做其他很多事情. DProbes 的有用特性是, 一旦这个能力建立到内核中,  探针可以在任何地方插入在一个运行中的系统中, 不用内核建立或者重启. DProbes 可以和 LTT 一起来插入一个新的跟踪事件在任意位置.

DProbes 工具可以从 IBM 的开放源码网站:http://oss.sof-ware.ibm.com 下载.

#### 第 5 章 并发和竞争情况

所以在使用spin_lock时要明确知道该锁不会在中断处理程序中使用。

在任何情况下使用spin_lock_irq都是安全的。因为它既禁止本地中断，又禁止内核抢占。

spin_lock比spin_lock_irq速度快，但是它并不是任何情况下都是安全的。



如果自旋锁在中断处理函数中被用到，那么在获取该锁之前需要关闭本地中断，spin_lock_irqsave 只是下列动作的一个便利接口：
1 保存本地中断状态
2 关闭本地中断
3 获取自旋锁

如果你确定在获取锁之前本地中断是开启的，那么就不需要保存中断状态，解锁的时候直接将本地中断启用就可以啦

如果你确实怀疑锁竞争在损坏性能, 你可能发现 lockmeter 工具有用. 这个补丁(从 
http://oss.sgi.com/projects/lockmeter/ 可得到) 装备内核来测量在锁等待花费的时间. 通过看这个报告, 
你能够很快知道是否锁竞争真的是问题.

在设备驱动中环形缓存出现相当多. 网络适配器, 特别地, 常常使用环形缓存来与处理器交换数据(报文). 注意, 对于 2.6.10, 
有一个通用的环形缓存实现在内核中可用; 如何使用它的信息看 <linux/kfifo.h>.

#### 第 6 章 高级字符驱动操作

对于一个 Linux 驱动使一个进程睡眠是一个容易做的事情. 但是, 有几个规则必须记住以安全的方式编码睡眠.

这些规则的第一个是: 当你运行在原子上下文时不能睡眠. 我们在第 5 章介绍过原子操作; 一个原子上下文只是一个状态,  这里多个步骤必须在没有任何类型的并发存取的情况下进行. 这意味着, 对于睡眠, 是你的驱动在持有一个自旋锁, seqlock, 或者 RCU 锁时不能睡眠.  如果你已关闭中断你也不能睡眠. 在持有一个旗标时睡眠是合法的, 但是你应当仔细查看这样做的任何代码. 如果代码在持有一个旗标时睡眠,  任何其他的等待这个旗标的线程也睡眠. 因此发生在持有旗标时的任何睡眠应当短暂, 并且你应当说服自己, 由于持有这个旗标,  你不能阻塞这个将最终唤醒你的进程.



另一件要记住的事情是, 当你醒来, 你从不知道你的进程离开 CPU 多长时间或者同时已经发生了什么改变. 
你也常常不知道是否另一个进程已经睡眠等待同一个事件; 那个进程可能在你之前醒来并且获取了你在等待的资源. 结果是你不能关于你醒后的系统状态做任何的假设, 
并且你必须检查来确保你在等待的条件是, 确实, 真的.

一个另外的相关的点, 当然, 是你的进程不能睡眠除非确信其他人, 在某处的, 将唤醒它. 做唤醒工作的代码必须也能够找到你的进程来做它的工作. 
确保一个唤醒发生, 是深入考虑你的代码和对于每次睡眠, 确切知道什么系列的事件将结束那次睡眠. 使你的进程可能被找到, 真正地, 
通过一个称为等待队列的数据结构实现的. 一个等待队列就是它听起来的样子:一个进程列表, 都等待一个特定的事件.

```c
wait_event(queue, condition)
wait_event_interruptible(queue, condition)
wait_event_timeout(queue, condition, timeout)
wait_event_interruptible_timeout(queue, condition, timeout)
```

```c
void wake_up(wait_queue_head_t *queue);
void wake_up_interruptible(wait_queue_head_t *queue);
```

只有 read, write, 和 open 文件操作受到非阻塞标志影响.



llseek 方法实现了 lseek 和 llseek 系统调用. 我们已经说了如果 llseek 方法从设备的操作中缺失, 内核中的缺省的实现进行移位通过修改 
filp->f_pos

唯一设备特定的操作是从设备中获取文件长度. 在 scull 中 read 和 write 方法如需要地一样协作, 如同在第 3 章所示.



- _IOC_NRBITS  

   

- _IOC_TYPEBITS  

   

- _IOC_SIZEBITS  

   

- _IOC_DIRBITS  

   ioctl 命令的不同位段所使用的位数. 还有 4 个宏来指定 MASK 和 4 个指定 SHIFT, 但是它们主要是给内部使用.  _IOC_SIZEBIT 是一个要检查的重要的值, 因为它跨体系改变. 

- _IOC_NONE  

   

- _IOC_READ  

   

- _IOC_WRITE  

   "方向"位段可能的值. "read" 和 "write" 是不同的位并且可相或来指定 read/write. 这些值是基于 0 的. 

- _IOC(dir,type,nr,size)  

   

- _IO(type,nr)  

   

- _IOR(type,nr,size)  

   

- _IOW(type,nr,size)  

   

- _IOWR(type,nr,size)  

   用来创建 ioclt 命令的宏定义. 

- _IOC_DIR(nr)  

   

- _IOC_TYPE(nr)  

   

- _IOC_NR(nr)  

   

- _IOC_SIZE(nr)  

   用来解码一个命令的宏定义. 特别地, _IOC_TYPE(nr) 是 _IOC_READ 和 _IOC_WRITE 的 OR 结合. 

- #include <asm/uaccess.h>  

   

- int access_ok(int type, const void *addr, unsigned  long size);  

   检查一个用户空间的指针是可用的. access_ok 返回一个非零值, 如果应当允许存取.

- VERIFY_READ  

   

- VERIFY_WRITE  

   access_ok 中 type 参数的可能取值. VERIFY_WRITE 是 VERIFY_READ 的超集. 

  ```c
  #include <asm/uaccess.h>  
   int put_user(datum,ptr);  
  int get_user(local,ptr);  
  int __put_user(datum,ptr);  
  int __get_user(local,ptr);  
  ```

   用来存储或获取一个数据到或从用户空间的宏. 传送的字节数依赖 sizeof(*ptr). 常规的版本调用 access_ok , 而常规版本(  __put_user 和 __get_user ) 假定 access_ok 已经被调用了. 

- #include <linux/capability.h>  

   定义各种 CAP_ 符号, 描述一个用户空间进程可有的能力. 

- int capable(int capability);  

   返回非零值如果进程有给定的能力. 

- #include <linux/wait.h>  

   

- typedef struct { /* ... */ }  wait_queue_head_t;  

   

- void init_waitqueue_head(wait_queue_head_t  *queue);  

   

- DECLARE_WAIT_QUEUE_HEAD(queue);  

   Linux 等待队列的定义类型. 一个 wait_queue_head_t 必须被明确在运行时使用 init_waitqueue_head 或者编译时使用  DEVLARE_WAIT_QUEUE_HEAD 进行初始化. 

- void wait_event(wait_queue_head_t q, int  condition);  

   

- int wait_event_interruptible(wait_queue_head_t q, int  condition);  

   

- int wait_event_timeout(wait_queue_head_t q, int  condition, int time);  

   

- int  wait_event_interruptible_timeout(wait_queue_head_t q, int condition,int  time);  

   使进程在给定队列上睡眠, 直到给定条件值为真值. 

- void wake_up(struct wait_queue **q);  

   

- void wake_up_interruptible(struct wait_queue  **q);  

   

- void wake_up_nr(struct wait_queue **q, int  nr);  

   

- void wake_up_interruptible_nr(struct wait_queue **q,  int nr);  

   

- void wake_up_all(struct wait_queue  **q);  

   

- void wake_up_interruptible_all(struct wait_queue  **q);  

   

- void wake_up_interruptible_sync(struct wait_queue  **q);  

   唤醒在队列 q 上睡眠的进程. _interruptible 的形式只唤醒可中断的进程. 正常地, 只有一个互斥等待者被唤醒, 但是这个行为可被 _nr  或者 _all 形式所改变. _sync 版本在返回之前不重新调度 CPU. 

- #include <linux/sched.h>  

   

- set_current_state(int state);  

   设置当前进程的执行状态. TASK_RUNNING 意味着它已经运行, 而睡眠状态是 TASK_INTERRUPTIBLE 和  TASK_UNINTERRUPTIBLE. 

- void schedule(void);  

   选择一个可运行的进程从运行队列中. 被选中的进程可是当前进程或者另外一个. 

- typedef struct { /* ... */ }  wait_queue_t;  

   

- init_waitqueue_entry(wait_queue_t *entry, struct  task_struct *task);  

   wait_queue_t 类型用来放置一个进程到一个等待队列. 

- void prepare_to_wait(wait_queue_head_t *queue,  wait_queue_t *wait, int state);  

   

- void prepare_to_wait_exclusive(wait_queue_head_t  *queue, wait_queue_t *wait, int state);  

   

- void finish_wait(wait_queue_head_t *queue,  wait_queue_t *wait);  

   帮忙函数, 可用来编码一个手工睡眠. 

- void sleep_on(wiat_queue_head_t  *queue);  

   

- void interruptible_sleep_on(wiat_queue_head_t  *queue);  

   老式的不推荐的函数, 它们无条件地使当前进程睡眠. 

- #include <linux/poll.h>  

   

- void poll_wait(struct file *filp, wait_queue_head_t  *q, poll_table *p);  

   将当前进程放入一个等待队列, 不立刻调度. 它被设计来被设备驱动的 poll 方法使用. 

- int fasync_helper(struct inode *inode, struct file  *filp, int mode, struct fasync_struct **fa);  

   一个"帮忙者", 来实现 fasync 设备方法. mode 参数是传递给方法的相同的值, 而 fa 指针指向一个设备特定的 fasync_struct  *. 

- void kill_fasync(struct fasync_struct *fa, int sig,  int band);  

   如果这个驱动支持异步通知, 这个函数可用来发送一个信号到登记在 fa 中的进程. 

- int nonseekable_open(struct inode *inode, struct file  *filp);  

   

- loff_t no_llseek(struct file *file, loff_t offset,  int whence);  

   nonseekable_open 应当在任何不支持移位的设备的 open 方法中被调用. 这样的设备应当使用 no_llseek 作为它们的 llseek  方法.

#### 第 7 章 时间, 延时, 和延后工作

##### 7.7.1. 时间管理

#include <linux/param.h>  

HZ  

 HZ 符号指定了每秒产生的时钟嘀哒的数目. 

#include <linux/jiffies.h>  

 volatile unsigned long jiffies;  

 u64 jiffies_64;  

 jiffies_64 变量每个时钟嘀哒时被递增; 因此, 它是每秒递增 HZ 次. 内核代码几乎常常引用 jiffies, 它在 64-位平台和  jiffies_64 相同并且在 32-位平台是它低有效的一半. 

```c
int time_after(unsigned long a, unsigned long  b);  
int time_before(unsigned long a, unsigned long  b);  
int time_after_eq(unsigned long a, unsigned long  b);  
int time_before_eq(unsigned long a, unsigned long  b);  
```

 这些布尔表达式以一种安全的方式比较 jiffies, 没有万一计数器溢出的问题和不需要存取 jiffies_64. 

u64 get_jiffies_64(void);  

 获取 jiffies_64 而没有竞争条件. 

```c
#include <linux/time.h>  
unsigned long timespec_to_jiffies(struct timespec  *value);  
void jiffies_to_timespec(unsigned long jiffies,  struct timespec *value);  
unsigned long timeval_to_jiffies(struct timeval  *value);  
void jiffies_to_timeval(unsigned long jiffies, struct  timeval *value);  
```

 在 jiffies 和其他表示之间转换时间表示. 

```c
#include <asm/msr.h>  
rdtsc(low32,high32);  
rdtscl(low32);  
rdtscll(var32);  
```

 x86-特定的宏定义来读取时戳计数器. 它们作为 2 半 32-位来读取, 只读低一半, 或者全部读到一个 long long 变量. 

```
#include <linux/timex.h>  
cycles_t get_cycles(void);  
```

 以平台独立的方式返回时戳计数器. 如果 CPU 没提供时戳特性, 返回 0. 

```c
#include <linux/time.h>  
unsigned long mktime(year, mon, day, h, m,  s);  
```

 返回自 Epoch 以来的秒数, 基于 6 个 unsigned int 参数. 

```c
void do_gettimeofday(struct timeval  *tv);  
```

 返回当前时间, 作为自 Epoch 以来的秒数和微秒数, 用硬件能提供的最好的精度. 在大部分的平台这个解决方法是一个微秒或者更好, 尽管一些平台只提供  jiffies 精度. 

```c
struct timespec  current_kernel_time(void);  
```

 返回当前时间, 以一个 jiffy 的精度.

##### 7.7.2. 延迟

```c
#include <linux/wait.h>  
long  wait_event_interruptible_timeout(wait_queue_head_t *q, condition, signed long  timeout);  
```

 使当前进程在等待队列进入睡眠, 安装一个以 jiffies 表达的超时值. 使用 schedule_timeout( 下面) 给不可中断睡眠. 

```c
#include <linux/sched.h>  
 signed long schedule_timeout(signed long  timeout);  
```

 调用调度器, 在确保当前进程在超时到的时候被唤醒后. 调用者首先必须调用 set_curret_state  来使自己进入一个可中断的或者不可中断的睡眠状态. 

```c
 #include <linux/delay.h>  
void ndelay(unsigned long nsecs);  
void udelay(unsigned long usecs);  
void mdelay(unsigned long msecs); 
```

 

 引入一个整数纳秒, 微秒和毫秒的延迟. 获得的延迟至少是请求的值, 但是可能更多. 每个函数的参数必须不超过一个平台特定的限制(常常是几千). 

```c
void msleep(unsigned int millisecs);  
unsigned long msleep_interruptible(unsigned int  millisecs);  
 void ssleep(unsigned int seconds);  
```

 使进程进入睡眠给定的毫秒数(或者秒, 如果使 ssleep).

##### 7.7.3. 内核定时器

```c
#include <asm/hardirq.h>  
int in_interrupt(void);  
int in_atomic(void);  
```

 返回一个布尔值告知是否调用代码在中断上下文或者原子上下文执行. 中断上下文是在一个进程上下文之外, 或者在硬件或者软件中断处理中.  原子上下文是当你不能调度一个中断上下文或者一个持有一个自旋锁的进程的上下文. 

```c
#include <linux/timer.h>  
void init_timer(struct timer_list *  timer);  
struct timer_list TIMER_INITIALIZER(_function,  _expires, _data);  
```

 这个函数和静态的定时器结构的声明是初始化一个 timer_list 数据结构的 2 个方法. 

- void add_timer(struct timer_list *  timer);  

   注册定时器结构来在当前 CPU 上运行. 

- int mod_timer(struct timer_list *timer, unsigned long  expires);  

   改变一个已经被调度的定时器结构的超时时间. 它也能作为一个 add_timer 的替代. 

- int timer_pending(struct timer_list *  timer);  

   宏定义, 返回一个布尔值说明是否这个定时器结构已经被注册运行. 

- void del_timer(struct timer_list *  timer);  

   

- void del_timer_sync(struct timer_list *  timer);  

   从激活的定时器链表中去除一个定时器. 后者保证这定时器当前没有在另一个 CPU 上运行.

##### 7.7.4. Tasklets  机制

```c
#include <linux/interrupt.h>  
 DECLARE_TASKLET(name, func, data);  
DECLARE_TASKLET_DISABLED(name, func,  data);  
void tasklet_init(struct tasklet_struct *t, void  (*func)(unsigned long), unsigned long data);  
```

 前 2 个宏定义声明一个 tasklet 结构, 而 tasklet_init 函数初始化一个已经通过分配或其他方式获得的 tasklet 结构. 第 2  个 DECLARE 宏标识这个 tasklet 为禁止的. 

```c
void tasklet_disable(struct tasklet_struct  *t);  
void tasklet_disable_nosync(struct tasklet_struct  *t);  
void tasklet_enable(struct tasklet_struct  *t);  
```

 禁止和使能一个 tasklet. 每个禁止必须配对一个使能( 你可以禁止这个 tasklet 即便它已经被禁止). 函数 tasklet_disable  等待 tasklet 终止如果它在另一个 CPU 上运行. 这个非同步版本不采用这个额外的步骤. 

```c
void tasklet_schedule(struct tasklet_struct  *t);  
void tasklet_hi_schedule(struct tasklet_struct  *t);  
```

 调度一个 tasklet 运行, 或者作为一个"正常" tasklet 或者一个高优先级的. 当软中断被执行, 高优先级 tasklets 被首先处理,  而正常 tasklet 最后执行. 

- void tasklet_kill(struct tasklet_struct  *t);  

   从激活的链表中去掉 tasklet, 如果它被调度执行. 如同 tasklet_disable, 这个函数可能在 SMP 系统中阻塞等待 tasklet  终止, 如果它当前在另一个 CPU 上运行.

##### 7.7.5. 工作队列

```c
 #include <linux/workqueue.h>  
struct workqueue_struct;  
struct work_struct;  
```

 这些结构分别表示一个工作队列和一个工作入口. 

```c
struct workqueue_struct *create_workqueue(const char  *name);  
struct workqueue_struct  *create_singlethread_workqueue(const char *name);  
void destroy_workqueue(struct workqueue_struct  *queue);  
```

 创建和销毁工作队列的函数. 一个对 create_workqueue 的调用创建一个有一个工作者线程在系统中每个处理器上的队列; 相反,  create_singlethread_workqueue 创建一个有一个单个工作者进程的工作队列. 

```c
DECLARE_WORK(name, void (*function)(void *), void  *data);  
INIT_WORK(struct work_struct *work, void  (*function)(void *), void *data);  
PREPARE_WORK(struct work_struct *work, void  (*function)(void *), void *data);  
```

 声明和初始化工作队列入口的宏. 

```c
int queue_work(struct workqueue_struct *queue, struct  work_struct *work);  
int queue_delayed_work(struct workqueue_struct  *queue, struct work_struct *work, unsigned long delay);  
```

 从一个工作队列对工作进行排队执行的函数. 

```c
int cancel_delayed_work(struct work_struct  *work);  
void flush_workqueue(struct workqueue_struct  *queue);  
```

 使用 cancel_delayed_work 来从一个工作队列中去除入口; flush_workqueue  确保没有工作队列入口在系统中任何地方运行. 

```c
int schedule_work(struct work_struct  *work);  
int schedule_delayed_work(struct work_struct *work,  unsigned long delay);  
void flush_scheduled_work(void);  
```

 使用共享队列的函数.

#### 第 8 章 分配内存

程序员应当记住 kmalloc 能够处理的最小分配是 32 或者 64 字节, 依赖系统的体系所使用的页大小.

相关于内存分配的函数和符号是:

```c
#include <linux/slab.h>  
void *kmalloc(size_t size, int flags);  
void kfree(void *obj);  
```

 内存分配的最常用接口. 

```c
#include <linux/mm.h>  
GFP_USER  
GFP_KERNEL  
GFP_NOFS  
GFP_NOIO  
GFP_ATOMIC  
```

 控制内存分配如何进行的标志, 从最少限制的到最多的. GFP_USER 和 GFP_KERNEL 优先级允许当前进程被置为睡眠来满足请求.  GFP_NOFS 和 GFP_NOIO 禁止文件系统操作和所有的 I/O 操作, 分别地, 而 GFP_ATOMIC 分配根本不能睡眠. 

```c
__GFP_DMA  
__GFP_HIGHMEM  
__GFP_COLD  
__GFP_NOWARN  
__GFP_HIGH  
__GFP_REPEAT  
__GFP_NOFAIL  
__GFP_NORETRY  
```

 这些标志修改内核的行为, 当分配内存时. 

```c
#include <linux/malloc.h>  
kmem_cache_t *kmem_cache_create(char *name, size_t  size, size_t offset, unsigned long flags, constructor(), destructor(  ));  
int kmem_cache_destroy(kmem_cache_t  *cache);  
```

 创建和销毁一个 slab 缓存. 这个缓存可被用来分配几个相同大小的对象. 

```c
SLAB_NO_REAP  
SLAB_HWCACHE_ALIGN  
SLAB_CACHE_DMA  
```

 在创建一个缓存时可指定的标志. 

```c
SLAB_CTOR_ATOMIC  
SLAB_CTOR_CONSTRUCTOR  
```

 分配器可用传递给构造函数和析构函数的标志. 

```c
void *kmem_cache_alloc(kmem_cache_t *cache, int  flags);  
void kmem_cache_free(kmem_cache_t *cache, const void  *obj);  
```

 从缓存中分配和释放一个单个对象. /proc/slabinfo 一个包含对 slab 缓存使用情况统计的虚拟文件. 

```c
#include <linux/mempool.h>  
mempool_t *mempool_create(int min_nr, mempool_alloc_t  *alloc_fn, mempool_free_t *free_fn, void *data);  
void mempool_destroy(mempool_t *pool); 
```

 

 创建内存池的函数, 它试图避免内存分配设备, 通过保持一个已分配项的"紧急列表". 

```c
void *mempool_alloc(mempool_t *pool, int  gfp_mask);  
void mempool_free(void *element, mempool_t  *pool);  
```

 从(并且返回它们给)内存池分配项的函数. 

```c
unsigned long get_zeroed_page(int  flags);  
unsigned long __get_free_page(int  flags);  
unsigned long __get_free_pages(int flags, unsigned  long order);  
```

 面向页的分配函数. get_zeroed_page 返回一个单个的, 零填充的页. 这个调用的所有的其他版本不初始化返回页的内容. 

- int get_order(unsigned long size);  

   返回关联在当前平台的大小的分配级别, 根据 PAGE_SIZE. 这个参数必须是 2 的幂, 并且返回值至少是 0. 

- void free_page(unsigned long addr); 

- void free_pages(unsigned long addr, unsigned long  order);  

   释放面向页分配的函数. 

- struct page *alloc_pages_node(int nid, unsigned int  flags, unsigned int order);  

- struct page *alloc_pages(unsigned int flags, unsigned  int order);  

- struct page *alloc_page(unsigned int  flags);  

   Linux 内核中最底层页分配器的所有变体. 

- void __free_page(struct page *page);  

- void __free_pages(struct page *page, unsigned int  order);  

- void free_hot_page(struct page *page);  

   使用一个 alloc_page 形式分配的页的各种释放方法. 

  ```c
  #include <linux/vmalloc.h>  
  void * vmalloc(unsigned long size);  
  void vfree(void * addr);  
   #include <asm/io.h>  
  void * ioremap(unsigned long offset, unsigned long  size);  
  void iounmap(void *addr);  
  ```

   分配或释放一个连续虚拟地址空间的函数. iormap 存取物理内存通过虚拟地址, 而 vmalloc 分配空闲页. 使用 ioreamp 映射的区是  iounmap 释放, 而从 vmalloc 获得的页使用 vfree 来释放. 

- #include <linux/percpu.h>  

- DEFINE_PER_CPU(type, name);  

- DECLARE_PER_CPU(type, name);  

   定义和声明每-CPU变量的宏. 

- per_cpu(variable, int cpu_id)  

- get_cpu_var(variable)  

- put_cpu_var(variable)  

   提供对静态声明的每-CPU变量存取的宏. 

- void *alloc_percpu(type);  

- void *__alloc_percpu(size_t size, size_t  align);  

- void free_percpu(void *variable);  

   进行运行时分配和释放每-CPU变量的函数. 

- int get_cpu( );  

- void put_cpu( );  

- per_cpu_ptr(void *variable, int cpu_id)   

   get_cpu 获得对当前处理器的引用(因此, 阻止抢占和移动到另一个处理器)并且返回处理器的ID; put_cpu 返回这个引用.  为存取一个动态分配的每-CPU变量, 用应当被存取版本所在的 CPU 的 ID 来使用 per_cpu_ptr. 对一个动态的每-CPU 变量当前 CPU  版本的操作, 应当用对 get_cpu 和 put_cpu 的调用来包围.  

- #include <linux/bootmem.h> 

- void *alloc_bootmem(unsigned long  size);   

- void *alloc_bootmem_low(unsigned long  size); 

- void *alloc_bootmem_pages(unsigned long  size);  

- void *alloc_bootmem_low_pages(unsigned long  size);  

- void free_bootmem(unsigned long addr, unsigned long  size);  

   在系统启动时进行分配和释放内存的函数(只能被直接连接到内核中去的驱动使用)



#### 第 9 章 与硬件通讯

本章介绍下列与硬件管理相关的符号:

```
- #include <linux/kernel.h>  
- void barrier(void)  
```

 这个"软件"内存屏蔽要求编译器对待所有内存是跨这个指令而非易失的. 

```c
- #include <asm/system.h>  
- void rmb(void);  
- void read_barrier_depends(void);  
- void wmb(void);  
- void mb(void);  
```

 硬件内存屏障. 它们请求 CPU(和编译器)来检查所有的跨这个指令的内存读, 写, 或都有. 

```c
- #include <asm/io.h>  
- unsigned inb(unsigned port);  
- void outb(unsigned char byte, unsigned  port);  
- unsigned inw(unsigned port);  
- void outw(unsigned short word, unsigned  port);  
- unsigned inl(unsigned port);  
- void outl(unsigned doubleword, unsigned  port); 
```

 

 用来读和写 I/O 端口的函数. 它们还可以被用户空间程序调用, 如果它们有正当的权限来存取端口. 

- unsigned inb_p(unsigned port);  

   如果在一次 I/O 操作后需要一个小延时, 你可以使用在前一项中介绍的这些函数的 6 个暂停对应部分; 这些暂停函数有以 _p 结尾的名子. 

  ```c
  - void insb(unsigned port, void *addr, unsigned long  count);  
  - void outsb(unsigned port, void *addr, unsigned long  count);  
  - void insw(unsigned port, void *addr, unsigned long  count);  
  - void outsw(unsigned port, void *addr, unsigned long  count);  
  - void insl(unsigned port, void *addr, unsigned long  count);  
  - void outsl(unsigned port, void *addr, unsigned long  count);  
  ```

   这些"字串函数"被优化为传送数据从一个输入端口到一个内存区, 或者其他的方式. 这些传送通过读或写到同一端口 count 次来完成. 

  ```c
  - #include <linux/ioport.h>  
  - struct resource *request_region(unsigned long start,  unsigned long len, char *name); 
  - void release_region(unsigned long start, unsigned  long len);  
  - int check_region(unsigned long start, unsigned long  len);  
  ```

   I/O 端口的资源分配器. 这个检查函数成功返回 0 并且在错误时小于 0. 

- struct resource *request_mem_region(unsigned long  start, unsigned long len, char *name);  

   

- void release_mem_region(unsigned long start, unsigned  long len);  

   

- int check_mem_region(unsigned long start, unsigned  long len);  

   为内存区处理资源分配的函数 

- #include <asm/io.h>  

   

- void *ioremap(unsigned long phys_addr, unsigned long  size);  

   

- void *ioremap_nocache(unsigned long phys_addr,  unsigned long size);  

   

- void iounmap(void *virt_addr);  

   ioremap 重映射一个物理地址范围到处理器的虚拟地址空间, 使它对内核可用. iounmap 释放映射当不再需要它时. 

- #include <asm/io.h>  

   

- unsigned int ioread8(void *addr);  

   

- unsigned int ioread16(void *addr);  

   

- unsigned int ioread32(void *addr);  

   

- void iowrite8(u8 value, void *addr);  

   

- void iowrite16(u16 value, void *addr);  

   

- void iowrite32(u32 value, void *addr);  

   用来使用 I/O 内存的存取者函数. 

- void ioread8_rep(void *addr, void *buf, unsigned long  count);  

   

- void ioread16_rep(void *addr, void *buf, unsigned  long count);  

   

- void ioread32_rep(void *addr, void *buf, unsigned  long count);  

   

- void iowrite8_rep(void *addr, const void *buf,  unsigned long count);  

   

- void iowrite16_rep(void *addr, const void *buf,  unsigned long count);  

   

- void iowrite32_rep(void *addr, const void *buf,  unsigned long count);  

   I/O 内存原语的"重复"版本. 

- unsigned readb(address);  

   

- unsigned readw(address);  

   

- unsigned readl(address);  

   

- void writeb(unsigned value, address);  

   

- void writew(unsigned value, address);  

   

- void writel(unsigned value, address);  

   

- memset_io(address, value, count);  

   

- memcpy_fromio(dest, source, nbytes);  

   

- memcpy_toio(dest, source, nbytes);  

   旧的, 类型不安全的存取 I/O 内存的函数. 

- void *ioport_map(unsigned long port, unsigned int  count);  

   

- void ioport_unmap(void *addr);  

   一个想对待 I/O 端口如同它们是 I/O 内存的驱动作者, 可以传递它们的端口给 ioport_map. 这个映射应当在不需要的时候恢复( 使用  ioport_unmap )

#### 第 10 章 中断处理

tasklet 常常是后半部处理的首选机制; 它们非常快, 但是所有的 tasklet 代码必须是原子的. tasklet 的可选项是工作队列, 
它可能有一个更高的运行周期但是允许睡眠.



本章中介绍了这些关于中断管理的符号:

- #include <linux/interrupt.h>  

   

- int request_irq(unsigned int irq, irqreturn_t  (*handler)( ), unsigned long flags, const char *dev_name, void  *dev_id);  

   

- void free_irq(unsigned int irq, void  *dev_id);  

   调用这个注册和注销一个中断处理. 

- #include <linux/irq.h.h>  

   

- int can_request_irq(unsigned int irq, unsigned long  flags);  

   这个函数, 在 i386 和 x86_64 体系上有, 返回一个非零值如果一个分配给定中断线的企图成功. 

- #include <asm/signal.h>  

   

- SA_INTERRUPT  

   

- SA_SHIRQ  

   

- SA_SAMPLE_RANDOM  

   给 request_irq 的标志. SA_INTERRUPT 请求安装一个快速处理者( 相反是一个慢速的). SA_SHIRQ 安装一个共享的处理者,  并且第 3 个 flag 声称中断时戳可用来产生系统熵. 

- /proc/interrupts  

   

- /proc/stat  

   报告硬件中断和安装的处理者的文件系统节点. 

- unsigned long probe_irq_on(void);  

   

- int probe_irq_off(unsigned long);  

   驱动使用的函数, 当它不得不探测来决定哪个中断线被设备在使用. probe_irq_on 的结果必须传回给 probe_irq_off 在中断产生之后.  probe_irq_off 的返回值是被探测的中断号. 

- IRQ_NONE  

   

- IRQ_HANDLED  

   

- IRQ_RETVAL(int x)  

   从一个中断处理返回的可能值, 指示是否一个来自设备的真正的中断出现了. 

- void disable_irq(int irq);  

   

- void disable_irq_nosync(int irq);  

   

- void enable_irq(int irq);  

   驱动可以使能和禁止中断报告. 如果硬件试图在中断禁止时产生一个中断, 这个中断永远丢失了. 一个使用一个共享处理者的驱动必须不使用这个函数. 

- void local_irq_save(unsigned long  flags);  

   

- void local_irq_restore(unsigned long  flags);  

   使用 local_irq_save 来禁止本地处理器的中断并且记住它们之前的状态. flags 可以被传递给 local_irq_restore  来恢复之前的中断状态. 

- void local_irq_disable(void);  

   

- void local_irq_enable(void);  

   在当前处理器熵无条件禁止和使能中断的函数.

#### 第 11 章 内核中的数据类型

下列符号在本章中介绍了:

- #include <linux/types.h>  

   

- typedef u8;  

   

- typedef u16;  

   

- typedef u32;  

   

- typedef u64;  

   保证是 8-位, 16-位, 32-位 和64-位 无符号整型值的类型. 对等的有符号类型也存在. 在用户空间, 你可用 __u8, __u16,  等等来引用这些类型. 

- #include <asm/page.h>  

   

- PAGE_SIZE  

   

- PAGE_SHIFT  

   给当前体系定义每页的字节数, 以及页偏移的位数( 对于 4 KB 页是 12, 8 KB 是 13 )的符号. 

- #include <asm/byteorder.h>  

   

- __LITTLE_ENDIAN  

   

- __BIG_ENDIAN  

   这 2 个符号只有一个定义, 依赖体系. 

- #include <asm/byteorder.h>  

   

- u32 __cpu_to_le32 (u32);  

   

- u32 __le32_to_cpu (u32);  

   在已知字节序和处理器字节序之间转换的函数. 有超过 60 个这样的函数: 在 include/linux/byteorder/  中的各种文件有完整的列表和它们以何种方式定义. 

- #include <asm/unaligned.h>  

   

- get_unaligned(ptr);  

   

- put_unaligned(val, ptr);  

   一些体系需要使用这些宏保护不对齐的数据存取. 这些宏定义扩展成通常的指针解引用, 为那些允许你存取不对齐数据的体系. 

- #include <linux/err.h>  

   

- void *ERR_PTR(long error);  

   

- long PTR_ERR(const void *ptr);  

   

- long IS_ERR(const void *ptr);  

   允许错误码由返回指针值的函数返回. 

- #include <linux/list.h>  

   

- list_add(struct list_head *new, struct list_head  *head);  

   

- list_add_tail(struct list_head *new, struct list_head  *head);  

   

- list_del(struct list_head *entry);  

   

- list_del_init(struct list_head *entry);   

   

- list_empty(struct list_head *head);  

   

- list_entry(entry, type, member);  

   

- list_move(struct list_head *entry, struct list_head  *head);  

   

- list_move_tail(struct list_head *entry, struct  list_head *head);  

   

- list_splice(struct list_head *list, struct list_head  *head);  

   操作环形, 双向链表的函数. 

- list_for_each(struct list_head *cursor, struct  list_head *list)  

   

- list_for_each_prev(struct list_head *cursor, struct  list_head *list)  

   

- list_for_each_safe(struct list_head *cursor, struct  list_head *next, struct list_head *list)  

   

- list_for_each_entry(type *cursor, struct list_head  *list, member)  

   

- list_for_each_entry_safe(type *cursor, type *next  struct list_head *list, member)  

   方便的宏定义, 用在遍历链表上.

#### 第 12 章 PCI 驱动