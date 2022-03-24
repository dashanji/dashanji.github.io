---
title: Chapter-1 boot部分(引导扇区)详解
description: Chapter 1 of FreeFlyOS
toc: true
authors:
tags:
weight: 1
categories:
series:
date: '2022-03-24'
lastmod: '2022-03-24'
draft: false
---
## boot.ld

```
/* 
**   链接脚本
*/
OUTPUT_FORMAT(elf32-i386)
OUTPUT_ARCH(i386)
ENTRY(start)
/*
*   ld有多种方法设置进程入口地址, 按以下顺序: (编号越前, 优先级越高)
*           1, ld命令行的-e选项
*           2, 连接脚本的ENTRY(SYMBOL)命令
*           3, 如果定义了start 符号, 使用start符号值
*           4, 如果存在 .text section , 使用.text section的第一字节的位置值
*           5, 使用值0
*
*
*/

SECTIONS 
{
    /* 将定位器符号置为0x7c00 */
    . = 0x7C00;

    /*
    将所有(*符号代表任意输入文件)输入文件bootsector.S的.start section合并
     成一个.start section, 该section的地址由定位器符号的值
     指定, 即0x7c00.
     bootsector.o整体作为一个start节
    */
    .start : {
        *bootsector.o(.text)
    }

    /*
    将所有(*符号代表任意输入文件)输入文件的.text section合并
     成一个.text section, 该section的地址紧接.start节.
     bootmain.o中的text作为一个text节
    */
    .text : { *(.text) }

    /*
    将所有(*符号代表任意输入文件)输入文件的.data section合并
     成一个.data section, 该section的地址紧接.text节.
     bootmain.o中的data作为一个data节
    */
    .data : { *(.data .rodata) }
    
    /DISCARD/ : { *(.eh_*) }
}
```

## bootsector.S

```
/*
*  主要功能：关中断、内存探测、80x86 CPU从实模式变成保护模式
*           跳转到加载内核的32位代码段
*    注意：本文件不是MBR(512B)，而是和bootmain.c链接成MBR
*/
.text
.code16   #.code16表示16位代码段
.global start
/*
*将ds、es和ss段寄存器均设置成cs段寄存器的值，并将栈顶指针esp指向
*0x7c00，栈向低地址增长。这步操作其实也可省略，因为在16位代码段中
*还用不到其他段寄存器，在需要使用的时候再初始化不迟
*/
start:
movw %cs,%ax
movw %ax,%ds	# ->Data Segment
movw %ax,%es	# ->Extra Segment
movw %ax,%ss	# ->Stack Segment
movl $0x7C00,%esp

/*
*关中断，在后面我们在内存中会建立中断向量表，所以事先关好中断，
*防止在建表过程中来了中断，所以事先屏蔽，防止这种情况产生。
*/
cli


/* 内存探测，内存地址0x8000作为内存探测段数的存储地址，
   方便后面调用 */
movw $0,0x8000
movw $0x8004,%di
xor %ebx,%ebx
mm_probe:
movl $0xe820,%eax
movl $20,%ecx
movl $0x534D4150,%edx
int $0x15
#产生进位则跳转
jnc cont
jmp probe_end
cont:
incl 0x8000
addw $20,%di
cmpl $0,%ebx
jnz mm_probe
probe_end:


/*
*打开地址线A20。实际上若我们使用qemu跑这个程序时，A20默认打开，
*但为了兼容性，最好还是手动将A20地址线打开.读者可以试一试将打开
*A20代码删去后，在保护模式(32位代码段#)下用回滚机制测试时是否
*仍然显示字符
*
*8042(键盘控制器)端口的P21和A20相连，置1则打开
*0x64端口 读：位1=1 输入缓冲器满(0x60/0x64口有给8042的数据）
*0x64端口 写: 0xd1->写8042的端口P2，比如位2控制P21 
*当写时，0x60端口提供数据，比如0xdf，这里面把P2置1
*
*由于MacOS下编译器的版本原因，若加上下面代码会超出512B,故舍去
*/

/*waitforbuffernull1:

#先确定8042是不是为空,如果不为空，则一直等待
xorl %eax,%eax
inb $0x64,%al
testb $0x2,%al
jnz waitforbuffernull1

#8042中没有命令，则开始向0x64端口发出写P2端口的命令
movb $0xd1,%al
outb %al,$0x64
waitforbuffernull2:

#再确定8042是不是为空,如果不为空，则一直等待 
xorl %eax,%eax
inb $0x64,%al
testb $0x2,%al
jnz waitforbuffernull2

#向0x60端口发送数据，即把P2端口设置为0xdf
movb $0xdf,%al
outb %al,$0x60*/

/* 加载gdt表,即将内存中的gdt基址和表长写入GDTR寄存器 */
lgdt gdt_48

/* 打开保护模式，将cr0的位0置为1,一般而言BIOS中断
    只在实模式下进行调用 */
movl %cr0,%eax
orl $0x1,%eax
movl %eax,%cr0

/*
*进入到32位代码段。0x8代表段选择子(16位)——0000000000001000
*其中最后2为代表特权级.linux内核只使用了2个特权级(0和3)，00代表
*0特权级(内核级)，倒数第3位的代表是gdt(全局描述符表)还是
*idt(局部描述符表)，0代表全局描述符表，前13位代表gdt的项数(第1项)，
*属于代码段。所以0x8代表特权级为0(内核级)的全局代码段,promode代表
*偏移地址。
*/
ljmp $0x8,$promode

/* 保护模式下的32位代码段 */
promode:
.code32
movw $0x10,%ax
movw %ax,%ds	#->Data Segment
movw %ax,%es	#->Extra Segment
movw %ax,%ss	#->Stack Segment

movw $0x18,%ax
movw %ax,%gs

movl $0x0,%ebp
movl $start,%esp

/* 调用bootmain.c中的bootmain函数 */
call bootmain

/* 在内存中做一块GDT表 */
.align 2
gdt:
.word 0,0,0,0

.word 0xFFFF	#第1项CS,基地址为0，限长
.word 0x0000    
.word 0x9A00
.word 0x00CF

.word 0xFFFF	#第2项DS,基地址为0
.word 0x0000
.word 0x9200
.word 0x00CF

.word 0xFFFF	#第3项VGA,基地址位0xb8000
.word 0x8000
.word 0x920b
.word 0x0000 

/*
*将gdtr专用寄存器指向我们在内存中做的一块GDT表,GDTR寄存器格式:
*48位(高32位地址+低16位限长)，intel是小端方式
*/
gdt_48:
.word 0x1f    #gdt表限长 sizeof(gdt)-1 低地址，放在gdtr的低字节
.long gdt     #gdt表基址  高地址，放在gdtr的高字节



```

## bootmain.c

```
/*
*   主要功能：将elf格式的内核代码读入到指定内存区域中
*               本文件和bootsector.S组成MBR
*      注意： MBR文件末尾以0xaa55结束，需sign文件
*              格式化成合法的MBR
*/

#define SECTSIZE        512
#define ELFHDR          ((struct elfhdr *)0x10000)      // scratch space
#define ELF_MAGIC    0x464C457FU    //0x464C457FU 

/* elf文件头 */
struct elfhdr{
	unsigned int magic;      // must equal ELF_MAGIC
	unsigned char elf[12];
	unsigned short type;     // 1=relocatable, 2=executable, 3=shared object, 4=core image
	unsigned short machine;  // 3=x86, 4=68K, etc.
	unsigned int version;    // file version, always 1
	unsigned int entry;      // entry point if executable
	unsigned int phoff;      // file position of program header or 0
	unsigned int shoff;      // file position of section header or 0
	unsigned int flags;      // architecture-specific flags, usually 0
	unsigned short ehsize;   // size of this elf header
	unsigned short phentsize;// size of an entry in program header
	unsigned short phnum;    // number of entries in program header or 0
	unsigned short shentsize;// size of an entry in section header
	unsigned short shnum;    // number of entries in section header or 0
	unsigned short shstrndx; // section number that contains section name strings
};

/* 程序头 */
struct proghdr {
    unsigned int p_type;   // loadable code or data, dynamic linking info,etc.
    unsigned int p_offset; // file offset of segment
    unsigned int p_va;     // virtual address to map segment
    unsigned int p_pa;     // physical address, not used
    unsigned int p_filesz; // size of segment in file
    unsigned int p_memsz;  // size of segment in memory (bigger if contains bss）
    unsigned int p_flags;  // read/write/execute bits
    unsigned int p_align;  // required alignment, invariably hardware page size
};

/*
*   inb(port):从port端口中读取一个字节数据返回
*/
static inline unsigned char inb(unsigned short port) {
    unsigned char data;
    asm volatile ("inb %1, %0" : "=a" (data) : "d" (port));
    return data;
}

/*
*insl(port,addr,cnt):从port端口循环读cnt次双字到addr位置
*
*cld指令是使DF=0， 即si，di寄存器自动增加
*
*rep指令的目的是重复其上面的指令.ECX的值是重复的次数.repe和repne，
*前者是repeat equal，意思是相等的时候重复，后者是repeat not equal，
*不等的时候重复；每循环一次cx自动减一。
*
*insl 指令是从 DX 指定的 I/O 端口将双字输入 ES:(E)DI 指定的内存位置
*
*/
static inline void insl(unsigned int port, void *addr, int cnt) {
    asm volatile (
            "cld;"
            "repne; insl;"
            : "=D" (addr), "=c" (cnt)
            : "d" (port), "0" (addr), "1" (cnt)
            : "memory", "cc");
}

/*
*   outb(port,data):将一个字节数据data读入port端口中
*/
static inline void outb(unsigned short port, unsigned char data) {
    asm volatile ("outb %0, %1" :: "a" (data), "d" (port));
}

/*
*   waitdisk:等待硬盘准备好
*/
static inline void waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        ;
}

/*
*   readsect(dst,secno):读取扇区号secno所在的扇区进入dst地址中
*/
static inline void readsect(void *dst, unsigned int secno) {
    
    // 等待硬盘准备好
    waitdisk();

    outb(0x1F2, 1);                  // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);               // 命令0x20 - 读取扇区 

    // 等待硬盘准备好
    waitdisk();
    // 读取一个扇区
    insl(0x1F0, dst, SECTSIZE / 4);
}

/*
*   readseg(va,count,offset):读取内核基址偏移为offset处的count字节
*                               放入虚拟地址va中。
*/
static void readseg(unsigned int va, unsigned int count, unsigned int offset) {
    unsigned int end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    unsigned int secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}

/*
*   bootmain():读取第1号扇区中的内核的ELF头，获取程序段头信息
*                并把所有程序段读入内存的相应虚拟地址中
*               由于此时未开分页机制，虚拟地址=物理地址
*                最后进入ELF头的入口地址，即内核地址
*/
void bootmain(void) {
    // read the 1st page off disk
    readseg((unsigned int )ELFHDR, SECTSIZE * 2, 0);

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((unsigned int )ELFHDR + ELFHDR->phoff);
    eph = ph + ELFHDR->phnum;
    unsigned int mask;
    //由于内核放在16MB处，至少需要28位对齐（0xFFFFFFF）
    for (; ph < eph; ph ++) {
        //qemu特性决定
        if(ph->p_va > (unsigned int )0xC0000000){
            mask=0xFFFFFFF;
        }
        else{
            mask=0xFFFFFFFF;
        }
        readseg(ph->p_va & mask, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->entry & 0xFFFFFFFF))();

    while (1);
}


```

## sign.c

```
/*
*   主要功能：在bootsector.S和bootmain.c链接成的
*            bootblock.out文件末尾添加0xaa55结束符
*      注意： 合法的MBR文件末尾以0xaa55结束。
*             
*/
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <sys/stat.h>

/*
*  main(argc,argv)：在第一个参数argv[1]文件的末尾
*    添加0x55AA，然后写入第二个参数argv[2]文件中
*/
int main(int argc, char *argv[]) {
    struct stat st;

    if (argc != 3) {
        fprintf(stderr, "Usage: <input filename> <output filename>\n");
        return -1;
    }

    if (stat(argv[1], &st) != 0) {
        fprintf(stderr, "Error opening file '%s': %s\n", argv[1], strerror(errno));
        return -1;
    }
    printf("'%s' size: %lld bytes\n", argv[1], (long long)st.st_size);

    if (st.st_size > 510) {
        fprintf(stderr, "%lld >> 510!!\n", (long long)st.st_size);
        return -1;
    }

    char buf[512];
    memset(buf, 0, sizeof(buf));

    FILE *ifp = fopen(argv[1], "rb");
    int size = fread(buf, 1, st.st_size, ifp);
    if (size != st.st_size) {
        fprintf(stderr, "read '%s' error, size is %d.\n", argv[1], size);
        return -1;
    }
    fclose(ifp);

    buf[510] = 0x55;
    buf[511] = 0xAA;
    
    FILE *ofp = fopen(argv[2], "wb+");
    size = fwrite(buf, 1, 512, ofp);
    if (size != 512) {
        fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
        return -1;
    }
    fclose(ofp);
    printf("build 512 bytes boot sector: '%s' success!\n", argv[2]);
    return 0;
}


```



## CMakeLists.txt

```
#设置项目名
project (bootblock C ASM)

add_library(bootsector OBJECT bootsector.S)
add_library(bootmain OBJECT bootmain.c)

#链接
add_executable(${PROJECT_NAME}.o 
    $<TARGET_OBJECTS:bootsector>
    $<TARGET_OBJECTS:bootmain>
)

target_link_options(${PROJECT_NAME}.o PRIVATE -T ${FreeFlyOS_SOURCE_DIR}/boot/boot.ld)
target_link_options(${PROJECT_NAME}.o PRIVATE -Wl,-melf_i386)


add_custom_command(TARGET ${PROJECT_NAME}.o
    POST_BUILD
    COMMAND
        ${CMAKE_OBJCOPY} -S -O binary ${PROJECT_NAME}.o ${PROJECT_NAME}.out
    COMMAND    
        gcc ${FreeFlyOS_SOURCE_DIR}/boot/sign.c -o sign
    COMMAND    
        ./sign ${PROJECT_NAME}.out ${PROJECT_NAME}
)


```

1、编译bootsector.S以及bootmain.c文件生成bootsector.o和bootmain.o

2、将boosector.o和bootmain.o链接到0x7c00地址处生成bootblock.o

3、使用objcopy将bootblack.o复制生成可执行文件bootblock.out

4、使用格式化工具sign将bootblack.out格式化成MBR

5、之后为了给主分区表预留空间，该MBR从0x1BE-0x1FD的空间为主分区表信息，本项目的MBR如下图所示。


![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190118916.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)

## 功能说明

作为一个内核的bootloader，它的主要功能如下：

1、在16位代码段下，应先设置段寄存器（DS、ES、SS）的值（CS默认上电赋值）。

2、关中断，为之后建立中断向量表打好基础。

3、内存探测，通过BIOS提供的0x15中断获取可用内存数据，并存储在内存物理地址0x8000处。

4、打开地址A20，按理说这步应该是要做的，但由于本人环境编译器版本问题，加上这段代码会使MBR超过512B，故只能删掉这部分代码，经过测试，在qemu模拟器不影响后续代码的执行，以此来减少MBR的空间，并为主分区表预留空间。

5、加载gdt表，其中设置了CS和DS段基址为0，段限长为4GB，这样就能访问0-4GB的物理地址了，而实际分配给虚拟机2GB内存，为后面地址映射作准备。

6、接着就是打开保护模式了，把cr0的第0位置1即可，此时就无法调用BIOS中断了，所以如果需要用BIOS中断获取硬件信息，最好在保护模式前就写好。

7、使用ljmp长跳转到保护模式下的32位代码段，该段的属性已经在gdt表中写好，只需确定跳转代码的偏移地址即可。

8、上一步操作已经确定了CS段，而数据段并未确定，所以需要设置DS、ES、SS段的值，这里还设置了gs段的值，之前主要是为了能够用VGA输出字符，但后续直接写了VGA的驱动，所以这里的GS段设置和之前GDT中的VGA段均可删掉。

9、设置下栈地址，栈顶指针指向0x7c00，也就是bootloader开始的位置，但栈是向下增长的，故不会影响bootloader代码，在bootmain.c中会调用函数，故需要一个栈来存放参数和返回地址等信息。

10、将内核加载到内存中，一般内核文件都是elf文件格式，为了能够读取内核的elf头，我们将内核文件放在引导扇区MBR后面一个扇区，通过x86_64-elf-readelf -a build/kernel/kernel可以看到内核的elf文件布局，如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190152618.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190222781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


从这张图中我们能看到整个内核文件基本分为3个部分：init部分（.init.text/data）、user部分（.user.text/data/rodata/bss）、kernel部分(.text/rodata/data/bss)，各个部分的段地址由链接脚本决定，由于这个时候我们还没开启分页，虚拟地址=物理地址，而且实际物理内存只有2G，所以内核部分（即从C1000000开始）的代码，故在bootmain中需要进行一个判断，若虚拟地址大于2G，即内核代码，放在0x1000000处，当然，也可以设置qemu模拟器的内存大小，比如分配4G，但要注意的一点是，当分配给qemu模拟器4G内存中，可用只有3.5G，此后到4G之间的物理地址无法使用。

然后就是通过IO端口和硬盘进行交互，将各个部分的各个段读到对应的虚拟地址上（虚拟地址=物理地址），最后跳转到ELF文件的entry中，即0x100000（init部分的代码）。

12、boot下的链接脚本，如下图所示，0x7C00为约定好的系统引导地址，即BIOS执行完后，会自动执行0x7C00开始的bootloader。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190309211.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)

13、关于CMakeLists.txt的一些解释

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190338584.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2N5Mjk1OTU3NDEw,size_16,color_FFFFFF,t_70)


1⃣️project ( )------首先设置项目名称，一般这个可以随意设置，然后要注明使用了哪些语言，比如C语言，ASM汇编，若不指定则在编译时会自动看成C语言，.S文件就编译不了，需要在上层目录下的CMakeLists.txt下使能编译器。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190356603.png)


2⃣️add_library------将源文件编译成静态库文件，可以把它看成x86_64-elf-gcc(MAC下)/gcc(Ubuntu下) -c bootsector.c -o bootsector.o，当然这个编译选项在上一层也就是整个项目目录下的CMakeLists.txt确定的，如下图所示。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210103190413733.png)


具体说明下每个参数是啥意思吧：

-Os：主要对程序的尺寸进行优化，为了减少MBR的大小，可谓绞尽脑汁。

-nostdlib：不连接系统标准启动文件和标准哭文件，只把指定的文件传递给连接库，这样我们就能重写printf等，不会和标准C库重名了。

-fno-builtin：不使用C语言的内建函数，所以我们设置的函数名可以和内建函数同名。

-Wall：显示所有警告。

-ggdb：产生debug信息，用于gdb调试

-m32：生成32位机器的汇编代码

-gstabs：以stabs格式声称调试信息，但是不包括gdb调试信息。

-nostdinc：不包含C语言的标准库的头文件。

-fno-stack-protector：不使用栈保护检测。

3⃣️add_executable------将静态库文件链接成可执行文件，实际使用时为x86_64-elf-ld命令（MAC下）/ld命令（Ubuntu下）

4⃣️target_link_options-----设置链接选项，-T指定链接脚本，-Wl 传递参数 ，-melf_i386链接为32位程序

5⃣️add_custom_command-----在生成项目文件后，继续执行指定的命令，比如把.o文件转化为二进制文件，然后通过格式化文件sign将bootloader格式化为MBR。

最后，boot目录大概就总结这么多吧，有不懂的地方随时联系我，295957410@qq.com，项目源码地址为https://github.com/dashanji/FreeFlyOS。


