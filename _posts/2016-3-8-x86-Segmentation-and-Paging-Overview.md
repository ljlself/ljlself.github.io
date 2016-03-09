---
layout: article
title: x86 Segmentation and Paging Overview
---

#x86 Segmentation and Paging Overview
###简介
x86的内存管理是段页式的，既有分段也有分页。

- Segmentation
    -  分段隔离了代码段，数据段和堆栈段等。通过这种方式多个程序可以在互不干扰的情况下正常运行。
- Paging
    - 实现了传统的内存管理需求。

###概念
Logic Address(逻辑地址)：

-  指令中出现的地址都是逻辑地址，逻辑地址并不是一个真实存在的地址，只是相对地址，需要经过转换才能成为物理地址，从而找到该地址上的内容。
-  在X86的Segmetation下，逻辑地址定义为[段标识+段内偏移量]。
-  段标识通过Selector来定义，段内偏移量称为Offset。

Linear Address(线性地址)：

- 又称为虚拟地址，是通过程序转换产生的地址。线性地址也不是一个真实的地址，这一点和逻辑地址有点像。
- 在x86的Segmentation中，进入分段之前对应的地址称为逻辑地址，而分段之后对应的地址则称为线性地址（基址+偏移量）。在Paging中，进入分页之前对应的是线性地址，进过分页之后对应的则是物理地址。
- 所有的Segments包含的地址空间，及所有的线性地址构成的地址空间，称为Linear Address Space。

###Overview
32Bit-Paging，4KBytes Page
![Overview](/image/x86_segmentation_and_paging/overview.jpg "")

###Segmentation：（Logical Address -> Linear Address）
Segment Registers

- 为了减少寻址时间和减少代码的复杂度，处理器提供了6个寄存器来存放Selector。每个寄存器都支持特定类型内存的引用（如code，stack和data）。程序要运行需要最基本的几个如代码段，数据段和堆栈段，对应了Code Segment Register，Stack Segment Register和Data Segment Register。这些寄存器负责存放对应的Segment Selector。
- 每个程序虽然可能对应非常多的Segments，但是只有6个Segments能够被立即使用。其他的段想要能够使用必须将他们的Selector放在这些Register中。
    - CS：Code Segment Register
    - SS：Stack Segment Register
    - DS：Data Segment Register
    - ES：Additional Data Register
    - FS：Additional Data Register
    - GS：Additional Data Register
- 每个Register分为两个部分
    - Visible Part：用于存放Selector
    - Hidden Part：在处理器load Selector的时候会把该Selector对应的Descriptor的基址，长度限制和访问权限等信息放在该部分，这样做的好处就是有时候处理器在寻址的时候不用再去等待一个总线周期才能访问到Descriptor中的信息。
- 操作Register的命令
    - 直接操作的指令有MOV, POP, LDS, LES, LSS, LGS和LFS。这些指令显示的引用Segment Register
    - 隐式操作的指令有CALL, JMP, RET, SYSENTER, SYSEXIT等等。这些指令有时候会改变Segment Register（大部分是指Code Segment Register）中的内容。

Selector(选择子)：

- 在x86中，要找到某个段，需要通过Selector来定位。将Selector装入6个段寄存器中的一个。每个Seletor为16位。前13位代表段描述符表项编号，如果描述符为0则说明当前段寄存器不可用，后一位代表使用LDT还是GDT。最后两位代表优先级与保护有关。
- ![Selector](/image/x86_segmentation_and_paging/selector.jpg )
    - Index: Selector的前13位，段描述符索引，表明所需要的段描述符在描述符表中的位置。
    - TI: 只占1位，值为0或1，表明需要定位的段描述符在GDT中还是在LDT中。
    - RPL(Request Privilege Level)：代表选择子的特权等级，共0-3四个特权等级。

Memory Management Registers

- GDTR
    - 保存GDT信息的寄存器。包含32位GDT基址信息和16位长度限制。
    - 通过LGDT和SGDT指令来加载和保存GDT的入口地址，CPU根据此入口地址来访问GDT。
- LDTR
    - LDTR记录LDT的起始地址。即一个32位的段基址，一个长度限制和其他一些属性。
    - LDTR中还包含了一个16位的Selector，用于找到在该LDT中对应的的段描述符。


Descroptor Table（段描述符表）：

- 段描述符表不是段本身，而只是线性地址空间中的一个数据结构。包含所有的段描述符，长度可变。最大长度为8192个段描述符（Selector中的13位），每个8字节。包含两种类型的描述符表LDT和GDT。
- LDT(Local Descriptor Table)：
    - 局部描述符表，描述每个程序的代码，数据和堆栈段。每个程序都有一个。
    - 因为LDT本身也是一个段，所以相应的会有一个段描述符来描述它。而这个描述符是存放在GDT中的。如果系统支持LDTs，那么每一个LDT在GDT中都有一个独立的段描述符。
    - LDT是由它的Selector来定位的，为了减少转换的消耗，LDT段的信息存放在LDTR中。
    - Local Descriptor的访问是先通过Selector在GDT中寻找，找到LDTR的地址后再根据LDTR中的信息找到对应的Local Descriptor。
- GDT(Global Descriptor Table)：
    - 全局描述符表，描述系统等。一个处理器只有一个，所有程序共享一个。GDT的基址和长度上限存放在寄存器GDTR中，作为入口。
- ![Descriptor Table](/image/x86_segmentation_and_paging/descriptor_table.jpg)

Descriptor（段描述符)：

- 存放在段描述符表（即GDT和LGT）中的数据结构。大小为一个双字，64位。保存着有关段的全部信息。
- ![Descriptor](/image/x86_segmentation_and_paging/descriptor.jpg)
- S位
    - [word 2]bit 12
    - 描述符分为两种，由描述符中的S位表示
    - S=0，则表明该描述符是系统描述符（System Descriptor），又分为两种
        - System Segment Descriptor
            - Local descriptor-table (LDT) segment descriptor.
            - Task-state segment (TSS) descriptor.
        - Gate Descriptor
            - Call-gate descriptor.
            - Interrupt-gate descriptor.
            - Trap-gate descriptor.
            - Task-gate descriptor.
    - S=1，则表明该描述符是Code and Data Descriptor   
- Type位
    - [word 2]bit 8 : bit 11
    - Code and Data Descriptor
        - 由4位构成，最高位11位确定Descriptor到底是Code Descriptor还是Data Descriptor，8，9，10三位分别代表了accessed (A), write-enable (W), and expansion-direction (E)。
        - ![Code and Data Descriptor](/image/x86_segmentation_and_paging/code_and_data_descriptor.jpg)
    - System Segment and Gate Descriptor
        - 由四位构成，最高位确定了是System Segment Descriptor还是Gate Descriptor
        - ![System Segment and Gate Descriptor](/image/x86_segmentation_and_paging/system_segment_and_gate_descriptor.jpg)
- Base Address位
    - 包含基址的信息。共32位，由三段构成。
    - [word 1]bit 16 : bit 31 + [word 2]bit 0 : bit 7 + [word 2]bit 24 : bit 31
- Segment Limit位
    - 指定段的大小。共20位，由两部分构成。
    - [word 1]bit 0 : bit 15 + [word 2]bit 16 : bit 19
    - 与G位一起决定段的大小限制
        - 如果G位未设置，那么段大小从1 byte到1 MByte，每次递增1byte
        - 如果G位已设置，那么段大小从4 KByte到4 GByte，每次递增4 KByte。
- P位
    - [word 2]bit 15
    - Present位。表明段是否在内存中。若P=1则表明在内存中，若P=0则表明不在内存中。
- DPL:指明了段的优先级，从0到4。
- D/B:
- L位：
- G位
    - 参见Segment Limit位
- AVL：为系统软件预留。


Segmentation过程：

- 逻辑地址分为两个部分，Segment Selector 和 Offset。
- 根据Selector中的TI位
    - 若TI=0
        - 从GDTR中取得GDT的入口地址。
        - 根据Segment Selector中高13位索引值得到所需要的段描述符（Discriptor）
        - 段描述符中包含了段的基址，Limit和其他一些属性。
        - 将段的基址与之前的Offset相加，得到32位的线性地址。
    - 若TI=1
        - 从GDTR中取得GDT的入口地址。
        - 根据LDTR中的Selector来确定LDT段的基址（即该LDT段在GDT中的起始位置）
        - 根据逻辑地址的Selector和LDT段的基址找到对应的段描述符。
        - 得到段描述符之后再根据段中包含的基址与之前的Offset相加，得到32位的线性地址。
    - ![Segmentation](/image/x86_segmentation_and_paging/segmentation.jpg )

###Paging:(Linear Address -> Physical Address)
简介

- 如果禁止Paging，那么线性地址为直接被转换为物理地址送往Memory。此时为Pure Segmentation的方案。
- 如果允许Paging
    - x86会有三种分页方式
        - 32bit-Paging
        - PAE-Paging
        - IA32e-Paging
    - 页的大小分为两种
        - 4-KByte Page
        - 4-MByte Page。

todo

### 参考文献
---
[1] Intel 64 and IA-32 Architectures Software Developer’s Manual. Volume 3A: System Programming Guide, Part 1. May, 2011.
[2] Andrew S.Tanenbaum. Modern Operating Systems (3rd Edition). 2009.
[3] Remzi H. Arpaci-Dusseau and Andrea C. Arpaci-Dusseau. Operating Systems: Three Easy Pieces. March, 2015.
