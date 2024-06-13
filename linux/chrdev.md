# linux设备驱动详解

## 基础架构

<img src="https://pic1.zhimg.com/v2-fbcffc3efcd72eda56aa629f6fb3b384_r.jpg" title="" alt="v2-fbcffc3efcd72eda56aa629f6fb3b384_r" data-align="center">

<img src="https://pic1.zhimg.com/v2-be68be1b5f92606d94be126fa299f94c_r.jpg" title="" alt="v2-be68be1b5f92606d94be126fa299f94c_r" data-align="center">

### **三种设备的定义分别如下：**

1. 字符设备：只能一个字节一个字节的读写的设备，不能随机读取设备内存中的某一数据，读取数据需要按照先后顺序进行。字符设备是面向流的设备，常见的字符设备如鼠标、键盘、串口、控制台、LED等。

2. 块设备：是指可以从设备的任意位置读取一定长度的数据设备。块设备如硬盘、磁盘、U盘和SD卡等存储设备。

3. 网络设备：网络设备比较特殊，不在是对文件进行操作，而是由专门的网络接口来实现。应用程序不能直接访问网络设备驱动程序。在/dev目录下也没有文件来表示网络设备。
* 对于字符设备和块设备来说，在/dev目录下都有对应的设备文件。linux用户程序通过设备文件或叫做设备节点来使用驱动程序操作字符设备和块设备。

## 字符设备

 参考：[Linux设备驱动之字符设备驱动（超级详细~） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/506834783)

<img src="https://pic2.zhimg.com/v2-9aca2269a704dadd0b4c56125738fd59_r.jpg" title="" alt="v2-9aca2269a704dadd0b4c56125738fd59_r" data-align="center">

如上图，在linux内核中使用cdev结构体来描述字符设备，通过其成员dev_t来定义设备号，以确定字符设备的唯一性。通过其成员file_operations来定义字符设备驱动提供给VFS的接口函数，如常见的open()、read()、write()等。

* 在Linux字符设备驱动中，模块加载函数通过register_chrdev_region( ) 或alloc_chrdev_region( )来静态或者动态获取设备号，通过cdev_init( )建立cdev与file_operations之间的连接，通过cdev_add( )向系统添加一个cdev以完成注册。模块卸载函数通过cdev_del( )来注销cdev，通过unregister_chrdev_region( )来释放设备号。
* 用户空间访问该设备的程序通过Linux系统调用，如open( )、read( )、write( )，来“调用”file_operations来定义字符设备驱动提供给VFS的接口函数。

**字符驱动模版**

<img src="https://pic4.zhimg.com/v2-3d0873b5d3ea11d371a7a081f80fab73_r.jpg" title="" alt="v2-3d0873b5d3ea11d371a7a081f80fab73_r" data-align="center">

这张图基本表示了字符驱动所需要的模板，只是缺少class的相关内容，class主要是用来自动创建设备节点的，还有就是一个比较常用的ioctl()函数没有列在上边。

1. 驱动初始化
   
   1. 分配cdev
      
      - 在2.6的内核中使用cdev结构体来描述字符设备，在驱动中分配cdev,主要是分配一个cdev结构体与申请设备号。
        例如：
        
        ```c
        /* 分配cdev*/
        struct cdev btn_cdev;
        /*……*/
        /* 1.1 申请设备号*/
        if(major){
            //静态
            dev_id = MKDEV(major, 0);
            register_chrdev_region(dev_id, 1, "button");
        } else {
            //动态
            alloc_chardev_region(&dev_id, 0, 1, "button");
            major = MAJOR(dev_id);
        }
        ```
      
      - 从上面的代码可以看出，申请设备号有动静之分，其实设备号还有主次之分
      
      - 在Linux中以主设备号用来标识与设备文件相连的驱动程序。次编号被驱动程序用来辨别操作的是哪个设备。cdev 结构体的 dev_t 成员定义了设备号，为 32 位，其中高 12 位为主设备号，低20 位为次设备号
      
      - 设备号的获得与生成：
        
        ```c
        获得：主设备号：MAJOR(dev_t dev);
        次设备号：MINOR(dev_t dev);
        生成：MKDEV(int major,int minor);
        ```
      
      - 设备号申请的动静之分：
      
      - 静态
        
        ```c
        int register_chrdev_region(dev_t from, unsigned count, const char *name)；
        /*功能：申请使用从from开始的count 个设备号(主设备号不变，次设备号增加）*/
        ```
      
      - 动态申请简单，易于驱动推广，但是无法在安装驱动前创建设备文件（因为安装前还没有分配到主设备号）
   
   2. 初始化cdev
      
      ```c
      void cdev_init(struct cdev *, struct file_operations *); 
      cdev_init()函数用于初始化 cdev 的成员，并建立 cdev 和 file_operations 之间的连接。
      ```
   
   3. 注册cdev
      
      ```c
      int cdev_add(struct cdev *, dev_t, unsigned);
      cdev_add()函数向系统添加一个 cdev，完成字符设备的注册。
      ```
   
   4. 硬件初始化
      
      - 硬件初始化主要是硬件资源的申请与配置，主要涉及地址映射，寄存器读写等相关操作，每一个开发板不同都有不同的实现方式，这里以FS4412为例：
        
        ```c
            /* 地址映射 */
            gpx2con = ioremap(GPX2CON, 4);
            if (gpx2con == NULL)
            {
                printk("gpx2con ioremap err\n");
                goto err3;
            }
            gpx2dat = ioremap(GPX2DAT, 4);
            if (gpx2dat == NULL)
            {
                printk("gpx2dat ioremap err\n");
                goto err4;
            }
        
            gpx1con = ioremap(GPX1CON, 4);
            gpx1dat = ioremap(GPX1DAT, 4);
        
            gpf3con = ioremap(GPF3CON, 4);
            gpf3dat = ioremap(GPF3DAT, 4);
        
            writel(readl(gpx2con)&(~(0xf<<28))|(0x1<<28), gpx2con);
            writel((0x1<<7), gpx2dat);
            writel(readl(gpx1con)&(~(0xf<<0))|(0x1<<0), gpx1con);
            writel((0x1<<0), gpx1dat);
            writel(readl(gpf3con)&(~(0xf<<16))|(0x1<<16), gpf3con);
            writel(readl(gpf3dat)|(0x1<<4), gpf3dat);
            writel(readl(gpf3con)&(~(0xf<<20))|(0x1<<20), gpf3con);
            writel(readl(gpf3dat)|(0x1<<5), gpf3dat);
        ```

2. 设备操作
   
   - 用户空间的程序以访问文件的形式访问字符设备，通常进行open、read、write、close等系统调用。而这些系统调用的最终落实则是file_operations结构体中成员函数，它们是字符设备驱动与内核的接口。以FS4412的LED驱动为例：
     
     ```c
     struct file_operations hello_fops = {
         .owner = THIS_MODULE,
         .open = hello_open,
         .release = hello_release,
         .read = hello_read,
         .write = hello_write,
     };
     ```
   
   - 上面代码中的hello_open、hello_write、hello_read是要在驱动中自己实现的
   1. open()
      
      - 原型：
        
        ```c
        int(*open)(struct inode *, struct file*); /*打开*/
        ```
   
   2. read()
      
      - 原型：
        
        ```c
        ssize_t(*read)(struct file *, char __user*, size_t, loff_t*); 
        /*用来从设备中读取数据，成功时函数返回读取的字节数，出错时返回一个负值*/
        ```
   
   3. write()
      
      - 原型
        
        ```c
        ssize_t(*write)(struct file *, const char__user *, size_t, loff_t*);
        /*向设备发送数据，成功时该函数返回写入的字节数。如果此函数未被实现，
        当用户进行write()系统调用时，将得到-EINVAL返回值*/
        ```
   
   4. close()
      
      - 原型
        
        ```c
        int(*release)(struct inode *, struct file*); 
        /*关闭*/
        ```
   
   5. ioctl()
      
      - 原型
        
        ```c
        int ioctl(int fd, ind cmd, …)；
        ```
      
      - 这里主要说一下ioctl函数。ioctl是设备驱动程序中对设备的I/O通道进行管理的函数。所谓对I/O通道进行管理，就是对设备的一些特性进行控制，例如串口的传输波特率、马达的转速等等
      
      - 其中fd就是用户程序打开设备时使用open函数返回的文件标示符，cmd就是用户程序对设备的控制命令，至于后面的省略号，那是一些补充参数，一般最多一个，有或没有是和cmd的意义相关的
      
      - ioctl函数是文件结构中的一个属性分量，就是说如果你的驱动程序提供了对ioctl的支持，用户就可以在用户程序中使用ioctl函数控制设备的I/O通道
      
      - 如果不用ioctl的话，也可以实现对设备I/O通道的控制，但那就是蛮拧了。例如，我们可以在驱动程序中实现write的时候检查一下是否有特殊约定的数据流通过，如果有的话，那么后面就跟着控制命令（一般在socket编程中常常这样做）。但是如果这样做的话，会导致代码分工不明，程序结构混乱，程序员自己也会头昏眼花的
      
      - 在驱动程序中实现的ioctl函数体内，实际上是有一个switch{case}结构，每一个case对应一个命令码，做出一些相应的操作。怎么实现这些操作，这是每一个程序员自己的事情，因为设备都是特定的，这里也没法说。关键在于怎么样组织命令码，因为在ioctl中命令码是唯一联系用户程序命令和驱动程序支持的途径
   
   6. 补充说明
      
      1. 在Linux字符设备驱动程序设计中，有3种非常重要的数据结构：struct file、struct inode、struct file_operations
         
         - struct file 代表一个打开的文件。系统中每个打开的文件在内核空间都有一个关联的struct file。它由内核在打开文件时创建, 在文件关闭后释放。其成员loff_t f_pos 表示文件读写位置
         
         - struct inode 用来记录文件的物理上的信息。因此,它和代表打开文件的file结构是不同的。一个文件可以对应多个file结构,但只有一个inode结构。其成员dev_t i_rdev表示设备号
         
         - struct file_operations 一个函数指针的集合，定义能在设备上进行的操作。结构中的成员指向驱动中的函数,这些函数实现一个特别的操作, 对于不支持的操作保留为NULL
      
      2. 在read( )和write( )中的buff 参数是用户空间指针。因此,它不能被内核代码直接引用，因为用户空间指针在内核空间时可能根本是无效的——没有那个地址的映射。因此，内核提供了专门的函数用于访问用户空间的指针
         
         ```c
         unsigned long copy_from_user(void *to, const void __user *from, unsigned long count);
         unsigned long copy_to_user(void __user *to, const void *from, unsigned long count);
         ```

3. 驱动注销
   
   1. 在字符设备驱动模块卸载函数中通过cdev_del()函数向系统删除一个cdev，完成字符设备的注销
      
      ```c
      /*原型：*/
      void cdev_del(struct cdev *);
      /*例：*/
      cdev_del(&btn_cdev);
      ```
   
   2. 释放设备号
      
      - 在调用cdev_del()函数从系统注销字符设备之后，unregister_chrdev_region()应该被调用以释放原先申请的设备号
        
        ```c
        /*原型：*/
        void unregister_chrdev_region(dev_t from, unsigned count);
        /*例：*/
        unregister_chrdev_region(MKDEV(major, 0), 1);
        ```

4. 字符设备驱动基础
   
   1. cdev结构体
   - 前面写到如何向系统申请一个设备号，设备号就像我们的身份证号一样，号本身并没有什么特殊的意义，只有把这个号和人对应才有意义，通用设备号也需要和一个特殊的东西对于，这就是cdev, cdev是linux下抽象出来的一个用来描述一个字符设备的结构体，在linux下定义如下：
     
     ```c
     struct cdev {
                     struct kobject kobj;
                     struct module *owner;
                     const struct file_operations *ops;
                     struct list_head list;
                     dev_t dev;
                     unsigned int count;
     };
     ```
   
   - 结构体中有几个成员事我们写驱动的时候必须关心的：
     dev 类型是dev_t,也就是我们的设备号 ，ops是一个同样也是一个结构体并且是一个字符驱动实现的主体，字符驱动通常需要和应用程序交互
   
   - cdev 结构体的dev_t 成员定义了设备号，为32位，其中12位是主设备号，20位是次设备号，我们只需使用二个简单的宏就可以从dev_t 中获取主设备号和次设备号
     
     ```c
     MAJOR(dev_t dev)
     MINOR(dev_t dev)
     //相反地，可以通过主次设备号来生成dev_t：
     MKDEV(int major,int minor)
     ```
   2. Linux 2.6内核提供一组函数用于操作cdev 结构体
      
      ```c
      void cdev_init(struct cdev*,struct file_operations *);
      struct cdev *cdev_alloc(void);
      int cdev_add(struct cdev *,dev_t,unsigned);
      void cdev_del(struct cdev *);
      ```
      
      - 其中（1）用于初始化cdev结构体，并建立cdev与file_operations 之间的连接。（2）用于动态分配一个cdev结构，（3）向内核注册一个cdev结构，（4）向内核注销一个cdev结构
      
      - cdev_add实现cdev的注册，linux内核里维护了一个cdev_map的表，所谓cdev的注册就是把我们的cdev注册到cdev_map表上,cdev_map表结构如图:
        
        <img src="https://pic4.zhimg.com/v2-be00f501599f5ba9a3af64d4cce83493_r.jpg" title="" alt="v2-be00f501599f5ba9a3af64d4cce83493_r" data-align="center">
   
   3. Linux 2.6内核分配和释放设备号
      
      - 在调用cdev_add()函数向系统注册字符设备之前，首先应向系统申请设备号，有二种方法申请设备号，一种是静态申请设备号：
        
        ```c
        int register_chrdev_region(dev_t from,unsigned count,const char *name)
        ```
      
      - 另一种是动态申请设备号：
        
        ```c
        int alloc_chrdev_region(dev_t *dev,unsigned baseminor,unsigned count,const char *name);
        ```
      
      - 其中，静态申请是已知起始设备号的情况，如先使用cat /proc/devices 命令查得哪个设备号未事先使用（不推荐使用静态申请）；动态申请是由系统自动分配，只需设置major = 0即可
      
      - 相反地，在调用cdev_del()函数从系统中注销字符设备之后，应该向系统申请释放原先申请的设备号，使用:void unregister_chrdev_region(dev_t from,unsigned count);
   
   4. cdev结构的file_operations结构体
      
      - 这个结构体是字符设备当中最重要的结构体之一，file_operations 结构体中的成员函数指针是字符设备驱动程序设计的主体内容，这些函数实际上在应用程序进行Linux 的 open()、read()、write()、close()、seek()、ioctl()等系统调用时最终被调用

5. 自动创建设备节点
   
   1. 使用device_creat创建设备
      
      ```c
      struct device *device_create(struct class *cls,struct device *parent, dev_t devt, void *drvdata, const char *fmt,...)
      
      //device_create是个可变参数函数，参数 cls就是设备要创建哪个类下面
      //就是设备要创建哪个类下面,参数 parent是父设备，一般为 NULL
      //devt设备号
      //drvdata是可能用到的数据
      //fmt是设备名字
      
      
      void device_destroy(struct class *cls, dev_t devt)
      //删除创建的类
      ```
   
   2. 参考用例如下
      
      ```c
      struct class *class; /* 类 */ 
      struct device *device; /* 设备 */ 
      dev_t devid; /* 设备号 */
      
      /* 驱动入口函数 */ 
      static int __init xxx_init(void)
      {
          /* 创建类 */
          class = class_create(THIS_MODULE, "xxx");
          /* 创建设备 */
          device = device_create(class, NULL, devid, NULL, "xxx");
          return 0;
      }
      
      /* 驱动出口函数 */
      static void __exit led_exit(void)
      {
          /* 删除设备 */
          device_destroy(newchrled.class, newchrled.devid);
          /* 删除类 */
          class_destroy(newchrled.class);
      }
      
      module_init(led_init); 25 module_exit(led_exit);
      ```
      
      
   
   
































