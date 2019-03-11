---
layout: post
title:  "同步与互斥_percpu变量"
categories: Linux内核
tags: 同步与互斥 percpu变量
typora-root-url: ..
---

本篇文章基于Linux 2.6.24

percpu变量的关键就是：**要求根据CPU的个数，在内存中生成多份拷贝，并且能够根据变量名和CPU编号，正确的对各个CPU的变量进行寻址。**

采用per-cpu变量有下列好处：所需数据很可能存在于处理器的缓存中，因此可以更快速地访问。如果在多处理器系统中多个CPU可能同时访问变量，会引发一些通信方面的问题，采用percpu变量恰好规避了这个问题。




这里读者只需要记住percpu变量每个CPU都有一个就行了，看完本篇文章后，再回来阅读前面加粗的话，相信会有更深刻的理解。

首先来看如何定义一个percpu变量：

```c
DEFINE_PER_CPU(int, cpu_number);

#define DEFINE_PER_CPU(type, name) \
    __attribute__((__section__(".data.percpu"))) __typeof__(type) per_cpu__##name
```

上面的代码定义了一个类型为int，名字为per_cpu_cpu_number的percpu变量。\_\_typeof\_\_(type)得到的就是int，而per_cpu\_\_##name得到的就是per_cpu_cpu_number，并且通过\_\_section\_\_把这个变量放到名为.data.percpu的数据段。因此在编译时所有的percpu变量都会统一放到.data.percpu数据段，当内核启动后，会根据检测到的cpu个数从数据段.data.percpu为各个CPU单独复制一份。**start_kernel**函数调用**setup_per_cpu_areas()**来完成这个工作，**setup_per_cpu_areas**定义如下：

```c
//__per_cpu_offset数组用来保存CPU对应 percpu段 相对ptr的偏移
unsigned long __per_cpu_offset[NR_CPUS] __read_mostly;

static void __init setup_per_cpu_areas(void)
{
	unsigned long size, i;
	char *ptr;
    //取CPU的数量
	unsigned long nr_possible_cpus = num_possible_cpus();

	/* Copy section for each CPU (we discard the original) */
    /* 计算每个CPU所需要的percpu段大小。
     *
     * @PERCPU_ENOUGH_ROOM：__per_cpu_end - __per_cpu_start + PERCPU_MODULE_RESERVE
     * @ALIGN()：使percpu段大小向上对齐
     */
	size = ALIGN(PERCPU_ENOUGH_ROOM, PAGE_SIZE);
	//分配内存
    ptr = alloc_bootmem_pages(size * nr_possible_cpus);

	for_each_possible_cpu(i) {
        /* __per_cpu_start对于每个CPU来讲是个定值，它和__per_cpu_end都是由链接器生成的
         * 计算每个CPU的__per_cpu_start相对于真实存放percpu段线性地址 的偏移
         * 请参考代码下方的示意图
         * memcpy拷贝所有percpu变量至每个CPU自己的percpu段
         */
		__per_cpu_offset[i] = ptr - __per_cpu_start;
		memcpy(ptr, __per_cpu_start, __per_cpu_end - __per_cpu_start);
		ptr += size;
	}
}
```

![1552095558772](/assert/1552095558772.png)



**那么在编译期源程序如何访问percpu变量呢**？内核定义了几个宏来实现对percpu变量的访问：

```c
//smp_processor_id()可以返回当前活动处理器的ID，用作下面的cpu参数。
#define per_cpu(var, cpu) (*({				\
	extern int simple_identifier_##var(void);	\
	RELOC_HIDE(&per_cpu__##var, __per_cpu_offset(cpu)); }))
```

这里我们举一个例子进行说明：假设访问per_cpu(var，1)，由于percpu变量在定义时由DEFINE_PER_CPU展开成per_cpu_var，因此RELOC_HIDE的第一个参数就是编译时per_cpu_var的地址，第二个参数__per_cpu_offset(cpu)为前面setup_per_cpu_areas计算出来的，它表示运行时复制的percpu数据区首地址与编译期确定的首地址的差值，这两个值相加就是实际存储percpu变量的地址。示意图如下：

![1552104630074](/assert/1552104630074.png)

在更（geng）新的内核（2.6.32）中，percpu变量的定义（DEFINE_PER_CPU）和存取（per_cpu(var, cpu)）方式没有发生改变，主要是\_\_per\_cpu\_offset数组的初始化方式发生了改变，见arch\x86\kernel\setup_percpu.c中函数：setup_per_cpu_areas()。同时可以参考如下链接：

http://www.voidcn.com/article/p-selydofa-na.html



