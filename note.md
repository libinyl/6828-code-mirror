# 验证笔记

warm boot 前:

```
┌──Register group: general───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│eax            0xffffff40          -192            ecx            0x0                 0               edx            0xffffff40          -192            ebx            0x10074             65652           │
│esp            0x7bec              0x7bec          ebp            0x7bf8              0x7bf8          esi            0x10074             65652           edi            0x0                 0               │
│eip            0x10000c            0x10000c        eflags         0x46                [ PF ZF ]       cs             0x8                 8               ss             0x10                16              │
│ds             0x10                16              es             0x10                16              fs             0x10                16              gs             0x10                16              │
```

warm boot 后:

```
──Register group: general───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│eax            0xffffff40          -192            ecx            0x0                 0               edx            0xffffff40          -192            ebx            0x10074             65652           │
│esp            0x7bec              0x7bec          ebp            0x7bf8              0x7bf8          esi            0x10074             65652           edi            0x0                 0               │
│eip            0x100015            0x100015        eflags         0x46                [ PF ZF ]       cs             0x8                 8               ss             0x10                16              │
│ds             0x10                16              es             0x10                16              fs             0x10                16              gs             0x10                16              │
```

除了 eip 正常增长外没有变化.


warm boot 之前:
```
EAX=ffffff40 EBX=00010074 ECX=00000000 EDX=ffffff40
ESI=00010074 EDI=00000000 EBP=00007bf8 ESP=00007bec
EIP=0010000c EFL=00000046 [---Z-P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
SS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
DS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
FS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
GS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00007c4c 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
```
warm boot 之后:

```
EAX=ffffff40 EBX=00010074 ECX=00000000 EDX=ffffff40
ESI=00010074 EDI=00000000 EBP=00007bf8 ESP=00007bec
EIP=00100015 EFL=00000046 [---Z-P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
CS =0008 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
SS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
DS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
FS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
GS =0010 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     00007c4c 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
```
除 eip 外没有变化.

设置 cr3 之后:

```
GDT=     00007c4c 00000017
IDT=     00000000 000003ff
CR0=00000011 CR2=00000000 CR3=00119000 CR4=00000000
```

使能 cr0 分页之后:

```
CR0=80010011 CR2=00000000 CR3=00119000 CR4=00000000
```

当前使能 paging 的意义在于,无需再访问每个变量的时候都RELOC.

用汇编设置完整的内存映射实在辛苦,所以先暂时建立当前的一小段映射.

当前的一小段建立多少? kernel 总共 4MB,所以映射一页就足够了,就是 4MB.

如何映射?

如何设置 entry_pgdir?

从哪里开始映射? 从 [KERNBASE, KERNBASE+4MB) ,即 [0xf0000000, 0xf0400000).

将其视作虚拟地址,则分布如下:


pd(10) | pt(10) | offset(12)
-------|--------|-----------
11 0000 0000 | 00 0000 0000 ~ 11 1111 1111 | 00 0000 0000 ~ 11 1111 1111

映射到物理内存的哪里? 映射到 [0, 4MB).

每 4kB 作为一个 page.共需要 1024 个 page.所以entry_pgtable如下:

```c
pte_t [NPTENTRIES] = {
	0x000000 | PTE_P | PTE_W,
	0x001000 | PTE_P | PTE_W,
	0x002000 | PTE_P | PTE_W,
	0x003000 | PTE_P | PTE_W,
	0x004000 | PTE_P | PTE_W,
	...
	0x3fd000 | PTE_P | PTE_W,
	0x3fe000 | PTE_P | PTE_W,
	0x3ff000 | PTE_P | PTE_W,
	}
```

如何映射? 

- CPU 在访问虚拟地址时,取高 10 位作为 pde 的索引,所以应为`KERNBASE>>PDXSHIFT`;
- 令此索引指向页表的物理位置


```c
// 故意把 0~4MB 的映射注释掉
pde_t entry_pgdir[NPDENTRIES] = {

    // 把 0~4MB 的映射注释掉
    
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```

然后运行后gdb报错:

```
Program received signal SIGTRAP, Trace/breakpoint trap.
=> 0xf010002f <relocated>:	Error while running hook_stop:
Cannot access memory at address 0xf010002f
relocated () at kern/entry.S:75
75		movl	$0x0,%ebp			# nuke frame pointer
```

QEMU 提示`Triple fault.`.

提示无法访问`0xf010002f`的地址.为何?

因为当前已开启 paging, 而跳转`relocated`的指令位于低地址(我们现在正在直接执行物理内存中的代码,是加载进来的物理地址!)

也就是说 EIP 也指向低地址.当开启分页后,当前地址空间瞬间切换为虚拟空间,此时虚拟地址空间的低 4MB 是没有代码的!

所以自然无法访问. 

所以要把低地址也映射过去:

```c
pde_t entry_pgdir[NPDENTRIES] = {
	// Map VA's [0, 4MB) to PA's [0, 4MB)
//	[0]
//		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P,
	// Map VA's [KERNBASE, KERNBASE+4MB) to PA's [0, 4MB)
	[KERNBASE>>PDXSHIFT]
		= ((uintptr_t)entry_pgtable - KERNBASE) + PTE_P + PTE_W
};
```



开启分页前:

```
(gdb) x 0x10000c
   0x10000c:    movw   $0x1234,0x472
(gdb) x 0xf010000c
   0xf010000c <entry>:  add    %al,(%eax)
```

开启分页后:

```
(gdb) x 0xf010000c
   0xf010000c <entry>:  movw   $0x1234,0x472
(gdb) x 0x10000c
   0x10000c:    movw   $0x1234,0x472
```

映射成功~~

更完整的观察:

```
(gdb) x/15 0xf010000c
   0xf010000c <entry>:  movw   $0x1234,0x472
   0xf0100015 <entry+9>:        mov    $0x119000,%eax
   0xf010001a <entry+14>:       mov    %eax,%cr3
   0xf010001d <entry+17>:       mov    %cr0,%eax
   0xf0100020 <entry+20>:       or     $0x80010001,%eax
   0xf0100025 <entry+25>:       mov    %eax,%cr0
   0xf0100028 <entry+28>:       mov    $0xf010002f,%eax
   0xf010002d <entry+33>:       jmp    *%eax
   0xf010002f <relocated>:      mov    $0x0,%ebp
   0xf0100034 <relocated+5>:    mov    $0xf0119000,%esp
   0xf0100039 <relocated+10>:   call   0xf0100040 <i386_init>
   0xf010003e <spin>:   jmp    0xf010003e <spin>
   0xf0100040 <i386_init>:      push   %ebp
   0xf0100041 <i386_init+1>:    mov    %esp,%ebp
   0xf0100043 <i386_init+3>:    sub    $0xc,%esp
```

如何从一个 pageinfo 地址获取对应物理页的物理地址?等式关系是什么?


(pageinfo 的地址-pages 的地址)/sizof(pginfo) = pginfo索引


物理地址    = pginfo 索引*PGSIZE
           = (pg 地址-pages 地址) * 4096
           = (pg 地址-pages 地址) * 2^(12)
           = (pg 地址-pages 地址) << 12
           
内核虚拟空间地址 = 物理地址 + KERNBASE. 
 
二级页表寻址步骤:

应该掌握的问题:

从虚拟地址到物理地址的映射?


启发: 立刻开启寄存器的值然后测试.减少头脑的状态,用具体代替抽象

有时候觉得难,是因为过于抽象,离结果太远.把结果提前!



几个常用的宏描述了 物理地址/内核虚拟地址/虚拟内存地址 之间的关系:






每页分 4KB 的好处是,每个 page 4KB 的第一个地址的最后三个bit  都是 0.也就是说.

可用内存: 131072KB

分页数量(pagetable的 数量)131072/4 = 32748 page

如果一级页表,则需要 32768 个

分页逻辑梳理:

内核需要存储物理地址与虚拟地址的映射关系.


引入虚拟内存后,需要管理物理内存与虚拟内存的映射关系.对于每个虚拟内存,都应该可以找到其对应的物理内存.


虚拟地址一共 4G. cpu 一次加载一个 word,也就是 4 个 byte.

如果用 int 类型数组存储映射关系,需要的 table entry 数量是 2^30 = 1 G.

不可能直接在内核消耗 1 个 G 的内存去管理这么多空间.

现在将虚拟内存每 4KB 划分为一个 page. 只维护虚拟内存的前 20 个 bit 与物理内存的前 20 个 bit.  也就是说每个 pagetable 的内容是 VA 前 20bit]


这样一来,从虚拟内存到物理内存的计算方式为 `PA(VA) = PT[ VA前20bit ] + VA后 12bit  `

为何 4KB 的分页,可以让后 12 位视为偏移量?

因为 4096=2^12.


实验三调用链:
```
start (kern/entry.S)
i386_init (kern/init.c)
    cons_init
    mem_init
    env_init
    trap_init (still incomplete at this point)
    env_create
        env_alloc
            env_setup_vm
        load_icode
            region_alloc
    env_run
        env_pop_tf
```