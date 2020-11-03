# lab1:OS启动、中断与设备管理

## 1 练习一：通过make生成执行文件的过程

### 1.1 操作系统镜像文件ucore.img是如何一步一步生成的？(Makefile中每一条相关命令和命令参数的含义及结果)

Makefile文件的基本描述规则为：
```
TARGET...：PREREQUISITES...
          COMMOND
          ...
          ...
```
> 其中target段描述规则的目标，一般为生成的最终文件名或者中间过程文件名，也可以是一个指令动作名；

> prerequistes：规则的依赖；一般是生成规则目标所需要的基本的文件名列表；

> commond：规则的命令行，实际执行的规则命令，即能够实际执行的shell命令和程序名， ***需要注意的是每一条命令占一行，且命令行必须以[Tap]字符开始***

#### Makefile 文件

##### 1-139 

设置环境变量，编译选项

##### 117

```
$(call add_files_cc,$(call listf_cc,$(LIBDIR)),libs,)
```

作用是把libs目录下的所有`.c（和.S）`文件编译产生`.o`文件放在`obj/libs`目录下


##### 136

```
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```


把`kern`目录下的所有`.c（和.S）`文件编译产生`.o`文件放在`obj/kern/**`目录下


##### 140-153

```
140 # create kernel target
141 kernel = $(call totarget,kernel)
142 
143 $(kernel): tools/kernel.ld
144 
145 $(kernel): $(KOBJS)
146         @echo + ld $@
147         $(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
148         @$(OBJDUMP) -S $@ > $(call asmfile,kernel)
149         @$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)
150 
151 $(call create_target,kernel)
152 
153 # -------------------------------------------------------------------
```

生成kernel：指定目录`bin/kernel`，链接脚本`kernel.ld`，把`kern`内的`.c`源代码编译为`.o`文件，然后链接所有`obj`文件得到可执行文件`kernel`

##### 153-170

```
153 # -------------------------------------------------------------------
154 
155 # create bootblock
156 bootfiles = $(call listf_cc,boot)
157 $(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))
158 
159 bootblock = $(call totarget,bootblock)
160 
161 $(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
162         @echo + ld $@
163         $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
164         @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
165         @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
166         @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
167 
168 $(call create_target,bootblock)
169 
170 # -------------------------------------------------------------------
```

生成`bootblock`：指定目录`bin/bootblock`，boot内的`.c/.S`文件编译生成`.o`文件，然后链接所有`.o`文件得到`bootblock.o`，反汇编得到`.asm`，生成`.out`文件，生成可执行文件`bootblock`

##### 170-176

```
170 # -------------------------------------------------------------------
171 
172 # create 'sign' tools
173 $(call add_files_host,tools/sign.c,sign,sign)
174 $(call create_target_host,sign,sign)
175 
176 # -------------------------------------------------------------------
```

生成sign工具：`tools/sign.c`，功能为将输入文件拷贝到输出，控制输出文件大小（将`bootloader`对齐到一个扇区的大小（512B））
##### 176-188
```
176 # -------------------------------------------------------------------
177 
178 # create ucore.img
179 UCOREIMG        := $(call totarget,ucore.img)
180 
181 $(UCOREIMG): $(kernel) $(bootblock)
182         $(V)dd if=/dev/zero of=$@ count=10000
183         $(V)dd if=$(bootblock) of=$@ conv=notrunc
184         $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc
185 
186 $(call create_target,ucore.img)
187 
188 #
```

生成`ucore.img`：生成一个10000字节的块，然后将`bootloader`和`kernel`拷贝过去


### 1.2 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

查看 `tools/sign.c` 文件

```
char buf[512];
if (size != 512) {
    fprintf(stderr, "write '%s' error, size is %d.\n", argv[2], size);
    return -1;
}
```

大小为512字节

```
buf[510] = 0x55;
buf[511] = 0xAA;
```

最后两个字节为0x55与0xAA

## 2 练习二：使用qemu执行并调试lab1中的软件

### 2.1 从CPU加电后执行的第一条指令开始,单步跟踪BIOS的执行


打开`tools/gdbinit`

```
file bin/kernel
target remote :1234
set architecture i8086 
break kern_init
continue
```

执行 `make debug`，`gdb` 调试 `x/i 0xffff0`

输出如下：

```
Breakpoint 1 at 0x100000: file kern/init/init.c, line 17.
(gdb) x/i 0xffff0
0xffff0:     ljmp   $0x3630,$0xf000e05b
```

`si`单步调试（stepi）

执行`qemu -d in_asm -D q.log`，将运行的汇编指令保存在q.log中

### 2.2 在初始化位置0x7c00设置实地址断点,测试断点正常

`b *0x7c00`设置断点

输出如下：

```
(gdb) b* 0x7c00
Breakpoint 2 at 0x7c00
```

`continue`继续

```
(gdb) continue
Continuing.
Breakpoint 2, 0x00007c00 in ?? ()
```

`x/10i $pc`显示下十条

```
=> 0x7c00:      cli    
0x7c01:      cld    
0x7c02:      xor    %eax,%eax
0x7c04:      mov    %eax,%ds
0x7c06:      mov    %eax,%es
0x7c08:      mov    %eax,%ss
0x7c0a:      in     $0x64,%al
```

### 2.3 从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较

```
	----------------
	IN: 
	0x00007c00:  cli    
	
	----------------
	IN: 
	0x00007c01:  cld    
	0x00007c02:  xor    %ax,%ax
	0x00007c04:  mov    %ax,%ds
	0x00007c06:  mov    %ax,%es
	0x00007c08:  mov    %ax,%ss
	
	----------------
	IN: 
	0x00007c0a:  in     $0x64,%al
	
	----------------
	IN: 
	0x00007c0c:  test   $0x2,%al
	0x00007c0e:  jne    0x7c0a
	
	----------------
	IN: 
	0x00007c10:  mov    $0xd1,%al
	0x00007c12:  out    %al,$0x64
	0x00007c14:  in     $0x64,%al
	0x00007c16:  test   $0x2,%al
	0x00007c18:  jne    0x7c14
	
	----------------
	IN: 
	0x00007c1a:  mov    $0xdf,%al
	0x00007c1c:  out    %al,$0x60
	0x00007c1e:  lgdtw  0x7c6c
	0x00007c23:  mov    %cr0,%eax
	0x00007c26:  or     $0x1,%eax
	0x00007c2a:  mov    %eax,%cr0
	
	----------------
	IN: 
	0x00007c2d:  ljmp   $0x8,$0x7c32
	
	----------------
	IN: 
	0x00007c32:  mov    $0x10,%ax
	0x00007c36:  mov    %eax,%ds
	
	----------------
	IN: 
	0x00007c38:  mov    %eax,%es
	
	----------------
	IN: 
	0x00007c3a:  mov    %eax,%fs
	0x00007c3c:  mov    %eax,%gs
	0x00007c3e:  mov    %eax,%ss
	
	----------------
	IN: 
	0x00007c40:  mov    $0x0,%ebp
	
	----------------
	IN: 
	0x00007c45:  mov    $0x7c00,%esp
	0x00007c4a:  call   0x7d0d
	
	----------------
	IN: 
	0x00007d0d:  push   %ebp
```

只是形式上符号变成实际的值，没有注释，其他大都一样

### 2.4 自己找一个bootloader或内核中的代码位置，设置断点并进行测试

在`0x7c12`设置断点

```
=> 0x7c12:      out    %al,$0x64
0x7c14:      in     $0x64,%al
0x7c16:      test   $0x2,%al
0x7c18:      jne    0x7c14
0x7c1a:      mov    $0xdf,%al
0x7c1c:      out    %al,$0x60
0x7c1e:      lgdtl  (%esi)
```

## 3 练习三：分析bootloader进入保护模式的过程

查看`boot/bootasm.S`

### 8-10

宏定义：内核代码段，内核数据段，保护模式使能标志

```
8 .set PROT_MODE_CSEG,        0x8                     # kernel code segment selector
9 .set PROT_MODE_DSEG,        0x10                    # kernel data segment selector
10 .set CR0_PE_ON,             0x1                     # protected mode enable flag
```

### 14-23

清理环境：关闭中断/清除方向标志/置0/数据段寄存器置0/附加段寄存器置0/堆栈段寄存器置0

```
14 start:
15 .code16                                             # Assemble for 16-bit mode
16     cli                                             # Disable interrupts
17     cld                                             # String operations increment
18 
19     # Set up the important data segment registers (DS, ES, SS).
20     xorw %ax, %ax                                   # Segment number zero
21     movw %ax, %ds                                   # -> Data Segment
22     movw %ax, %es                                   # -> Extra Segment
23     movw %ax, %ss                                   # -> Stack Segment
```

### 29-43

开启A20地址线：64端口中状态寄存器的值为0x2，`seta20.1`功能为等待64h端口空闲，键盘控制器空闲后发送写输出端口的指令，然后通过64h端口判断8042（键盘控制器`"8042" PS/2 Controller`）是否空闲，空闲则将0xdf写入60h端口，打开了A20，此时可访问的内存大小由1M变为4G

```
29 seta20.1:
30     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
31     testb $0x2, %al
32     jnz seta20.1
33 
34     movb $0xd1, %al                                 # 0xd1 -> port 0x64
35     outb %al, $0x64                                 # 0xd1 means: write data to 8042's P2 port
36 
37 seta20.2:
38     inb $0x64, %al                                  # Wait for not busy(8042 input buffer empty).
39     testb $0x2, %al
40     jnz seta20.2
41 
42     movb $0xdf, %al                                 # 0xdf -> port 0x60
43     outb %al, $0x60                                 # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

### 49，77-86

加载`GDT`表（相当于64位数组）：`GDT`表由三个全局描述符组成，空/代码段描述符/数据段描述符，大小0x17

```
49     lgdt gdtdesc
77 # Bootstrap GDT
78 .p2align 2                                          # force 4 byte alignment
79 gdt:
80     SEG_NULLASM                                     # null seg
81     SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)           # code seg for bootloader and kernel
82     SEG_ASM(STA_W, 0x0, 0xffffffff)                 # data seg for bootloader and kernel
83 
84 gdtdesc:
85     .word 0x17                                      # sizeof(gdt) - 1
86     .long gdt                                       # address gdt
```

### 50-52，56

进入保护模式：`CR0_PE_ON`恒为1，`cr0`置0实模式，`cr0`置1保护模式，跳转指令进入保护模式

```
50     movl %cr0, %eax
51     orl $CR0_PE_ON, %eax
52     movl %eax, %cr0
56     ljmp $PROT_MODE_CSEG, $protcseg
```

### 58-70

设置保护模式下的段寄存器并建立堆栈

```
58 .code32                                             # Assemble for 32-bit mode
59 protcseg:
60     # Set up the protected-mode data segment registers
61     movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
62     movw %ax, %ds                                   # -> DS: Data Segment
63     movw %ax, %es                                   # -> ES: Extra Segment
64     movw %ax, %fs                                   # -> FS
65     movw %ax, %gs                                   # -> GS
66     movw %ax, %ss                                   # -> SS: Stack Segment
67 
68     # Set up the stack pointer and call into C. The stack region is from 0--start(0x7c00)
69     movl $0x0, %ebp
70     movl $start, %esp
```

### 71

转到保护模式完成，进入boot主方法

```
71     call bootmain
```

## 4 练习四：分析bootloader加载ELF格式的OS的过程

### 4.1 bootloader如何读取硬盘扇区的？

#### 0号硬盘I/O端口

```
0x1F0：0号硬盘数据寄存器
0x1F1：错误寄存器
0x1F2：数据扇区计数
0x1F3：扇区数
0x1F4：柱面（低字节）
0x1F5：柱面（高字节）
0x1F6：驱动器/磁头寄存器
0x1F7：状态寄存器（读），命令寄存器（写）
```

#### 查看 boot/bootmain.c

##### 44-61

`readsect`函数：读取磁盘扇区，将`secno`对应的扇区拷贝到指针`dst`处；读取扇区数为1，`secno`的0-7，8-15，16-23，24-27表偏移量，28位0表示访问"Disk 0"，29-31位强制设为1；最后0x1F7输出0x20，把磁盘扇区数据读到指定`dst`位置

```
    // 将第secno扇区放到dst内存处
    static void
    readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();
	
	//0x1f2控制读写的扇区数，设置为1
    outb(0x1F2, 1);                         // count = 1
    //在LBA模式下 0x1f3 - 0x1f6为LBA参数
    //其中0x1f6只有前四位有效，第4位控制主盘or从盘
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    // 上面四条指令联合制定了扇区号
    // 在这4个字节线联合构成的32位参数中
    // 29-31位强制设为1
    // 28位(=0)表示访问"Disk 0"
    // 0-27位是28位的偏移量
    //给读取目标扇区的命令
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

	    insl(0x1F0, dst, SECTSIZE / 4);         // 读取到dst位置，
	                                            // 除以4因为这里以DW为单位
	}
```

##### 67-83

`readseg`函数：`secno`加1因为0扇区被引导占用，`elf`从1扇区开始

```
	static void
	readseg(uintptr_t va, uint32_t count, uint32_t offset) {
        //找到要读的“终点”
	    uintptr_t end_va = va + count;
	    // 找到真正的起点
	    va -= offset % SECTSIZE;
	    // 加1因为0扇区被引导占用
	    // ELF文件从1扇区开始
	    // 传进来的offset是相对于相对于elfhdr（文件头）的地址
	    uint32_t secno = (offset / SECTSIZE) + 1; 
        // 依次将对应扇区的内容读入至缓存。
	    for (; va < end_va; va += SECTSIZE, secno ++) {
	        readsect((void *)va, secno);
	    }
	}
```

### 4.2 bootloader是如何加载ELF格式的OS？

`bootloader`从磁盘读取8个扇区到内存中，检查是否为有效的`ELF`文件，根据 `header` 表将程序段读取到 `header` 指定的内存偏移地址中

#### elf格式的os

```
[~/.local/share/Trash/files/lab1/bin]
moocos-> file kernel
kernel: ELF 32-bit LSB  executable, Intel 80386, version 1 (SYSV), statically linked, not stripped
```

#### 查看 libs/elf.h

##### 9-25

结构体elfhdr（在磁盘中的存储结构）

```
struct elfhdr {
    uint32_t e_magic;     // must equal ELF_MAGIC
    uint8_t e_elf[12];
    uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    uint16_t e_machine;   // 3=x86, 4=68K, etc.
    uint32_t e_version;   // file version, always 1
    uint32_t e_entry;     // entry point if executable
    uint32_t e_phoff;     // file position of program header or 0
    uint32_t e_shoff;     // file position of section header or 0
    uint32_t e_flags;     // architecture-specific flags, usually 0
    uint16_t e_ehsize;    // size of this elf header
    uint16_t e_phentsize; // size of an entry in program header
    uint16_t e_phnum;     // number of entries in program header or 0
    uint16_t e_shentsize; // size of an entry in section header
    uint16_t e_shnum;     // number of entries in section header or 0
    uint16_t e_shstrndx;  // section number that contains section name strings
};
```

##### 28-37

在内存中的存储结构，`p_pa`为对应当前段的虚拟地址

```
struct proghdr {
    uint32_t p_type;   // loadable code or data, dynamic linking info,etc.
    uint32_t p_offset; // file offset of segment
    uint32_t p_va;     // virtual address to map segment
    uint32_t p_pa;     // physical address, not used
    uint32_t p_filesz; // size of segment in file
    uint32_t p_memsz;  // size of segment in memory (bigger if contains bss）
    uint32_t p_flags;  // read/write/execute bits
    uint32_t p_align;  // required alignment, invariably hardware page size
};
```

#### 查看 boot/bootmain.c

##### 85-115

`e_magic`判断是否合法；从磁盘中加载OS，`ph`表示`ELF`段表首地址，`eph`表示段表末地址；循环读取每个段到内存；`e_entry`内核入口地址

```
	void
	bootmain(void) {
	    // 首先读取ELF的头部
	    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);
	
	    // 通过储存在头部的幻数判断是否是合法的ELF文件
	    if (ELFHDR->e_magic != ELF_MAGIC) {
	        goto bad;
	    }
	
	    struct proghdr *ph, *eph;
	
	    // ELF头部有描述ELF文件应加载到内存什么位置的描述表，
	    // 先将描述表的头地址存在ph
	    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
	    eph = ph + ELFHDR->e_phnum;
	
	    // 按照描述表将ELF文件中数据载入内存
	    for (; ph < eph; ph ++) {
	        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
	    }
	    

	    // 根据ELF头部储存的入口信息entry，找到内核的入口
	    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
	
	bad:
	    outw(0x8A00, 0x8A00);
	    outw(0x8A00, 0x8E00);
	    while (1);
	}
```

### 5 练习五：实现函数调用堆栈跟踪函数

#### 函数栈

函数调用->函数参数依次入栈->返回地址入栈—>原来函数栈顶作为当前函数的栈底->函数运行完把压入栈的bp出栈
```
栈结构示意图：
+| 栈底方向|
|  ...            |
| ...             | 高位地址
| 参数3 |
| 参数2 |
| 参数1 |
| 返回地址 |
|上一层[ebp] | <-------- [ebp]
|局部变量          |   低位地址

```
#### kern/debug/kdebug.c

最后`eip`调用函数的返回地址，`ebp`获得上一个函数的栈针
```
void
print_stackframe(void) {
     /* LAB1 YOUR CODE : STEP 1 */
     /* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
      * (2) call read_eip() to get the value of eip. the type is (uint32_t);
      * (3) from 0 .. STACKFRAME_DEPTH
      *    (3.1) printf value of ebp, eip
      *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address (uint32_t)ebp +2 [0..4]
      *    (3.3) cprintf("\n");
      *    (3.4) call print_debuginfo(eip-1) to print the C calling function name and line number, etc.
      *    (3.5) popup a calling stackframe
      *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
      *                   the calling funciton's ebp = ss:[ebp]
      */
	uint32_t ebp=read_ebp(),eip=read_eip();
    // ebp即为基指针（该指针永远指向系统栈最上面一个栈帧的底部），通过它可以一层一层的找到栈中各函数的调用信息
	// eip是下一步代码执行的位置
	int i,j;
	for(i=0;ebp!=0&&i<STACKFRAME_DEPTH;i++){
        //按照16进制输出，补齐8位的宽度
		cprintf("ebp:0x%08x eip:0x%08x args:", ebp, eip);
        // 找到该函数的参数的起点位置
		uint32_t *args = (uint32_t *)ebp + 2;
		for (j = 0; j < 4; j ++) {
		    cprintf("0x%08x ", args[j]);
		}
		cprintf("\n");
        // eip总是存的下一执行语句的位置
		print_debuginfo(eip - 1);
		eip = ((uint32_t *)ebp)[1];
		ebp = ((uint32_t *)ebp)[0];
	}
}
```

#### 执行 make qemu

最后一行对应`bootmain`，进入保护模式之后就是`call bootmain`，是第一个使用堆栈的函数，从`0x7c00`开始，因为`call`指令压栈，所以`0x7c00-0x0008`（一个字节）得`0x7bf8`

```
ebp:0x00007b28   eip:0x00100ab3   args: 0x00010094 0x00010094 0x00007b58 0x00100096 
    kern/debug/kdebug.c:306: print_stackframe+25
-> ebp:0x00007b38   eip:0x00100dc8   args: 0x00000000 0x00000000 0x00000000 0x00007ba8 
    kern/debug/kmonitor.c:125: mon_backtrace+14
-> ebp:0x00007b58   eip:0x00100096   args: 0x00000000 0x00007b80 0xffff0000 0x00007b84 
    kern/init/init.c:48: grade_backtrace2+37
-> ebp:0x00007b78   eip:0x001000c4   args: 0x00000000 0xffff0000 0x00007ba4 0x00000029 
    kern/init/init.c:53: grade_backtrace1+42
-> ebp:0x00007b98   eip:0x001000e7   args: 0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+27
-> ebp:0x00007bb8   eip:0x00100111   args: 0x0010345c 0x00103440 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+38
-> ebp:0x00007be8   eip:0x00100055   args: 0x00000000 0x00000000 0x00000000 0x00007c4f 
    kern/init/init.c:28: kern_init+84
-> ebp:0x00007bf8   eip:0x00007d74   args: 0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d73 --

```

## 6 练习六：完善中断初始化和处理

### 中断分类

外部中断：由CPU外部设备引起的外部事件如I/O中断、时钟中断、控制台中断等是异步产生的（即产生的时刻不确定），与CPU的执行无关

内部中断：在CPU执行指令期间检测到不正常的或非法的条件(如除零错、地址访问越界)所引起的内部事件

系统调用：在程序中使用请求系统服务的系统调用而引发的事件

### 6.1 中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

根据文件功能描述，可以知道中断描述相关内容在mmu.h文件中定义，定义中断描述表的代码如下：
```
/* Gate descriptors for interrupts and traps */
struct gatedesc {
    unsigned gd_off_15_0 : 16;        // low 16 bits of offset in segment
    unsigned gd_ss : 16;            // segment selector
    unsigned gd_args : 5;            // # args, 0 for interrupt/trap gates
    unsigned gd_rsv1 : 3;            // reserved(should be zero I guess)
    unsigned gd_type : 4;            // type(STS_{TG,IG32,TG32})
    unsigned gd_s : 1;                // must be 0 (system)
    unsigned gd_dpl : 2;            // descriptor(meaning new) privilege level
    unsigned gd_p : 1;                // Present
    unsigned gd_off_31_16 : 16;        // high bits of offset in segment
};
```
代码中每一个标识符后面的数字表明该变量所占用的位数，一共有64位8个字节其中gd_off_15_0和gd_off_31_16，分别表示中断程序入口地址的低位和高位。其中根据注释信息可以知道0-1、6-7字节是offset，2-3字节是段选择子，通过段选择子得到段基址，加上offset即得到中断处理代码的入口.

### 6.2 编程完善kern/trap/trap.c中对中断向量表进行初始化的函数`idt_init`

#### 查看 kern/mm/mmu.h

`SETGATE`宏：4字节的中断描述表项，`off`是中断服务例程偏移量，`sel`是中断服务例程代码段选择子，`istrap`判断中断0系统调用1，`dpl`是访问权限

``` 
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```

#### kern/trap/trap.c

`__vectors[]`存放中断符，初始化`IDT`，权限内核级中断，使用`lidt`指令加载`IDT`

```
void
idt_init(void) {
     /* LAB1 YOUR CODE : STEP 2 */
     /* (1) Where are the entry addrs of each Interrupt Service Routine (ISR)?
      *     All ISR's entry addrs are stored in __vectors. where is uintptr_t __vectors[] ?
      *     __vectors[] is in kern/trap/vector.S which is produced by tools/vector.c
      *     (try "make" command in lab1, then you will find vector.S in kern/trap DIR)
      *     You can use  "extern uintptr_t __vectors[];" to define this extern variable which will be used later.
      * (2) Now you should setup the entries of ISR in Interrupt Description Table (IDT).
      *     Can you see idt[256] in this file? Yes, it's IDT! you can use SETGATE macro to setup each item of IDT
      * (3) After setup the contents of IDT, you will let CPU know where is the IDT by using 'lidt' instruction.
      *     You don't know the meaning of this instruction? just google it! and check the libs/x86.h to know more.
      *     Notice: the argument of lidt is idt_pd. try to find it!
      */
	extern uintptr_t __vectors[];
	int i;
	for(i=0;i<256;i++) {
		SETGATE(idt[i],0,GD_KTEXT,__vectors[i],DPL_KERNEL);
	}
	SETGATE(idt[T_SWITCH_TOK],0,GD_KTEXT,__vectors[T_SWITCH_TOK],DPL_USER);
    // 初始化ldt
	lidt(&idt_pd);
}
```

### 6.3 编程完善trap.c中的中断处理函数trap

kern/trap/trap.c

ticks加到TICK_NUM时打印

```
static void print_ticks() {
    cprintf("%d ticks\n",TICK_NUM);
#ifdef DEBUG_GRADE
    cprintf("End of Test.\n");
    panic("EOT: kernel seems ok.");
#endif
}
static void
trap_dispatch(struct trapframe *tf) {
    char c;
    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        /* LAB1 YOUR CODE : STEP 3 */
        /* handle the timer interrupt */
        /* (1) After a timer interrupt, you should record this event using a global variable (increase it), such as ticks in kern/driver/clock.c
         * (2) Every TICK_NUM cycle, you can print some info using a funciton, such as print_ticks().
         * (3) Too Simple? Yes, I think so!
         */
	ticks++;
        if(ticks == TICK_NUM) {
            print_ticks();
            ticks = 0;
        }
        break;
    case IRQ_OFFSET + IRQ_COM1:
        c = cons_getc();
        cprintf("serial [%03d] %c\n", c, c);
        break;
    case IRQ_OFFSET + IRQ_KBD:
        c = cons_getc();
        cprintf("kbd [%03d] %c\n", c, c);
        break;
    //LAB1 CHALLENGE 1 : YOUR CODE you should modify below codes.
    case T_SWITCH_TOU:
    case T_SWITCH_TOK:
        panic("T_SWITCH_** ??\n");
        break;
    case IRQ_OFFSET + IRQ_IDE1:
    case IRQ_OFFSET + IRQ_IDE2:
        /* do nothing */
        break;
    default:
        // in kernel, it must be a mistake
        if ((tf->tf_cs & 3) == 0) {
            print_trapframe(tf);
            panic("unexpected trap in kernel.\n");
        }
    }
}
```

执行 `make qemu`

输出如下：

```
++ setup timer interrupts
100 ticks
100 ticks
100 ticks
100 ticks
```
