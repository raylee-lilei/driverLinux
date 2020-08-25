# Linux驱动开发学习

### 驱动的概念：

​			软件驱动：驱动硬件，使得 硬件处于某种工作模式，提供 控制硬件的方案

### 驱动的地位：

​			驱动是连接内核与设备的桥梁

![image-20200825110023551](https://gitee.com/raylee-lilei/cdn/raw/master/image-20200825110023551.png)

## 设备分类

#### 字符设备

​			IO传输是以字符进行传输

​			对字符设备读写时，实际硬件一般紧接着发生（同步）

​			LCD  鼠标  键盘  触摸屏

#### 网络设备

​			网卡驱动

#### 块设备

​			以块（4K）为单位传输

​			读写时实际硬件一般不会紧接着发生（异步）

​             磁盘类、闪存

### 字符设备驱动编写

​			驱动编写

​			驱动编译

​			驱动使用



**模块化，动态加载删除模块**

### 模块编写

​			入口（加载）：module_init(入口函数名)

​								int  __init  XX_func(void){}

​			出口（卸载）: module_exit(卸载函数名)

​								void  __exit XX_func(void){}

​			GPL协议声明 :  MODULE_LICENSE("GPL")

**编写**

```C++
#include <linux/init.h>
#include <linux/module.h>

int __init demo_init(void){
    printk("----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
    return 0;
}

void __exit  demo_exit(void){
     printk("----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
}

module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```

**编译**

内部编译：将内核模块源文件放在内核源码中进行编译 kconfig , Makefile, make menuconfig

静态编译：将内核模块编译到Image镜像中



外部编译：将内核模块源文件放在内核源码外进行编译

动态编译：编译 生成动态模块XXX.ko



Makefile:

```
KERNDIR:= /lib/modules/3.2.0-29.../build/
PWD:=$(shell pwd)
obj-m:=demo.o

all:
   make -C $(KERNDIR) M=$(PWD) modules
   
clean:
   make -C $(KERNDIR) M=$(PWD) clean
```

**模块使用**

查看内核模块信息的命令

 		modinfo      XXX.ko

将内核模块加载到内核中，和内核形成一个整体运行

​		查看当前内核中已经插入的动态模块

​					lsmod

​		查看内核的日志信息命令

​					dmesg

 					-c  清除内核日志信息

​		insmod   XXX.ko

将内核中的内核模块，从内核中卸载出来

​		rmmod XXX 



内核模块加载的时候执行加载函数，并且只会执行一次

内核模块卸载的时候执行卸载函数，并且只会执行一次





### 字符块设备驱动

字符设备和块设备有设备文件，网络设备没有设备文件

![image-20200825171216095](https://gitee.com/raylee-lilei/cdn/raw/master/image-20200825171216095.png)





描述所有字符设备驱动的结构体：

cdev

struct cdev{

​		struct module * owner;

​		const struct file_operations  *ops;

​		dev_t  dev;	

​		unsigned int count;	

​		struct list_head  list;

}



1.操作方法集   提供给应用层的方法集

struct file_operations  {

​		read

​		write

​		fasync

​		open

​		release	

​		poll

}



2. dev_t  dev 设备号，唯一标识设备

   ​		32位无符号整形

   **主设备号**（USB）

   ​		MAJOR(dev_t  dev)     在设备号中提取主设备号

   ​		MINOR(dev_t dev)     在设备号中提取次设备号

   ​        MKDEV（int ma,  int mi）   主次生成设备号

   **次设备号**（USB1   USB2 ...）

   

   #define  MAJOR(dev)   ((unsigned  int)   ((dev) >> 20))

   右移20得到高12位

   #define  MINOR(dev)   ((unsigned  int)   ((dev) &((1U << 20) -1)))

   得到低20位



**编写字符设备驱动**

0. 分配设备号，注册设备

   1. 自动分配

      /*********************************************************************************************************************************************

      *功能：分配设备号

      *参数： 

      * ​                   @dev    dev_t 类型的变量  取地址
      * ​                   @baseminor    次设备号起始
      * ​                   @count          个数
      * ​                   @ name          名字

      *返回值：成功返回0  ， 失败返回复数错误码

      ***********************************************************************************************************************************************/

      int  alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count,const char *name)

   2. 指定设备号注册

      /*********************************************************************************************************************************************

      *功能：指定设备号注册

      *参数： 

      * ​                   @from    设备号（MKDEV（major，minor））
      * ​                   @count          个数
      * ​                   @ name          名字

      *返回值：成功返回0  ， 失败返回复数错误码

      ***********************************************************************************************************************************************/

      int  register_chrdev_region(dev_t  from,unsigned count,const char *name)

   3. 注销设备号

      /*********************************************************************************************************************************************

      *功能：注销设备号

      *参数： 

      * ​                   @from    设备号
      * ​                   @count         个数

      ***********************************************************************************************************************************************/

      void   unregister_chrdev_region(dev_t   from, unsigned   count)

1.  cdev 结构体分配内存空间

   ```
   		/**********************************************************************************************************
   
   ​		*功能：为cdev结构体分配内存空间
   
   ​		*参数:
   
   ​		*        @ void
   
   ​		*
   
   ​		*返回值：    成功返回分配到的结构体地址
           *          失败返回NULL
   
   ​		*******************************************************************************************************************/
   
   		struct cdev* cdev_alloc(void)
   ```

2. 初始化cdev结构体

   		/**********************************************************************************************************
   	
   		*功能：初始化cdev结构体
   	
   		*参数:
   	
   		*        @cdev cdev 结构体指针
   		*        @fops 操作方法集指针
   	
   		*返回值：  void
   		*******************************************************************************************************************/
   	
   		void cdev_init(struct cdev *cdev, const struct file_operations *fops)

3. 添加字符设备到内核中，由内核统一管理

   	/**********************************************************************************************************
   	
   	*功能：初始化cdev结构体
   	
   	*参数:
   	
   	*        @cdev cdev 结构体指针
   	*        @dev  设备号
   	*        @count 设备个数
   	
   	*返回值：  成功放回0   失败返回 error
   	*******************************************************************************************************************/
   	
   	int cdev_add(struct cdev *p, dev_t dev ,unsigned count)

4. 删除字符设备

   ```
   void cdev_del(struct cdev *p)
   ```

   

```C++
#include <linux/init.h>
#include <linux/module.h>
#include <linux/cdev.h>
#include <linux/fs.h>

#define BASEMINOR 0
#define COUNT 3
#define NAME "dev_demo"

struct cdev* cdevp =NULL;

int demo_open(struct inode *inode, struct file *filp){
    printk(KERN_INFO "----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
    return 0;
}
int demo_release(struct inode *inode, struct file *filp){
    printk(KERN_INFO "----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
    return 0;
}
struct file_operations fops ={
    .owner = THIS_MODULE,
    .open = demo_open,
    .release = demo_release,
};

dev_t devno;

int __init demo_init(void){
    
    //1.分配设备号
    int ret = 0;
    ret = alloc_chrdev_region(&devno, BASEMINOR, COUNT,NAME);
    if(ret < 0){
        printf(KERN_ERR "alloc_chrdev_region   failed... \n");
        goto err1;
    }
    printk(KERN_ERR "major = %d \n",MAJOR(devno));
    
    //2.分配cdev结构体
    cdevp = cdev_alloc();
    if(NULL ==cdevp){
        printf(KERN_ERR "cdev_alloc failed... \n");
        ret = -ENOMEM;
    goto err2;
    }
    
    //3.cdev_init
    cdev_init(cdevp, &fops);
    
    //4. cdev_add
    ret = cdev_add(cdevp,devno ,COUNT);
    if(ret <0){
        printf(KERN_ERR "cdev_add failed... \n");
        goto err2;
    }
    
    printk("----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
    return 0;

err1:
    return ret;
err2:
    unregister_chrdev_region(devno , COUNT);
}

void __exit  demo_exit(void){
     cdev_del(cdevp);
     unregister_chrdev_region(devno , COUNT);
     printk("----%s----%s----%d------\n",__FILE__,__func__,__LINE__);
}

module_init(demo_init);
module_exit(demo_exit);
MODULE_LICENSE("GPL");
```





