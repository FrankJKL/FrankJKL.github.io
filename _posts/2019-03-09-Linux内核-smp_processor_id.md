---
layout: post
title:  "smp_processor_id()获取当前执行cpuid"
categories: Linux内核
tags: smp_processor_id percpu变量
typora-root-url: ..
---

基于Linux 2.6.32内核进行分析，看本篇文章前，建议先看看[percpu变量](https://frankjkl.github.io/2019/03/08/Linux%E5%86%85%E6%A0%B8-percpu%E5%8F%98%E9%87%8F/)这篇文章

smp_processor_id()用来获取当前cpu的id，首先来看smp_processor_id的定义：

```c
# define smp_processor_id() raw_smp_processor_id()
```

接下来：

```c
#define raw_smp_processor_id() (percpu_read(cpu_number))
```





有必要解释一下这里的cpu_number的由来：

```c
DEFINE_PER_CPU(int, cpu_number);//每个CPU的cpuid是放置在cpu_number这个percpu变量中
```

后面最终调用了：

```c
percpu_from_op("mov", per_cpu__##var, "m" (per_cpu__##var))
// 这里的var为cpu_number
```

这个宏展开后，实质上等价于：

```c
#define percpu_from_op(op, var, constraint)
({                                                        
    typeof(per_cpu_cpu_number) ret__;
    switch (sizeof(per_cpu_cpu_number)) { 
    case 4:
        asm(“movl  %%fs:%P1, %0" //在64位体系结构下为 movl  %%gs:%P1, %0 
            : "=r" (ret__)
            : "m" (per_cpu_cpu_number));
        break;
        ...
})
```

这里fs寄存器里面是什么东西？为什么这里就可以获取到cpu的id？请看完后文后到这里进行回答：

smp_processor_id实际要去读取的是cpu_number这个percpu变量，因为在系统初始化的时候，把每个cpu的id都设置到了cpu_number这个percpu变量中，请看代码：

```c
setup_per_cpu_areas()
{
    for_each_possible_cpu(cpu) {
        .....
    	per_cpu(cpu_number, cpu) = cpu;//看到没，在这里设置进去了，等下我们就要去取里面的内容
        setup_percpu_segment(cpu);
        .....
    }
} 
```

至于per_cpu的实现，请参考文章开头给出的那篇文章。总之这里就是把各个CPU的cpuid设置到了cpu_number这个percpu变量中。这里是为了给大家说明：我们不是要去cpu_number里面取cpuid吗？那什么时候放进去的呢。现在大家明白了吧。

接下来正真看fs寄存器里有什么东西：

```c
//在/arch/x86/kernel/head_32.S中
movl $(__KERNEL_PERCPU), %eax
movl %eax,%fs //fs段寄存器指向GDT的27偏移项
```

那GDT的27偏移项描述符是谁呢？

```c
static inline void setup_percpu_segment(int cpu)
{
#ifdef CONFIG_X86_32
        struct desc_struct gdt;

        pack_descriptor(&gdt, per_cpu_offset(cpu), 0xFFFFF,
                        0x2 | DESCTYPE_S, 0x;
        gdt.s = 1;
        write_gdt_entry(get_cpu_gdt_table(cpu),
                        GDT_ENTRY_PERCPU, &gdt, DESCTYPE_S);
#endif
}
```

这里设置的在 GDT 表中的偏移 GDT_ENTRY_PERCPU 与 前面fs的\_\_KERNEL_PERCPU一样都为27，所以fs段选择子对应的GDT描述符项基地址为per_cpu_offset(cpu)。也就是说每个CPU的fs段选择子都是一样的27，但是设置的段描述符的基地址每个CPU都是不一样的（因为base指定为per_cpu_offset(cpu)）。

再回到最开始的嵌入式汇编：

```c
asm("movb %%fs:%P1, %0"
        :"=q"(ret__)
        :"m"(per_cpu__cpu_number));
```

不难看出：per_cpu_offset[cpu] + &per_cpu\_\_cpu_number 便得到了相应 cpu 的每 per_cpu\_\_cpu_number变量的地址，与正常percpu变量存取方式不同，cpu_number的offset是通过GDT表项得到的。关于 per_cpu_offset数组的由来 请参考 setup_per_cpu_areas 函数。

整个smp_processor_id()取CPUid的示意图：

![1552126437225](/assert/1552126437225.png)



FQA：

> 在start_kernel的开始调用boot_cpu_init，且boot_cpu_init在setup_per_cpu_areas()之前调用，在这个函数里面，已经在使用smp_processor_id了！但是此时运行时per_cpu_areas还没初始化，这是怎么回事？

这是因为此时的fs引用的GDT描述符的base为0（此时GDT的GDT_ENTRY_PERCPU描述符base为0，参考arch/x86/kernel/cpu/common.c：DEFINE_PER_CPU_PAGE_ALIGNED），smp_processor_id()直接引用cpu_number的地址来获取cpuid，cpu_number在编译时分配到.data.percpu段，所以此时smp_processor_id返回为0。且此时 AP （应用处理器）还没有启动，BP（引导处理器）完成启动，所以就是 cpu 0。



> 2.6.24内核中运行时cpu_number的值是在哪里初始化的？percpu变量专用的GDT描述符项在哪里设置的？

```c
/* Initialize the CPU's GDT.  This is either the boot CPU doing itself
   (still using the master per-cpu area), or a CPU doing it for a
   secondary which will soon come up. */
__cpuinit void init_gdt(int cpu)
{
	struct desc_struct *gdt = get_cpu_gdt_table(cpu);

	pack_descriptor((u32 *)&gdt[GDT_ENTRY_PERCPU].a,//依然在27（GDT_ENTRY_PERCPU）偏移项
			(u32 *)&gdt[GDT_ENTRY_PERCPU].b,
			__per_cpu_offset[cpu], 0xFFFFF,
			0x80 | DESCTYPE_S | 0x2, 0x8);

	per_cpu(this_cpu_off, cpu) = __per_cpu_offset[cpu];
	per_cpu(cpu_number, cpu) = cpu;
}
```



参考整个系统GDT：

*  ------- start of kernel segments:
  *
*  12 - kernel code segment                <==== new cacheline
*  13 - kernel data segment
*  14 - default user CS
*  15 - default user DS
*  16 - TSS
*  17 - LDT
*  18 - PNPBIOS support (16->32 gate)
*  19 - PNPBIOS support
*  20 - PNPBIOS support
*  21 - PNPBIOS support
*  22 - PNPBIOS support
*  23 - APM BIOS support
*  24 - APM BIOS support
*  25 - APM BIOS support
  *
*  26 - ESPFIX small SS
*  27 - per-cpu                        [ offset to per-cpu data area ]
*  28 - stack_canary-20                [ for stack protector ]
*  29 - unused
*  30 - unused
*  31 - TSS for double fault handler