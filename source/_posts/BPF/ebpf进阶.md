---
title: ebpf进阶
date: 2023-05-25 15:14:25
categories: ebpf
tags:
- ebpf
- private
---



# ebpf 



## BSD Packet Filter（简称为 BPF）

为什么性能这么好呢？这主要得益于 BPF 的两大设计：

- 第一，内核态引入一个新的虚拟机，所有指令都在内核虚拟机中运行。
- 第二，用户态使用 BPF 字节码来定义过滤表达式，然后传递给内核，由内核虚拟机解释执行。

使得包过滤可以直接在内核中执行，避免了向用户态复制每个数据包，从而极大提升了包过滤的性能.



## eBPF 是怎么工作的?

​	eBPF 程序并不像常规的线程那样，启动后就一直运行在那里，它需要事件触发后才会执行。这些事件包括系统调用、内核跟踪点、内核函数和用户态函数的调用退出、网络事件，等等。借助于强大的内核态插桩（kprobe）和用户态插桩（uprobe），eBPF 程序几乎可以在内核和应用的任意位置进行插桩。



## Linux 内核是如何实现 eBPF 程序的安全和稳定的呢？

​	**确保安全和稳定一直都是 eBPF 的首要任务**，不安全的 eBPF 程序根本就不会提交到内核虚拟机中执行。

​	借助 [LLVM](https://llvm.org/) 把编写的 eBPF 程序转换为 BPF 字节码，然后再通过 bpf 系统调用提交给内核执行。内核在接受 BPF 字节码之前，会首先通过验证器对字节码进行校验，只有校验通过的 BPF 字节码才会提交到即时编译器执行。

[参考链接](https://www.brendangregg.com/ebpf.html)

[官方链接](https://ebpf.io/)

<img src=linux_ebpf_internals.png>

典型的验证过程：

- 只有特权进程才可以执行 bpf 系统调用；
- BPF 程序不能包含无限循环；
- BPF 程序不能导致内核崩溃；
- BPF 程序必须在有限时间内完成。

## 退出程序会发生什么
程序和map被自动卸载，这是因为内核使用引用计数来跟踪它们

## eBPF 限制有哪些？

最常见的 eBPF 限制：

- eBPF 程序必须被验证器校验通过后才能执行，且不能包含无法到达的指令；
- eBPF 程序不能随意调用内核函数，只能调用在 API 中定义的辅助函数；
- eBPF 程序栈空间最多只有 512 字节，想要更大的存储，就必须要借助映射存储；
- 在内核 5.2 之前，eBPF 字节码最多只支持 4096 条指令，而 5.2 内核把这个限制提高到了 100 万条；
- 由于内核的快速变化，在不同版本内核中运行时，需要访问内核数据结构的 eBPF 程序很可能需要调整源码，并重新编译。

> 虽然 Linux 内核很早就已经支持了 eBPF，但很多新特性都是在 4.x 版本中逐步增加的，具体可以看下[这个链接](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#main-features)。所以，想要稳定运行 eBPF 程序，**内核版本至少需要 4.9 或者更新**。而在开发和学习 eBPF 时，**为了体验最新的 eBPF 特性，我推荐使用更新的 5.x 内核**



## eBPF 在内核中的运行时主要由 5 个模块组成：

-   第一个模块是 **eBPF 辅助函数**。它提供了一系列用于 eBPF 程序与内核其他模块进行交互的函数。这些函数并不是任意一个 eBPF 程序都可以调用的，具体可用的函数集由 BPF 程序类型决定。
-   第二个模块是 **eBPF 验证器**。它用于确保 eBPF 程序的安全。验证器会将待执行的指令创建为一个有向无环图（DAG），确保程序中不包含不可达指令；接着再模拟指令的执行过程，确保不会执行无效指令。
-   第三个模块是由 **11 个 64 位寄存器、一个程序计数器和一个 512 字节的栈组成的存储模块**。这个模块用于控制 eBPF 程序的执行。其中，R0 寄存器用于存储函数调用和 eBPF 程序的返回值，这意味着函数调用最多只能有一个返回值；R1-R5 寄存器用于函数调用的参数，因此函数调用的参数最多不能超过 5 个；而 R10 则是一个只读寄存器，用于从栈中读取数据。
-   第四个模块是**即时编译器**(JIT)，它将 eBPF 字节码编译成本地机器指令，以便更高效地在内核中执行。
-   第五个模块是 **BPF 映射（map）**，它用于提供大块的存储。这些存储可被用户空间程序用来进行访问，进而控制 eBPF 程序的运行状态。

## eBPF程序从源代码到执行的过程中所经历的阶段

<img src=eBPF程序从源代码到执行的过程中所经历的阶段.png>

C(或Rust)源代码被编译成eBPF字节码，eBPF字节码通过JIT编译或解释成本机机器码指令

> 越来越多的eBPF程序也在用 Rust 编写，因为Rust编译器支持eBPF字节码作为目标。

## 探测方式
内核探测（kprobe）或用户探测（uprobe）



## 环境初始化：

1.   安装 vagrant [参考连接](https://www.vagrantup.com/downloads)

```bash
# Ubuntu/Debian
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install vagrant

# Centos/RHEL
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install vagrant

# mac
brew install vagrant
```

2.   创建环境

```bash
# 创建和启动Ubuntu 21.10虚拟机
vagrant init ubuntu/impish64
vagrant up

# 登录到虚拟机
vagrant ssh
```

3.   安装依赖

```bash
# For Ubuntu20.10+
sudo apt-get update -y
sudo apt-get install -y  make clang llvm build-essential libelf-dev libbpf-dev bpfcc-tools libbpfcc-dev golang-go linux-tools-$(uname -r) linux-headers-$(uname -r)

# For RHEL8.2+
sudo yum update -y
sudo yum install libbpf-devel make clang llvm elfutils-libelf-devel bpftool bcc-tools bcc-devel
```

4.   添加测试程序

4.1 程序解读：
```python
#!/usr/bin/python
from bcc import BPF

# ebpf 开发使用c
program = r"""
    int hello(void *ctx) {
        bpf_trace_printk("Hello World!");
return 0; }
"""

# bcc 会编译 c 代码，创建 bpf 对象
b = BPF(text=program)
# 获取要附加到内核函数的名称，因为不同系统对应的名称不同
# 每新建一个进程都会调用 execve ，然后触发 hello 
syscall = b.get_syscall_fnname("execve")
# 将 bpf程序加载到内核，并附加到一个事件上
b.attach_kprobe(event=syscall, fn_name="hello")
# 无限循环，直到停止程序
b.trace_print()
```

4.2 测试：
使用 C 开发一个 eBPF 程序

```bash
cat <<EOF | sudo tee hello.c 
int hello_world(void *ctx)
{
	// bpf_trace_printk() 是一个最常用的 BPF 辅助函数，它的作用是输出一段字符串
	// 它的输出并不是通常的标准输出（stdout），而是内核调试文件 /sys/kernel/debug/tracing/trace_pipe ，可以直接使用 cat 命令来查看这个文件的内容
    bpf_trace_printk("Hello, World!");
    return 0;
}
EOF
```

使用 Python 和 BCC 库开发一个用户态程序:

```bash
cat <<EOF | sudo tee hello.py
#!/usr/bin/env python3
# 1) 处导入了 BCC 库的 BPF 模块
from bcc import BPF

# 2) 调用 BPF() 加载第一步开发的 BPF 源代码
b = BPF(src_file="hello.c")
# 3) 将 BPF 程序挂载到内核探针（简称 kprobe），其中 do_sys_openat2() 是系统调用 openat() 在内核中的实现
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")

# 4) 读取内核调试文件 /sys/kernel/debug/tracing/trace_pipe 的内容，并打印到标准输出中
b.trace_print()
EOF
```

执行 eBPF 程序

```bash
chmod +x hello.py
sudo ./hello.py

# 输出日志
b' systemd-journal-350     [001] d...  1972.891622: bpf_trace_printk: Hello, World!'
```

>   在运行的时候，BCC 会调用 LLVM，把 BPF 源代码编译为字节码，再加载到内核中运行

输出的格式可由 `/sys/kernel/debug/tracing/trace_options` 来修改；

-   systemd-journal-350 表示进程的名字和 PID；
-   [001] 表示 CPU 编号；
-   d… 表示一系列的选项；
-   1972.891622 表示时间戳；
-   bpf_trace_printk 表示函数名；
-   最后的 “Hello, World!” 就是调用 `bpf_trace_printk()` 传入的字符串

5.   改进(利用通过 BPF 映射，同运行在内核中的 BPF 程序进行交互)

```bash
cat <<EOF | sudo tee trace-open.c
// 包含头文件
#include <uapi/linux/openat2.h>
#include <linux/sched.h>

// 定义数据结构
struct data_t {
	u32 pid;
	u64 ts;
	char comm[TASK_COMM_LEN];
	char fname[NAME_MAX];
};

// 定义性能事件映射
BPF_PERF_OUTPUT(events);

// 定义kprobe处理函数
// refer https://elixir.bootlin.com/linux/latest/source/fs/open.c#L1196 for the param definitions.
int hello_world(struct pt_regs *ctx, int dfd, const char __user * filename,
		struct open_how *how)
{
	struct data_t data = { };

    // 获取PID和时间
    // bpf_get_current_pid_tgid 用于获取进程的 TGID 和 PID。因为这儿定义的 data.pid 数据类型为 u32，所以高 32 位舍弃掉后就是进程的 PID
	data.pid = bpf_get_current_pid_tgid();
	// bpf_ktime_get_ns 用于获取系统自启动以来的时间，单位是纳秒
	data.ts = bpf_ktime_get_ns();
	
	// 获取进程名
	// bpf_get_current_comm 用于获取进程名，并把进程名复制到预定义的缓冲区中
	if (bpf_get_current_comm(&data.comm, sizeof(data.comm)) == 0) {
	    // bpf_probe_read 用于从指定指针处读取固定大小的数据，这里则用于读取进程打开的文件名
		bpf_probe_read(&data.fname, sizeof(data.fname),
			       (void *)filename);
	}

    // 提交性能事件
    // 调用 perf_submit() 把数据提交到刚才定义的 BPF 映射中
	events.perf_submit(ctx, &data, sizeof(data));
	return 0;
}
EOF
```

```bash
cat <<EOF | sudo tee trace-open.py
#!/usr/bin/env python3
# Tracing openat2() system call.
from bcc import BPF
from bcc.utils import printb


# 1) 加载 eBPF 程序并挂载到内核探针上
b = BPF(src_file="trace_open.c")
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")

# 2) 输出一行 Header 字符串表示数据的格式
print("%-18s %-16s %-6s %-16s" % ("TIME(s)", "COMM", "PID", "FILE"))

# 3) 定义一个数据处理的回调函数，打印进程的名字、PID 以及它调用 openat 时打开的文件
start = 0
def print_event(cpu, data, size):
    global start
    event = b["events"].event(data)
    if start == 0:
        start = event.ts
    time_s = (float(event.ts - start)) / 1000000000
    printb(b"%-18.9f %-16s %-6d %-16s" % (time_s, event.comm, event.pid, event.fname))


# 4) 定义了名为 “events” 的 Perf 事件映射，而后通过一个循环调用 perf_buffer_poll 读取映射的内容，并执行回调函数输出进程信息
b["events"].open_perf_buffer(print_event)
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
EOF
```

运行：

```bash
chmod +x trace-open.py
sudo ./trace-open.pys

TIME(s)            COMM             PID    FILE
0.000000000        b'irqbalance'    693    b'/proc/interrupts'
0.000101960        b'irqbalance'    693    b'/proc/stat'
0.000134411        b'irqbalance'    693    b'/proc/irq/20/smp_affinity'
0.000150126        b'irqbalance'    693    b'/proc/irq/11/smp_affinity'
0.000161972        b'irqbalance'    693    b'/proc/irq/0/smp_affinity'
0.000172789        b'irqbalance'    693    b'/proc/irq/1/smp_affinity'
0.000183295        b'irqbalance'    693    b'/proc/irq/4/smp_affinity'
```



## 查询系统中正在运行的 eBPF 程序：

```bash
# sudo bpftool prog list
89: kprobe  name hello_world  tag 38dd440716c4900f  gpl
      loaded_at 2021-11-27T13:20:45+0000  uid 0
      xlated 104B  jited 70B  memlock 4096B
      btf_id 131
      pids python3(152027)
```

-   89 是这个 eBPF 程序的编号
-   kprobe 是程序的类型
-   hello_world 是程序的名字

### 输出中显示关于程序的额外信息, 额外的统计数据以粗体显示
```bash
# sysctl -w kernel.bpf_stats_enabled=1
# sudo bpftool prog list
```

## 导出 eBPF 程序的指令:

```bash
# sudo bpftool prog dump xlated id 89
int hello_world(void * ctx):
; int hello_world(void *ctx)
   0: (b7) r1 = 33                  /* ! */
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
   1: (6b) *(u16 *)(r10 -4) = r1
   2: (b7) r1 = 1684828783          /* dlro */
   3: (63) *(u32 *)(r10 -8) = r1
   4: (18) r1 = 0x57202c6f6c6c6548  /* W ,olleH */
   6: (7b) *(u64 *)(r10 -16) = r1
   7: (bf) r1 = r10
;
   8: (07) r1 += -16
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
   9: (b7) r2 = 14
  10: (85) call bpf_trace_printk#-61616
; return 0;
  11: (b7) r0 = 0
  12: (95) exit
```

-   89 替换成查询到的编号
-   分号开头的部分，是写的 C 代码，而其他行则是具体的 BPF 指令



**具体每一行的 BPF 指令又分为三部分**：

-   第一部分，**冒号前面**的数字 0-12 ，代表 BPF **指令行数**；
-   第二部分，**括号中**的16进制数值，表示 BPF **指令码**。它的具体含义你可以参考 [IOVisor BPF 文档](https://github.com/iovisor/bpf-docs/blob/master/eBPF.md)，比如第 0 行的 0xb7 表示为 64 位寄存器赋值。
-   第三部分，**括号后面**的部分，就是 BPF 指令的**伪代码**。



**BPF 指令的含义：**

-   第0-8行，借助 R10 寄存器从栈中把字符串 “Hello, World!” 读出来，并放入 R1 寄存器中；
-   第9行，向 R2 寄存器写入字符串的长度 14（即代码注释里面的 `sizeof(_fmt)` ）；
-   第10行，调用 BPF 辅助函数 `bpf_trace_printk` 输出字符串；
-   第11行，向 R0 寄存器写入0，表示程序的返回值是0；
-   最后一行，程序执行成功退出。



具体指令的定义，请参考 [include/uapi/linux/bpf_common.h](https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf_common.h) 以及 [include/uapi/linux/bpf.h](https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf.h)），BPF 指令作为排查 eBPF 程序疑难杂症时的参考。



## 导出 eBPF 程序加载到内核的机器指令:

```bash
// 针对的是已经加载到内核内存中的程序（如果验证失败不会加载）
# bpftool prog dump jited id 89

int hello_world(void * ctx):
bpf_prog_38dd440716c4900f_hello_world:
; int hello_world(void *ctx)
   0:	nopl   0x0(%rax,%rax,1)
   5:	xchg   %ax,%ax
   7:	push   %rbp
   8:	mov    %rsp,%rbp
   b:	sub    $0x10,%rsp
  12:	mov    $0x21,%edi
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
  17:	mov    %di,-0x4(%rbp)
  1b:	mov    $0x646c726f,%edi
  20:	mov    %edi,-0x8(%rbp)
  23:	movabs $0x57202c6f6c6c6548,%rdi
  2d:	mov    %rdi,-0x10(%rbp)
  31:	mov    %rbp,%rdi
;
  34:	add    $0xfffffffffffffff0,%rdi
; ({ char _fmt[] = "Hello, World!"; bpf_trace_printk_(_fmt, sizeof(_fmt)); });
  38:	mov    $0xe,%esi
  3d:	call   0xffffffffd8c7e834
; return 0;
  42:	xor    %eax,%eax
  44:	leave
  45:	ret
```



## BPF 指令的加载和执行过程:

#### 1. BCC 的执行过程:

```bash
# -ebpf表示只跟踪bpf系统调用
sudo strace -v -f -ebpf ./hello.py

// 输出
bpf(BPF_PROG_LOAD,
    {
        prog_type=BPF_PROG_TYPE_KPROBE,
        insn_cnt=13,
        insns=[
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x21},
            {code=BPF_STX|BPF_H|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-4, imm=0},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x646c726f},
            {code=BPF_STX|BPF_W|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-8, imm=0},
            {code=BPF_LD|BPF_DW|BPF_IMM, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0x6c6c6548},
            {code=BPF_LD|BPF_W|BPF_IMM, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0x57202c6f},
            {code=BPF_STX|BPF_DW|BPF_MEM, dst_reg=BPF_REG_10, src_reg=BPF_REG_1, off=-16, imm=0},
            {code=BPF_ALU64|BPF_X|BPF_MOV, dst_reg=BPF_REG_1, src_reg=BPF_REG_10, off=0, imm=0},
            {code=BPF_ALU64|BPF_K|BPF_ADD, dst_reg=BPF_REG_1, src_reg=BPF_REG_0, off=0, imm=0xfffffff0},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_2, src_reg=BPF_REG_0, off=0, imm=0xe},
            {code=BPF_JMP|BPF_K|BPF_CALL, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0x6},
            {code=BPF_ALU64|BPF_K|BPF_MOV, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0},
            {code=BPF_JMP|BPF_K|BPF_EXIT, dst_reg=BPF_REG_0, src_reg=BPF_REG_0, off=0, imm=0}
        ],
        prog_name="hello_world",
        ...
    },
    128) = 4
```

>   BCC 负责了 eBPF 程序的编译和加载过程

查询 `bpf` 系统调用的格式（执行 `man bpf` 命令）

```c
int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

-   第一个参数是指定执行哪个命令。bpf()系统调用不只是做一件事——有许多不同的命令可用于操作eBPF程序和映射，`BPF_PROG_LOAD` 表示加载 BPF 程序。
-   第二个参数是 `bpf_attr` 类型的结构体，表示 BPF 程序的属性。bpf()系统调用的attr参数保存指定命令参数所需的任何数据，size表示attr中有多少字节的数据。其中，有几个需要留意的参数，比如：
    -   `prog_type` 表示 BPF 程序的类型，这儿是 `BPF_PROG_TYPE_KPROBE` ，跟Python 代码中的 `attach_kprobe` 一致；
    -   `insn_cnt` (instructions count) 表示指令条数；
    -   `insns` (instructions) 包含了具体的每一条指令，这儿的 13 条指令跟前面 `bpftool prog dump` 的结果是一致的（具体的指令格式可以参考内核中 [bpf_insn](https://elixir.bootlin.com/linux/v5.4/source/include/uapi/linux/bpf.h#L65) 的定义）；
    -   `prog_name` 则表示 BPF 程序的名字，即 `hello_world` 。
-   第三个参数 128 表示属性的大小。


<img src=用户空间程序与eBPF程序交互，并使用系统调用在内核中映射.png>


​	eBPF 程序并不像常规的线程那样，启动后就一直运行在那里，它需要事件触发后才会执行。所以，除了把 eBPF 程序加载到内核之外，还需要把加载后的程序跟具体的内核函数调用事件进行绑定。在 eBPF 的实现中，诸如内核跟踪（kprobe）、用户跟踪（uprobe）等的事件绑定，都是通过 `perf_event_open()` 来完成的。

对于 Hello World 来说，由于调用了 `attach_kprobe` 函数，很明显，这是一个内核跟踪事件：

```python
b.attach_kprobe(event="do_sys_openat2", fn_name="hello_world")
```



```bash
# sudo strace -v -f ./hello.py

// 输出
...
/* 1) 加载BPF程序 */
bpf(BPF_PROG_LOAD,...) = 4
...

/* 2）查询事件类型 */
openat(AT_FDCWD, "/sys/bus/event_source/devices/kprobe/type", O_RDONLY) = 5
read(5, "6\n", 4096)                    = 2
close(5)                                = 0
...

/* 3）创建性能监控事件 */
perf_event_open(
    {
        type=0x6 /* PERF_TYPE_??? */,
        size=PERF_ATTR_SIZE_VER7,
        ...
        wakeup_events=1,
        config1=0x7f275d195c50,
        ...
    },
    -1,
    0,
    -1,
    PERF_FLAG_FD_CLOEXEC) = 5

/* 4）绑定BPF到kprobe事件 */
ioctl(5, PERF_EVENT_IOC_SET_BPF, 4)     = 0
...
```

可以看出 BPF 与性能事件的绑定过程分为以下几步：

-   首先，借助 bpf 系统调用，加载 BPF 程序，并记住返回的文件描述符；
-   然后，查询 kprobe 类型的事件编号。BCC 实际上是通过 `/sys/bus/event_source/devices/kprobe/type` 来查询的；
-   接着，调用 `perf_event_open` 创建性能监控事件。比如，事件类型（type 是上一步查询到的 6）、事件的参数（ `config1 包含了内核函数 do_sys_openat2` ）等；
-   最后，再通过 `ioctl` 的 `PERF_EVENT_IOC_SET_BPF` 命令，将 BPF 程序绑定到性能监控事件



## 追踪 bpf 系统调用：

借助 BCC 宏定义 `TRACEPOINT_PROBE(category, event)` 比较方便，例如：

```bash
cat <<EOF | sudo tee test.c
TRACEPOINT_PROBE(syscalls, sys_enter_bpf)
{
    bpf_trace_printk("%d\\n", args->cmd);
    return 0;
}
EOF
```



```bash
cat <<EOF |sudo tee test.py
#!/usr/bin/env python3

from bcc import BPF

# load BPF program
b = BPF(src_file="test.c")
b.trace_print()
EOF
```



## 查询bpftool支持的特性:

```bash
# sudo bpftool version -p

如果出现下面的输出说明默认不支持libbfd，需要下载内核源码并安装binutils-dev之后重新编译bpftool：
{
  "features": {
    "libbfd": false
  }
}
```


## BPF 框架
可以使用bpftool从现有的ELF文件格式的eBPF对象自动生成这个框架代码
```bash
bpftool gen skeleton hello-buffer-config.bpf.o > hello-buffer-config.skel.h
```

查看这个框架头文件，会发现它包含 eBPF 程序和映射的结构定义，以及几个以名称（基于目标文件的名称）`hello_buffer_config_bpf__` 开头的函数。 这些函数管理 eBPF 程序和 map 的生命周期。 不必使用骨架代码，可以直接调用 libbpf, 但自动生成的代码通常会节省些输入

在生成的框架文件的末尾，会看到一个名为 `hello_buffer_config_bpf__elf_bytes` 的函数，它返回 ELF 对象文件 hello-buffer-config.bpf.o 的字节内容。 一旦生成了骨架，就不再需要那个目标文件了。 可以通过运行 make 生成 hello-buffer-config 可执行文件然后删除 `.o` 文件来测试它； 可执行文件中包含 eBPF 字节码。


## BPF 命令 

**不同版本的内核所支持的 BPF 命令是不同的**，具体支持的命令列表可以参考内核头文件 include/uapi/linux/bpf.h 中 `bpf_cmd` 的定义。比如，v5.13 内核已经支持 [36 个 BPF 命令](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L828)：

```c
enum bpf_cmd {
  BPF_MAP_CREATE,
  BPF_MAP_LOOKUP_ELEM,
  BPF_MAP_UPDATE_ELEM,
  BPF_MAP_DELETE_ELEM,
  BPF_MAP_GET_NEXT_KEY,
  BPF_PROG_LOAD,
  BPF_OBJ_PIN,
  BPF_OBJ_GET,
  BPF_PROG_ATTACH,
  BPF_PROG_DETACH,
  BPF_PROG_TEST_RUN,
  BPF_PROG_GET_NEXT_ID,
  BPF_MAP_GET_NEXT_ID,
  BPF_PROG_GET_FD_BY_ID,
  BPF_MAP_GET_FD_BY_ID,
  BPF_OBJ_GET_INFO_BY_FD,
  BPF_PROG_QUERY,
  BPF_RAW_TRACEPOINT_OPEN,
  BPF_BTF_LOAD,
  BPF_BTF_GET_FD_BY_ID,
  BPF_TASK_FD_QUERY,
  BPF_MAP_LOOKUP_AND_DELETE_ELEM,
  BPF_MAP_FREEZE,
  BPF_BTF_GET_NEXT_ID,
  BPF_MAP_LOOKUP_BATCH,
  BPF_MAP_LOOKUP_AND_DELETE_BATCH,
  BPF_MAP_UPDATE_BATCH,
  BPF_MAP_DELETE_BATCH,
  BPF_LINK_CREATE,
  BPF_LINK_UPDATE,
  BPF_LINK_GET_FD_BY_ID,
  BPF_LINK_GET_NEXT_ID,
  BPF_ENABLE_STATS,
  BPF_ITER_CREATE,
  BPF_LINK_DETACH,
  BPF_PROG_BIND_MAP,
};
```

**常用BPF命令：**

|                           BPF 命令                           |                    功能描述                     |
| :----------------------------------------------------------: | :---------------------------------------------: |
|                        BPF_MAP_CREATE                        |                 创建一个BPF映射                 |
| BPF_MAP_LOOKUP_ELEM <br />BPF_MAP_UPDATE_ELEM<br />BPF_MAP_DELETE_ELEM<br />BPF_MAP_GET_NEXT_KEY<br />BPF_MAP_LOOKUP_AND_DELETE_ELEM<br />BPF_MAP_GET_NEXT_KEY<br /> |              BPF映射相关，增删改查              |
|                        BPF_PROG_LOAD                         |                  验证并加载BPF                  |
|         BPF_PROG_ATTACH<br/>  BPF_PROG_DETACH<br />          |           把程序挂载、卸载到内核事件            |
|                         BPF_OBJ_PIN                          | 把BPF程序或映射挂载到sysfs中的/sys/fs/bpf目录中 |
|                         BPF_OBJ_GET                          |         从/sys/fs/bpf目录中查找BPF程序          |
|                         BPF_BTF_LOAD                         |                验证并加载BPF信息                |



## 辅助函数 Helper Functions
不许从eBPF程序直接调用任何内核函数(除非它已注册为kfunc)，但是eBPF提供了许多帮助函数，使程序能够从内核访问信息。不同的辅助函数对不同的BPF程序类型有效。例如，辅助函数`bpf_get_current_pid_tgid()`检索当前用户空间进程ID和线程ID，但是从XDP 程序调用它是没有意义的，因为它是由在网络接口接收数据包触发的，因为没有涉及到用户空间进程。所以，未知函数并不意味着函数是完全未知的，只是对于这个BPF程序类型来说它是未知的.

如果查看[kernel/bpf/helpers.c,2](https://elixir.bootlin.com/linux/latest/source/kernel/bpf/helpers.c)，会发现每个辅助函数都有一个`bpf_func_proto` 结构，类似于这个例子中的辅助函数`bpf_map_lookup_elem()`:
```c
const struct bpf_func_proto bpf_map_lookup_elem_proto = { 
    .func = bpf_map_lookup_elem,
    .gpl_only = false,
    .pkt_access = true,
    .ret_type = RET_PTR_TO_MAP_VALUE_OR_NULL,
    .arg1_type = ARG_CONST_MAP_PTR,
    .arg2_type = ARG_PTR_TO_MAP_KEY,
};
```
这个结构定义了辅助函数的参数和返回值的约束。因为验证器跟踪每个寄存器中保存的值的类型，所以它可以发现是否试图将错误类型的参数传递给辅助函数


​辅助函数的详细定义，可以在命令行中执行 `man bpf-helpers` ，或者参考内核头文件 [include/uapi/linux/bpf.h](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L1463) ，来查看它们的详细定义和使用说明.

举例：

```bash
$ bpftool feature probe
...
eBPF helpers supported for program type kprobe:
	- bpf_map_lookup_elem
	- bpf_map_update_elem
	- bpf_map_delete_elem
	- bpf_probe_read
	- bpf_ktime_get_ns
...
```

### 获取内核程序类型的辅助函数列表(`bpftool feature`)
可以使用 bpftool feature 命令获取您的内核版本中每种程序类型可用的辅助函数列表。 这显示了系统配置并列出了所有可用的程序类型和映射类型，甚至还列出了每种程序类型支持的所有辅助函数


### **常用辅助函数：**

| 辅助函数                                                     | 功能描述                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| bpf_trace_printk(fmt, fmt_size, ...)                         | 向调试文件系统写入调试信息                                   |
| bpf_map_lookup_elem(map, key)<br />bpf_map_update_elem(map, key, value, flags)<br />bpf_map_delete_elem(map, key) | BPF映射操作函数，对应查找、更新和删除元素                    |
| bpf_probe_read(dst, size, ptr)<br />bpf_probe_read_user(dst, size, ptr)<br />bpf_probe_read_kernel(dst, size, ptr) | 从内存指针中读取数据<br />从用户空间内存指针中读取数据<br />从内核空间内存指针中读取数据 |
| bpf_probe_read_str(dst, size, ptr)<br />bpf_probe_read_user_str(dst, size, ptr)<br />bpf_probe_read_kernel_str(dst, size, ptr) | 从内存指针中读取字符串<br />从用户空间内存指针中读取字符串<br />从内核空间内存指针中读取字符串 |
| bpf_ktime_get_ns()                                           | 获取系统启动以来的时长，单位纳秒                             |
| bpf_get_current_pid_tgid()                                   | 获取当前线程TGID(高32位)和PID(低32位)                        |
| bpf_get_current_comm(but, size)                              | 获取当前线程的任务名称                                       |
| bpf_get_current_task()                                       | 获取当前任务的task结构体                                     |
| bpf_perf_event_output(cox, map, flags, data, size)           | 向性能事件缓冲区中写入数据                                   |
| bpf_get_stackid(ctx, map, flags)                             | 获取内核态和用户态调用栈                                     |


## BPF内核函数(`kfuncs`)
尽管存在内核版本之间变化的风险，eBPF程序员仍然需要能够访问eBPF程序中的一些内部函数。这可以使用称为BPF内核函数(kfuncs)的机制来实现

Kfuncs允许向BPF子系统注册内部内核函数，以便验证器允许从eBPF程序调用它们。每个eBPF程序类型都有一个注册，允许调用给定的kfunc。
与辅助函数不同，kfuncs不提供兼容性保证，因此eBPF程序员必须考虑内核版本之间变化的可能性。

简而言之，**eBPF 程序的类型决定了它可以附加到哪些事件，而这些事件又定义了它接收的上下文信息的类型。 程序类型还定义了它可以调用的一组辅助函数和 kfunc**。

程序类型被广泛认为分为两类：
- 跟踪（或perf）程序类型
- 网络相关程序类型

### 跟踪程序类型：
附加到kprobes、tracepoints、raw tracepoints、fentry/fexit probes和perf事件的程序都被设计为为内核中的eBPF程序提供一种有效的方法，以便向用户空间报告有关事件的跟踪信息。这些与跟踪相关的类型并不会影响内核响应它们所附加的事件的方式

用不同的程序附加到与`execute()`相关的各种事件上为例：[代码参考](https://github.com/lizrice/learning-ebpf/blob/main/chapter7/hello.bpf.c)
```c
#include "vmlinux.h"
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_tracing.h>
#include <bpf/bpf_core_read.h>
#include "hello.h"

const char kprobe_sys_msg[16] = "sys_execve";
const char kprobe_msg[16] = "do_execve";
const char fentry_msg[16] = "fentry_execve";
const char tp_msg[16] = "tp_execve";
const char tp_btf_exec_msg[16] = "tp_btf_exec";
const char raw_tp_exec_msg[16] = "raw_tp_exec";
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} output SEC(".maps");

struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, u32);
    __type(value, struct msg_t);
} my_config SEC(".maps");

// 附加到execve()系统调用的kprobe，附加到系统调用的一个原因是，它们是稳定的接口，在内核版本之间不会改变(跟踪点也是如此）
SEC("ksyscall/execve")
int BPF_KPROBE_SYSCALL(kprobe_sys_execve, const char *pathname)
{
    struct data_t data = {}; 

    bpf_probe_read_kernel(&data.message, sizeof(data.message), kprobe_sys_msg); 
    bpf_printk("%s: pathname: %s", kprobe_sys_msg, pathname);

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;
    bpf_get_current_comm(&data.command, sizeof(data.command));
    bpf_probe_read_user(&data.path, sizeof(data.path), pathname);

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
}

// TODO!! Work on ARM
// 附加到内核中的任何非内联函数，SEC名称的格式与之前附加在syscall入口点上的版本相同，但没有必要定义特定平台的变体，因为do_execve()和大多数内核函数一样，是所有平台通用的。
#ifndef __TARGET_ARCH_arm64
/*
是如何知道为这个参数使用这种类型的？是因为内核中的 do_execve()函数有如下签名：
int do_execve(
    struct filename *filename, 
    const char __user *const __user *__argv, 
    const char __user *const __user *__envp
)

选择忽略 do_execve() 参数 __argv 和 __envp，只声明文件名参数，使用类型 struct filename * 来匹配内核函数的定义。 鉴于参数在内存中按顺序排列的方式，
忽略最后 n 个参数是可以的，但如果想使用后面的参数，就不能忽略列表中较早的参数
*/
SEC("kprobe/do_execve")
int BPF_KPROBE(kprobe_do_execve, struct filename *filename) {
    struct data_t data = {}; 

    // 综上所述:系统调用kprobe的上下文参数是一个结构，表示用户空间传递给系统调用的值。“常规”(非系统调用)kprobe的上下文参数是一个结构体，表示调用该函数的内核代码传递给被调用函数的参数，因此该结构体取决于函数定义
    bpf_probe_read_kernel(&data.message, sizeof(data.message), kprobe_msg); 

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_get_current_comm(&data.command, sizeof(data.command));
    const char *name = BPF_CORE_READ(filename, name);
    bpf_probe_read_kernel(&data.path, sizeof(data.path), name);

    bpf_printk("%s: filename->name: %s", kprobe_msg, name);

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;   
}
#endif

// This should really look at the kernel version, because fentry is supported on
// ARM from Linux 6.0 onwards
// 这个部分的名字告诉libbpf在do_ execve()内核函数的开始部分附加到fentry钩子。就像在kprobe的例子中一样，上下文参数反映了传递给内核函数的参数
#ifndef __TARGET_ARCH_arm64
SEC("fentry/do_execve")
int BPF_PROG(fentry_execve, struct filename *filename) {
    struct data_t data = {}; 

    bpf_probe_read_kernel(&data.message, sizeof(data.message), fentry_msg); 

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_get_current_comm(&data.command, sizeof(data.command));
    const char *name = BPF_CORE_READ(filename, name);
    bpf_probe_read_kernel(&data.path, sizeof(data.path), name);

    bpf_printk("%s: filename->name: %s", fentry_msg, name);

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;   
}
#endif

// name: sys_enter_execve
// ID: 622
// format:
//         field:unsigned short common_type;       offset:0;       size:2; signed:0;
//         field:unsigned char common_flags;       offset:2;       size:1; signed:0;
//         field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
//         field:int common_pid;   offset:4;       size:4; signed:1;

//         field:int __syscall_nr; offset:8;       size:4; signed:1;
//         field:const char * filename;    offset:16;      size:8; signed:0;
//         field:const char *const * argv; offset:24;      size:8; signed:0;
//         field:const char *const * envp; offset:32;      size:8; signed:0;
struct my_syscalls_enter_execve {
    unsigned short common_type;
    unsigned char common_flags;
    unsigned char common_preempt_count;
    int common_pid;

    long syscall_nr;
    void *filename_ptr;
    long argv_ptr;
    long envp_ptr;
};

SEC("tp/syscalls/sys_enter_execve")
int tp_sys_enter_execve(struct my_syscalls_enter_execve *ctx) {
    struct data_t data = {}; 

    bpf_probe_read_kernel(&data.message, sizeof(data.message), tp_msg);
    bpf_printk("%s: ctx->filename_ptr: %s", tp_msg, ctx->filename_ptr);

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_get_current_comm(&data.command, sizeof(data.command));
    bpf_probe_read_user(&data.path, sizeof(data.path), ctx->filename_ptr);  

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));   
    return 0;
}


// trace_event_raw_sched_process_exec is defined in vmlinux.h
SEC("tp_btf/sched_process_exec")
int tp_btf_exec(struct trace_event_raw_sched_process_exec *ctx)
{
    struct data_t data = {}; 
    // pid_t pid = ctx->pid; 

    bpf_probe_read_kernel(&data.message, sizeof(data.message), tp_btf_exec_msg);

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    // TODO!! Resolve issues accessing data that isn't aligned to an 8-byte boundary
    // bpf_printk("%s %d\n", tp_btf_exec_msg, pid);
    // bpf_probe_read_kernel_str(&data.command, sizeof(data.command), ctx->pid); 

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
}

SEC("raw_tp/sched_process_exec")
int raw_tp_exec(struct bpf_raw_tracepoint_args *ctx)
{
    struct data_t data = {}; 

    bpf_probe_read_kernel(&data.message, sizeof(data.message), raw_tp_exec_msg);

    data.pid = bpf_get_current_pid_tgid() >> 32;
    data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

    bpf_perf_event_output(ctx, &output, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
}

char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

bpftool perf 子命令允许查看附加到perf相关事件的程序: 
```bash
$ sudo bpftool perf show
    pid 232272  fd 16: prog_id 392  kprobe  func __x64_sys_execve  offset 0
    pid 232272  fd 17: prog_id 394  kprobe  func do_execve  offset 0
    pid 232272  fd 19: prog_id 396  tracepoint  sys_enter_execve
    pid 232272  fd 20: prog_id 397  raw_tracepoint  sched_process_exec
    pid 232272  fd 21: prog_id 398  raw_tracepoint  sched_process_exec
```
- 连接到 execve() 系统调用入口点的kprobe(entry point)
- 连接到内核函数 do_execve() 的kprobe(kernel function)
- 放在 execve() 系统调用入口处的追踪点(tracepoint)
- 在 execve() 处理过程中调用的两个版本的原始跟踪点(raw tracepoint)

#### kprobes and kretprobes
通常，kprobes附加到一个**函数的入口处**，用kretprobes附加到一个**函数的出口处**

#### fentry and fexit
在内核5.5版本中，引入了一种更有效的机制来追踪内核函数的进入和退出（在x86处理器上；BPF trampoline支持直到Linux 6.0才出现在ARM处理器上）。如果你使用的是一个足够新的内核，fentry/fexit 现在是追踪进入或退出内核函数的首选方法。你可以在kprobe或fentry类型的程序中编写同样的代码

Fentry 和 fexit 附着点被设计成**比 kprobe 更有效**,当想在一个函数的末尾生成一个事件时，还有一个优势：fexit 钩子可以访问函数的输入参数，而 kretprobe 则不能

#### tracepoints
追踪点是内核代码中的标记位置, 它们并不是eBPF独有的，长期以来一直被用于生成内核跟踪输出和SystemTap等工具。与使用kprobes附加到任意指令不同，**追踪点在不同的内核版本之间是稳定的**（尽管一个旧的内核可能没有被添加到一个新的内核中的全套追踪点）

通过 `/sys/kernel/tracing/available_events` 可以看到内核中可用的跟踪子系统; eBPF 程序的跟踪点 section 定义应匹配其中一项，以便 libbpf 可以自动将其附加到跟踪点。 定义的形式为 `SEC("tp/tracing subsystem/tracepoint name")`

```bash
[root@bash ~]# cat /sys/kernel/tracing/available_events|grep "sys_enter_execve"
syscalls:sys_enter_execveat
syscalls:sys_enter_execve
```

section 定义告诉libbpf这是一个跟踪点程序，以及它应该附加在哪里，像这样:
```c
SEC("tp/syscalls/sys_enter_execve")
```

跟踪点的上下文参数，BTF可以在这里帮助我们，但是首先让我们考虑一下当BTF不可用时需要什么。每个跟踪点都有一种格式来描述从中跟踪出来的字段。作为一个示例，下面是execve()系统调用入口处跟踪点的格式:
```bash
[root@bash ~]# cat /sys/kernel/tracing/events/syscalls/sys_enter_execve/format
name: sys_enter_execve
ID: 746
format:
	field:unsigned short common_type;	offset:0;	size:2;	signed:0;
	field:unsigned char common_flags;	offset:2;	size:1;	signed:0;
	field:unsigned char common_preempt_count;	offset:3;	size:1;	signed:0;
	field:int common_pid;	offset:4;	size:4;	signed:1;

	field:int __syscall_nr;	offset:8;	size:4;	signed:1;
	field:const char * filename;	offset:16;	size:8;	signed:0;
	field:const char *const * argv;	offset:24;	size:8;	signed:0;
	field:const char *const * envp;	offset:32;	size:8;	signed:0;

print fmt: "filename: 0x%08lx, argv: 0x%08lx, envp: 0x%08lx", ((unsigned long)(REC->filename)), ((unsigned long)(REC->argv)), ((unsigned long)(REC->envp))
```
使用这些信息定义了一个名为 my_syscalls_enter_ execve 的匹配结构
```c
struct my_syscalls_enter_execve { 
    unsigned short common_type; 
    unsigned char common_flags; 
    unsigned char common_preempt_count; 
    int common_pid;
    
    long syscall_nr; 
    long filename_ptr; 
    long argv_ptr; 
    long envp_ptr;
};
```
eBPF程序不允许访问前四个字段。如果尝试访问它们，程序将无法通过验证，并出现无效的 bpf_context 访问错误。
示例eBPF程序附加到这个跟踪点，可以使用指向该类型的指针作为其上下文参数，如下所示:
```c
SEC("tp/syscalls/sys_enter_execve")
int tp_sys_enter_execve(struct my_syscalls_enter_execve *ctx) {
```

### 启用 BTF 跟踪点
在前面的例子中，写了一个叫做 my_syscalls_enter_execve 的结构来定义自己的eBPF程序的上下文参数。但是当在自己的eBPF代码中定义一个结构或者解析原始参数时，自定义的代码有可能不符合它所运行的内核。有了BTF的支持，在 vmlinux.h 中会有一个结构被定义，它与传递给tracepoint eBPF程序的上下文结构相匹配。你的eBPF程序应该使用 `SEC("tp_btf/tracepoint name")` 部分定义，其中 tracepoint 名称是 `/sys/kernel/tracing/available_events` 中所列的可用事件之一, 看起来像这样
```c
SEC("tp_btf/sched_process_exec")
int handle_exec(struct trace_event_raw_sched_process_exec *ctx)
```
结构名称与跟踪点名称相匹配，前缀为 `trace_event_raw_`


### 用户空间附件
在用户空间代码中也有类似的附着点: 用于附加到用户空间函数的入口和出口的 uprobe 和 uretprobe，以及用于附加到应用程序代码或用户空间库中的指定跟踪点的用户静态定义跟踪点(usdt)。这些都使用 `BPF_PROG_TYPE_KPROBE` 程序类型


## 内联函数 `__always_inline`

一般 eBPF 程序调用的所有函数都以静态 `always_inline` 为前缀, 这指示编译器将函数的指令内联，这是因为在旧版本内核中，不允许 BPF 程序跳转到单独的函数。较新的内核和 LLVM 版本中已经可以支持非内联函数调用，但是设置 `always_inline` 可确保 BPF 验证器保能够更好工作。

想把重复的代码封装到一个函数中，避免重复代码过多，但是在早期ebpf程序不允许调用除辅助函数之外的其他函数，为了解决这个问题，出现了内联函数进行解决

```c
static __always_inline void my_function(void *ctx, int val)
```

一般代码中使用函数的地方会导致编译器发出一个跳转指令，使执行跳转到构成被调用函数的指令集，当该函数完成后再跳回（如下左半部分）。使用内联函数，没有跳转指令

<img src=内联和非内联指令的布局.png>

如果该函数被多个地方调用，将导致该函数的指令在编译后的可执行文件中出现多个副本，所以有时编译器可能为了优化而选择内联一个函数，这就是可能无法将 kprobe 附加到某些内核函数的原因之一。


## 尾调用 `bpf_tail_call()`

尾调用可以调用和执行另一个ebpf程序，并且**取代执行上下文**。尾调用完不会返回调用者那里。

尾调用动机是为了避免在一个函数被递归调用时一次又一次的向堆栈添加帧，最终会导致堆栈溢出错误，尾调用允许在不增加堆栈的情况下调用一系列的函数。（ebpf堆栈限制在512字节）

```c
long bpf_tail_call(void *ctx, struct bpf_map *prog_array_map, u32 index)
```

如果成功也不返回，失败也会继续执行。

index 代表该ebpf程序中哪个被调用。**如何提前设置index？**

```python
#!/usr/bin/python3  
from bcc import BPF
from time import sleep
import ctypes as ct

program = r"""
// 用于定义 BPF_MPAP_TYPE_PROG_ARRAY 类型的map，调用了map syscall 并允许 300 个条目
BPF_PROG_ARRAY(syscall, 300);
// 把这个eBPF程序附加到sys_enter原始跟踪点上，它在任何系统调用时都会被命中。传递给附加到原始跟踪点的eBPF程序的上下文采用bpf_raw_tracepoint_args结构的形式
int hello(struct bpf_raw_tracepoint_args *ctx) {
    // 在sys_enter的情况下，原始的tracepoint参数包括操作码，用于识别正在进行的系统调用
    int opcode = ctx->args[1];
    // 对程序数组中键与操作码匹配的条目进行尾部调用
    syscall.call(ctx, opcode);
    // 如果尾部调用成功，追踪操作码值的这一行将永远不会被命中
    bpf_trace_printk("Another syscall: %d", opcode);
    return 0;
}
// 将被加载到系统调用程序数组映射中
int hello_exec(void *ctx) {
    bpf_trace_printk("Executing a program");
    return 0;
}
// 另一个将被加载到syscall程序数组中的程序
int hello_timer(struct bpf_raw_tracepoint_args *ctx) {
    int opcode = ctx->args[1];
    switch (opcode) {
        case 222:
            bpf_trace_printk("Creating a timer");
            break;
        case 226:
            bpf_trace_printk("Deleting a timer");
            break;
        default:
            bpf_trace_printk("Some other timer operation");
            break;
    }
    return 0;
}
// 是一个不执行任何操作的尾部调用程序。在不希望生成任何跟踪的系统调用中使用这种方法。
int ignore_opcode(void *ctx) {
    return 0;
}
"""

b = BPF(text=program)
# 将主eBPF程序附加到sys_enter tracepoint（跟踪点）上
b.attach_raw_tracepoint(tp="sys_enter", fn_name="hello")

# 为每个尾部调用程序返回一个文件描述符。注意，尾部调用需要具有与它们的父程序相同的程序类型（当前是 BPF.RAW_TRACEPOINT）
ignore_fn = b.load_func("ignore_opcode", BPF.RAW_TRACEPOINT)
exec_fn = b.load_func("hello_exec", BPF.RAW_TRACEPOINT)
timer_fn = b.load_func("hello_timer", BPF.RAW_TRACEPOINT)

# 用户空间代码在系统调用 map 中创建数据条目。不需要将每个可能的操作码都填充 map; 如果没有特定操作码的条目，
# 则仅仅意味着不会执行尾调用。另外，有多个指向同一个eBPF程序的条目也是完全没问题的。
prog_array = b.get_table("syscall")
prog_array[ct.c_int(59)] = ct.c_int(exec_fn.fd)
prog_array[ct.c_int(222)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(223)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(224)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(225)] = ct.c_int(timer_fn.fd)
prog_array[ct.c_int(226)] = ct.c_int(timer_fn.fd)

# 有些系统调用被系统频繁地运行，以至于每一个系统调用都有一行跟踪记录，使跟踪输出变得杂乱无章，无法阅读
# Ignore some syscalls that come up a lot
prog_array[ct.c_int(21)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(22)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(25)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(29)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(56)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(57)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(63)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(64)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(66)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(72)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(73)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(79)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(98)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(101)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(115)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(131)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(134)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(135)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(139)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(172)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(233)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(280)] = ct.c_int(ignore_fn.fd)
prog_array[ct.c_int(291)] = ct.c_int(ignore_fn.fd)

# 将跟踪输出打印到屏幕上，直到用户终止该程序
b.trace_print()
```

## ebpf虚拟机
eBPF虚拟机与任何虚拟机一样，是计算机的软件实现。它接受eBPF字节码指令形式的程序，这些指令必须转换为在CPU上运行的本机指令。

在eBPF的**早期**实现中，字节码指令是在内核中**解释**的——也就是说，每当eBPF程序运行时，内核都会检查指令并将其转换为机器代码，然后执行。由于性能原因和避免eBPF解释器中幽灵相关漏洞的可能性，解释器在很大程度上已经被JIT(即时)编译所取代。编译意味着到本机机器指令的转换只发生一次，即当程序加载到内核中时。


eBPF字节码由一组指令组成，这些指令作用于(虚拟的)eBPF寄存器。eBPF指令集和寄存器模型被设计成整齐地映射到通用的CPU架构，这样从字节码编译或解释到机器代码的步骤就相当简单了

## 寄存器

eBPF虚拟机使用10个通用寄存器，编号为0到9。另外，寄存器10被用作堆栈帧指针(只能读，不能写)。在执行BPF程序时，值存储在这些寄存器中以跟踪状态。

可以在linux内核源代码的 `include/uapi/linux/bpf.h` 头文件中看到从`BPF_REG_0` 到 `BPF_REG_10` 的枚举。

eBPF程序的 context 参数在它开始执行之前被加载到**寄存器1**中。函数的返回值存储在**寄存器0**中。
在从eBPF代码调用函数之前，该函数的参数被放置在寄存器1到寄存器5中(如果参数少于5个, 不是所有寄存器都被使用)。

-   `R0` 寄存器: 用于存储函数调用和 eBPF 程序的返回值，这意味着函数调用最多只能有一个返回值；
-   `R1-R5` 寄存器: 用于函数调用的参数，因此函数调用的参数最多不能超过 5 个；
-   `R10` 则是一个只读寄存器，用于从栈中读取数据。

>   参数超过5个的解决办法：[待补充]()

**返回值检测**：
eBPF程序的返回码存储在寄存器0 (R0)中。如果程序**没有初始化**`R0`，验证器将失败，如下代码将返回错误：`R0 !read_ok`
```c
SEC("xdp")
int xdp_hello(struct xdp_md *ctx) {
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end;
     // bpf_printk("%x", data_end);
     // return XDP_PASS;
}
```
这将使验证器失败。但是，如果将包含辅助函数`bpf_printf()`的行放回去，即使源代码没有设置显式返回值，验证器也不会报错!
这是因为寄存器0也被用来保存辅助函数的返回代码。在eBPF程序中从一个辅助函数返回后，寄存器0不再是未初始化的。

## BPF 映射

​		BPF 映射用于提供大块的键值存储，这些存储可被用户空间程序访问，进而获取 eBPF 程序的运行状态。eBPF 程序最多可以访问 64 个不同的 BPF 映射，并且不同的 eBPF 程序也可以通过相同的 BPF 映射来共享它们的状态。

​		BPF 辅助函数中并没有 BPF 映射的创建函数，BPF 映射只能通过用户态程序的系统调用来创建。

​		设置映射的类型。内核头文件[ include/uapi/linux/bpf.h ](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L867)中的 `bpf_map_type` 定义了所有支持的映射类型，可以使用如下的 bpftool 命令，来查询当前系统支持哪些映射类型：

```bash
$ bpftool feature probe | grep map_type
eBPF map_type hash is available
eBPF map_type array is available
eBPF map_type prog_array is available
eBPF map_type perf_event_array is available
eBPF map_type percpu_hash is available
eBPF map_type percpu_array is available
eBPF map_type stack_trace is available
...
```

​	创建一个 BPF 映射，并返回映射的文件描述符示例：

```c
int bpf_create_map(enum bpf_map_type map_type,
		   unsigned int key_size,
		   unsigned int value_size, unsigned int max_entries)
{
  union bpf_attr attr = {
    .map_type = map_type,
    .key_size = key_size,
    .value_size = value_size,
    .max_entries = max_entries
  };
  return bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
}
```

**常用映射类型：**

| 映射类型                                                  | 功能描述                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| BPF_MAP_TYPE_HASH                                         | 哈希表映射，用于保存 key/value 对                            |
| BPF_MAP_TYPE_LRU_HASH                                     | 类似于哈希表映射，但在表满的时候自动按LRU算法删除最久未被使用的元素 |
| BPF_MAP_TYPE_ARRAY                                        | 数组映射，用于保存固定大小的数组（注意数组元素无法删除）     |
| BPF_MAP_TYPE_PROG_HASH                                    | 程序数组映射，用于保存BPF程序的引用，特别适合于尾调用（即调用其他eBPF程序） |
| BPF_MAP_TYPE_PERF_EVENT_ARRAY                             | 性能事件数组映射，用于保存性能事件跟踪记录                   |
| BPF_MAP_TYPE_PERCPU_HASH<br />BPF_MAP_TYPE_PERCPU_ARRAY   | 每个CPU单独维护的哈希表和数组映射                            |
| BPF_MAP_TYPE_STACK_TRACE                                  | 调用栈跟踪映射，用于存储调用栈信息                           |
| BPF_MAP_TYPE_ARRAY_OF_MAPS<br />BPF_MAP_TYPE_HASH_OF_MAPS | 映射数组和映射哈希，用于保存其他映射的引用                   |
| BPF_MAP_TYPE_CGROUP_ARRAY                                 | CGROUP 数组映射，用于存储cgroups 引用                        |
| BPF_MAP_TYPE_SOCKMAP                                      | 套接字映射，用于存储套接字引用，特别适用于套接字重定向       |

​	BPF 系统调用中并没有删除映射的命令，因为 **BPF 映射会在用户态程序关闭文件描述符的时候自动删除**（即`close(fd)` ）。 如果想在程序退出后还保留映射，就需要调用 `BPF_OBJ_PIN` 命令，将映射挂载到 /sys/fs/bpf 中。

​	通过 bpftool 来查看或操作映射：

```bash
//创建一个哈希表映射，并挂载到/sys/fs/bpf/stats_map(Key和Value的大小都是2字节)
# bpftool map create /sys/fs/bpf/stats_map type hash key 2 value 2 entries 8 name stats_map

//查询系统中的所有映射
# bpftool map
//示例输出
//340: hash  name stats_map  flags 0x0
//        key 2B  value 2B  max_entries 8  memlock 4096B

//向哈希表映射中插入数据
# bpftool map update name stats_map key 0xc1 0xc2 value 0xa1 0xa2

//查询哈希表映射中的所有数据
 
# bpftool map dump name stats_map
//示例输出
//key: c1 c2  value: a1 a2
//Found 1 element

//删除哈希表映射
# rm /sys/fs/bpf/stats_map
```



## **BPF 类型格式 (BTF)**

​	升级不同版本内核会对bpf程序依赖产生影响，为了**解决内核数据结构的定义问题**，从内核 5.2 开始，只要开启了 `CONFIG_DEBUG_INFO_BTF`，在编译内核时，内核数据结构的定义就会自动内嵌在内核二进制文件 vmlinux 中。并且，还可以借助下面的命令，把这些数据结构的定义导出到一个头文件中（通常命名为 `vmlinux.h`），有了内核数据结构的定义，在开发 eBPF 程序时只需要引入一个 `vmlinux.h` 即可，不用再引入一大堆的内核头文件了。

```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

<img src="https://static001.geekbang.org/resource/image/45/20/45bbf696e8620d322d857ceab3871720.jpg?wh=1920x1204" alt="img" style="zoom: 40%;" />

​	**如何让 eBPF 程序在内核升级之后，不需要重新编译就可以直接运行**，一次编译到处执行（Compile Once Run Everywhere，简称 CO-RE）



一个完整的 eBPF 程序通常包含**用户态**和**内核态**两部分：

-   用户态程序通过 BPF 系统调用，完成 eBPF 程序的加载、事件挂载以及映射创建和更新
-   内核态中的 eBPF 程序则需要通过 BPF 辅助函数完成所需的任务

### 对象文件中的 BTF 信息
BTF 的内核文档分两部分描述了 BTF 数据如何在 ELF 目标文件中编码：
- `.BTF`，其中包含数据和字符串信息
- `.BTF.ext`，其中包含函数和行信息 

可以使用 `readelf` 查看这些部分已添加到目标文件中，如下所示：
```bash
$ readelf -S hello-buffer-config.bpf.o | grep BTF
[10] .BTF              PROGBITS     0000000000000000  000002c0
[11] .rel.BTF          readelf      0000000000000000  00000e50
[12] .BTF.ext          PROGBITS     0000000000000000  00000b18
[13] .rel.BTF.ext      readelf      0000000000000000  00000ea0
```

bpftool实用程序允许我们检查对象文件中的 BTF 数据，如下所示:
```bash
bpftool btf dump file hello-buffer-config.bpf.o
```

输出看起来就像从加载的程序和映射中转储BTF信息得到的输出

## eBPF 程序类型:

​	程序类型决定了一个 eBPF 程序可以挂载的事件类型和事件参数; 根据内核头文件 [include/uapi/linux/bpf.h](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L908) 中 `bpf_prog_type` 的定义，Linux 内核 v5.13 已经支持 30 种不同类型的 eBPF 程序（注意， `BPF_PROG_TYPE_UNSPEC`表示未定义）：

```c
enum bpf_prog_type {
	BPF_PROG_TYPE_UNSPEC, /* Reserve 0 as invalid program type */
	BPF_PROG_TYPE_SOCKET_FILTER,
	BPF_PROG_TYPE_KPROBE,
	BPF_PROG_TYPE_SCHED_CLS,
	BPF_PROG_TYPE_SCHED_ACT,
	BPF_PROG_TYPE_TRACEPOINT,
	BPF_PROG_TYPE_XDP,
	BPF_PROG_TYPE_PERF_EVENT,
	BPF_PROG_TYPE_CGROUP_SKB,
	BPF_PROG_TYPE_CGROUP_SOCK,
	BPF_PROG_TYPE_LWT_IN,
	BPF_PROG_TYPE_LWT_OUT,
	BPF_PROG_TYPE_LWT_XMIT,
	BPF_PROG_TYPE_SOCK_OPS,
	BPF_PROG_TYPE_SK_SKB,
	BPF_PROG_TYPE_CGROUP_DEVICE,
	BPF_PROG_TYPE_SK_MSG,
	BPF_PROG_TYPE_RAW_TRACEPOINT,
	BPF_PROG_TYPE_CGROUP_SOCK_ADDR,
	BPF_PROG_TYPE_LWT_SEG6LOCAL,
	BPF_PROG_TYPE_LIRC_MODE2,
	BPF_PROG_TYPE_SK_REUSEPORT,
	BPF_PROG_TYPE_FLOW_DISSECTOR,
	BPF_PROG_TYPE_CGROUP_SYSCTL,
	BPF_PROG_TYPE_RAW_TRACEPOINT_WRITABLE,
	BPF_PROG_TYPE_CGROUP_SOCKOPT,
	BPF_PROG_TYPE_TRACING,
	BPF_PROG_TYPE_STRUCT_OPS,
	BPF_PROG_TYPE_EXT,
	BPF_PROG_TYPE_LSM,
	BPF_PROG_TYPE_SK_LOOKUP,
};
```

​	因为不同内核的版本和编译配置选项不同，一个内核并不会支持所有的程序类型, 可以在命令行中执行下面的命令，来查询当前系统支持的程序类型：

```bash
bpftool feature probe | grep program_type
```

程序类型大致可以划分为三类：

-   第一类是**跟踪**，即从内核和程序的运行状态中提取跟踪信息，来了解当前系统正在发生什么。
-   第二类是**网络**，即对网络数据包进行过滤和处理，以便了解和控制网络数据包的收发过程。
-   第三类是除跟踪和网络之外的**其他**类型，包括安全控制、BPF 扩展等等。



### 跟踪类 eBPF 程序

**主要用于从系统中提取跟踪信息，进而为监控、排错、性能优化等提供数据支撑**，常见的跟踪类 BPF 程序的主要功能以及使用限制：

<img src="image-20230103230938281.png" alt="image-20230103230938281" style="zoom:50%;" />



### 网络类 eBPF 程序

**主要用于对网络数据包进行过滤和处理，进而实现网络的观测、过滤、流量控制以及性能优化等各种丰富的功能**。根据事件触发位置的不同，网络类 eBPF 程序又可以分为 XDP（eXpress Data Path，高速数据路径）程序、TC（Traffic Control，流量控制）程序、套接字程序以及 cgroup 程序



<img src=BPF程序类型与网络堆栈中的不同点挂钩.png>

上面显示了一些常用的程序类型的连接位置。这些程序类型都需要 `CAP_NET_ADMIN` 和`CAP_BPF` 或 `CAP_SYS_ADMIN` 的能力来允许。
传递给这些类型的程序的上下文是有关的网络信息，尽管结构的类型取决于内核在网络堆栈的相关位置的数据。**在堆栈的底部，数据是以第二层网络数据包的形式存在的，它基本上是一系列已经或准备在 "线上 "传输的字节。在堆栈的顶部，应用程序使用套接字，内核创建套接字缓冲区来处理从这些套接字发送和接收的数据**


接收到的网络报文由一串字节组成，如图所示：
<img src=IP网络数据包的布局.png>

```c
unsigned char lookup_protocol(struct xdp_md *ctx) {
    unsigned char protocol = 0;
    void *data = (void *)(long)ctx->data;
    void *data_end = (void *)(long)ctx->data_end; struct ethhdr *eth = data;

    if (data + sizeof(struct ethhdr) > data_end)
        return 0;
    
    // Check that it's an IP packet
    if (bpf_ntohs(eth->h_proto) == ETH_P_IP) {
        // Return the protocol of this packet
        // 1 = ICMP
        // 6 = TCP
        // 17 = UDP
        struct iphdr *iph = data + sizeof(struct ethhdr);
        if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) <= data_end)
            protocol = iph->protocol;
    }
    return protocol; 
}
```
bpf_ntohs() 函数确保这两个字节在这个主机上是按照预期的顺序排列。网络协议是big-endian的，但大多数处理器是little-endian的，这意味着它们以不同的顺序保存多字节值。这个函数将网络顺序转换为主机顺序


#### **XDP 程序**
只要入站网络数据包到达它所连接的接口，就会触发 XDP 程序。 ctx 参数是指向 `xdp_md` 结构的指针，该结构保存有关传入数据包的元数据。
```c
struct xdp_md { __u32 data;
    __u32 data_end;
    __u32 data_meta;
    
    /* Below access go through struct xdp_rxq_info */ 
    __u32 ingress_ifindex; /* rxq->dev->ifindex */ 
    __u32 rx_queue_index; /* rxq->queue_index */
    __u32 egress_ifindex; /* txq->dev->ifindex */ 
};
```
**前三个字段的__u32类型它们实际上是指针**。`data`字段表示数据包在内存中的起始位置，`data_end`显示数据包结束的位置。要通过eBPF验证器，必须显式地检查对数据包内容的任何读取或写入是否在`data` 到 `data_end`范围内；在数据包之前的内存中还有一个区域，在 `data_meta` 和 `data` 之间，用于存储关于这个数据包的元数据。 这可以用于多个eBPF程序之间的协调，这些程序可能在堆栈中的不同地方处理同一个数据包。


类型定义为 `BPF_PROG_TYPE_XDP`，它在**网络驱动程序刚刚收到数据包时**触发执行。由于无需通过繁杂的内核网络协议栈，XDP 程序可用来实现高性能的网络处理方案，常用于 DDoS 防御、防火墙、4 层负载均衡等场景。XDP 程序并不是绕过了内核协议栈，它只是在内核协议栈之前处理数据包，而处理过的数据包还可以正常通过内核协议栈继续处理。

![图片](https://static001.geekbang.org/resource/image/3b/31/3b77fea948d6264bfb4b4c266526dd31.png?wh=768x420)

根据网卡和网卡驱动是否原生支持 XDP 程序，XDP 运行模式可以分为下面这三种：

-   **通用模式**：它不需要网卡和网卡驱动的支持，XDP 程序像常规的网络协议栈一样运行在内核中，性能相对较差，一般用于测试；
-   **原生模式**：它需要网卡驱动程序的支持，XDP 程序在网卡驱动程序的**早期**路径运行；
-   **卸载模式**：它需要网卡固件支持 XDP 卸载，XDP 程序**直接运行在网卡上**，而不再需要消耗主机的 CPU 资源，具有最好的性能。

XDP 程序在处理过网络包之后，都需要根据 eBPF 程序执行结果，决定数据包的去处。这些执行结果对应以下 5 种 XDP 程序结果码：

<img src="image-20230103231650873.png" alt="image-20230103231650873" style="zoom:50%;" />

通常来说，XDP 程序通过 `ip link` 命令加载到具体的网卡上，加载格式为：

```bash
# eth1 为网卡名
# xdpgeneric 设置运行模式为通用模式
# xdp-example.o 为编译后的 XDP 字节码
sudo ip link set dev eth1 xdpgeneric object xdp-example.o
```

而卸载 XDP 程序也是通过 `ip link` 命令，具体参数如下：

```bash
sudo ip link set veth1 xdpgeneric off
```

除了 `ip link`之外， BCC 也提供了方便的库函数，可以在同一个程序中管理 XDP 程序的生命周期：

```python
from bcc import BPF

# 编译XDP程序
b = BPF(src_file="xdp-example.c")
fn = b.load_func("xdp-example", BPF.XDP)

# 加载XDP程序到eth0网卡
device = "eth0"
b.attach_xdp(device, fn, 0)

# 其他处理逻辑
...

# 卸载XDP程序
b.remove_xdp(device)
```

##### 代码demo
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

int counter = 0;

// 宏SEC()定义了一个叫做xdp的段，可以在编译后的对象文件中看到。可以简单地认为它定义了一个eXpress Data Path（XDP）类型的eBPF程序
SEC("xdp")
// 实际的eBPF程序。在eBPF中，程序名就是函数名，所以这个程序被称为hello
int hello(struct xdp_md *ctx) {
    bpf_printk("Hello World %d", counter);
    counter++; 
    return XDP_PASS;
}

// 定义了许可证字符串，这是eBPF程序的一个关键需求。内核中的一些BPF助手函数被定义为"GPL only"。如果想使用这些函数中的任何一个，BPF代码必须声明为具有与gpl兼容的许可证。如果声明的许可与程序使用的函数不兼容，验证者将会反对
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

##### 编译：Makefile demo
```makefile
TARGETS = hello hello-func

all: $(TARGETS)
.PHONY: all

$(TARGETS): %: %.bpf.o 

%.bpf.o: %.bpf.c
	clang \
	    -target bpf \
		-I/usr/include/$(shell uname -m)-linux-gnu \
		-g \
	    -O2 -o $@ -c $<

clean: 
	- rm *.bpf.o
	- rm -f /sys/fs/bpf/hello 
	- rm -f /sys/fs/bpf/hello-func
```

将从 hello.bpf.c 中的源代码生成一个名为 hello.bpf.o 的对象文件

##### 检测ebpf 文件内容：`file` 命令
```bash
$ file hello.bpf.o
    hello.bpf.o: ELF 64-bit LSB relocatable, eBPF, version 1 (SYSV), with debug_info,
    not stripped
```
这表明它是一个ELF（可执行和可链接格式）文件，包含eBPF代码，用于64位平台的 LSB（最小有效位）架构。如果在编译步骤中使用了 `-g` 标志，它将包括调试信息。

将 `-g` 标志传递给 Clang，以便它包含 BTF 所必需的调试信息。 但是，`-g` 标志还会将 DWARF 调试信息添加到输出对象文件中，但 eBPF 程序不需要，因此可以通过运行以下命令将其删除来减小对象的大小：
```bash
llvm-strip -g <object file>
```

Clang需要 `-O2` 优化标志(级别2或更高)来生成将通过验证器的BPF字节码。


可以用 `llvm-objdump` 进一步检查这个对象，看看eBPF指令：

```bash
$ llvm-objdump -S hello.bpf.o
    # 确认hello.bpf.o是一个带有eBPF代码的64位ELF文件
    hello.bpf.o:    file format elf64-bpf
    # 对标记为xdp的部分的反汇编，它与C源代码中的SEC()定义相匹配
    Disassembly of section xdp:
    # 这个部分是一个名为hello的函数
    0000000000000000 <hello>:
    # 有五行 eBPF 字节码指令，对应于源行 bpf_printk("Hello World %d", counter");
    ;  bpf_printk("Hello World %d", counter");
        0:   18 06 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r6 = 0 ll
        2:   61 63 00 00 00 00 00 00 r3 = *(u32 *)(r6 + 0)
        3:   18 01 00 00 00 00 00 00 00 00 00 00 00 00 00 00 r1 = 0 ll
        5:   b7 02 00 00 0f 00 00 00 r2 = 15
        6:   85 00 00 00 06 00 00 00 call 6
    # 三行 eBPF 字节码指令递增计数器变量
    ; counter++;
        7:   61 61 00 00 00 00 00 00 r1 = *(u32 *)(r6 + 0)
        8:   07 01 00 00 01 00 00 00 r1 += 1
        9:   63 16 00 00 00 00 00 00 *(u32 *)(r6 + 0) = r1
    # 另外两行字节码是从源代码中生成的返回XDP_PASS;
    ;  return XDP_PASS;
       10:   b7 00 00 00 02 00 00 00 r0 = 2
       11:   95 00 00 00 00 00 00 00 exit
```

举例说明：

```bash
    5:   b7 02 00 00 0f 00 00 00 r2 = 15
```

- 操作码是0xb7，对应的伪码是dst = imm，代表 "将目的地设置为立即值"。
- 目标是由第二个字节0x02定义的，它意味着 "寄存器2"。这里的 "即时"（或字面）值是0x0f，也就是十进制的15, 可以理解为这条指令告诉内核 "将寄存器2设置为15",这对应于指令右侧看到的输出:r2 = 15
- 

##### 加载ebpf程序到内核
使用bpftool将程序加载到内核中 

```bash
$ bpftool prog load hello.bpf.o /sys/fs/bpf/hello
```

从编译的对象文件中加载eBPF程序，并将其 "挂"在 /sys/fs/bpf/hello 的位置上，这个命令没有输出响应表示成功，可以使用 ls 确认该程序已经就绪

这些被钉住的对象并不是持续存在于磁盘上的真实文件。它们是在一个伪文件系统上创建的，它的行为就像一个带有目录和文件的基于磁盘的常规文件系统。但它们被保存在内存中，这意味着它们不会在系统重启时保持原位

有一个 BPF(BPF_PROG_BIND_MAP) 系统调用将 map 与程序相关联，这样 map 就不会在**用户空间**加载程序退出并且不再持有map的文件描述符引用时立即被清理。

```bash
$ ls /sys/fs/bpf
hello

$ bpftool prog list
# 用 json 格式输出
$ bpftool prog show id 540 --pretty
{
    # 该程序的 ID 是 540
    "id": 540,
    # 类型字段说明这个程序可以使用 XDP 事件附加到网络接口
    "type": "xdp",
    # 程序的名字是hello，是源代码中的函数名
    "name": "hello",
    # 标签是该程序的另一个标识符，标签是程序指令的SHA（安全哈希算法）之和，可作为程序的另一个标识符。每次加载或卸载程序时，ID可能会有所不同，但标签将保持不变
    "tag": "d35b94b4c0c10efb",
    # 该程序是使用 GPL 兼容许可证定义的
    "gpl_compatible": true,
    # 时间戳显示程序何时加载
    "loaded_at": 1659461987,
    # 用户 ID 0（即 root）加载了该程序
    "uid": 0,
    # 该程序中有 96 个字节的已翻译 eBPF 字节码
    "bytes_xlated": 96,
    # 该程序已经过 JIT 编译，编译结果为 148 字节的机器码
    "jited": true,
    "bytes_jited": 148,
    # bytes _memlock 字段表示这个程序保留了 4096 字节的内存，不会被分页出去
    "bytes_memlock": 4096,
    # 该程序引用 ID 为 165 和 166 的 BPF 映射
    "map_ids": [165,166
    ],
    # 表示这个程序有一个BTF信息块。只有在使用-g标志进行编译时，该信息才会包含在目标文件中
    "btf_id": 254
}
```


##### 使用场景

场景一：防火墙、四层负载均衡等

由于 XDP 程序执行时 skb 都还没创建，开销非常低，因此效率非常高。适用于 DDoS 防御、四层负载均衡等场景。

XDP 就是通过 BPF hook 对内核进行运行时编程（run-time programming），但基于内核而不是绕过（bypass）内核。

##### Hook 位置：网络驱动
XDP 是在网络驱动中实现的，有专门的 TX/RX queue（native 方式）。

对于没有实现 XDP 的驱动，内核中实现了一个称为 “generic XDP” 的 fallback 实现， 见 [net/core/dev.c](https://github.com/torvalds/linux/blob/v5.8/net/core/dev.c)。
- Native XDP：处理的阶段非常早，在 skb 创建之前，因此性能非常高；
- Generic XDP：在 skb 创建之后，因此性能比前者差，但功能是一样的。

##### 程序签名

**传入参数**：`struct xdp_md *`
定义，非常轻量级：

```c
// include/uapi/linux/bpf.h

/* user accessible metadata for XDP packet hook */
struct xdp_md {
    __u32 data;
    __u32 data_end;
    __u32 data_meta;

    /* Below access go through struct xdp_rxq_info */
    __u32 ingress_ifindex; /* rxq->dev->ifindex */
    __u32 rx_queue_index;  /* rxq->queue_index  */
    __u32 egress_ifindex;  /* txq->dev->ifindex */
};
```

**返回值**：`enum xdp_action`
```c
// include/uapi/linux/bpf.h

enum xdp_action {
    XDP_ABORTED = 0,
    XDP_DROP,
    XDP_PASS,
    XDP_TX,
    XDP_REDIRECT,
};
```

**加载方式**：netlink socket
通过 netlink socket 消息 attach：

- 首先创建一个 netlink 类型的 socket：`socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)`
- 然后发送一个 `NLA_F_NESTED | 43` 类型的 netlink 消息，表示这是 XDP message。消息中包含 BPF fd, the interface index (ifindex) 等信息。

用 tc attach BPF 程序，其实背后使用的也是 netlink socket。

程序示例
1. [samples/bpf/bpf_load.c](https://github.com/torvalds/linux/blob/v5.8/samples/bpf/bpf_load.c)



#### **TC 程序**

经过 XDP 流量控制通过后到当前环节，当一个网络数据包到达这个点时，它将以 `sk_buff` 的形式出现在内核内存中。这是一个在整个内核的网络堆栈中使用的数据结构。TC子系统的目的是调节网络流量的调度方式.

**为什么XDP程序不也使用这个结构来处理它们的上下文?**。答案是，XDP的钩子发生在网络数据到达网络堆栈之前，在`sk_buff`结构被建立之前。

堆栈中给定的网络数据在两个方向之一流动：入口 ingress（从网络接口入站）或出口 egress（向网络接口出站）。 eBPF程序可以附加在任何一个方向，并将只影响该方向的流量。与XDP不同的是，它可以附加多个eBPF程序，这些程序将被依次处理。

##### 代码demo
```c
int tc_pingpong(struct __sk_buff *skb) {
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;
    // is_icmp_ping_request()函数解析数据包，不仅检查它是ICMP消息，而且检查它是一个回显(ping)请求
    if (!is_icmp_ping_request(data, data_end)) { 
        return TC_ACT_OK;
    }
    struct iphdr *iph = data + sizeof(struct ethhdr);
    struct icmphdr *icmp = data + sizeof(struct ethhdr) + sizeof(struct iphdr); swap_mac_addresses(skb);
    // 因为这个函数要向发送方发送一个响应，所以源地址和目的地址需要交换
    swap_ip_addresses(skb);
    // Change the type of the ICMP packet to 0 (ICMP Echo Reply) // (was 8 for ICMP Echo request)
    // 通过改变ICMP报头中的类型字段，将其转换为回显响应
    update_icmp_type(skb, 8, 0);
    // Redirecting a clone of the modified skb back to the interface // it arrived on
    // 将修改后的skb的克隆重定向回接口
    bpf_clone_redirect(skb, skb->ifindex, 0);
    // 发送响应之前克隆了数据包，因此应该丢弃原始数据包
    return TC_ACT_SHOT; 
}
```
[源代码](https://github.com/lizrice/learning-ebpf/blob/main/chapter8/network.bpf.c)


类型定义为 `BPF_PROG_TYPE_SCHED_CLS` 和 `BPF_PROG_TYPE_SCHED_ACT`，分别作为 [Linux 流量控制](https://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html) 的分类器和执行器。Linux 流量控制通过网卡队列、排队规则、分类器、过滤器以及执行器等，实现了对网络流量的整形调度和带宽控制.

- `BPF_PROG_TYPE_SCHED_CLS`：tc classifier，分类器
- `BPF_PROG_TYPE_SCHED_ACT`：tc action，动作

1. BPF_PROG_TYPE_SCHED_CLS
##### 使用场景
场景一：tc 分类器

`tc(8)` 命令支持 eBPF，因此能直接将 BPF 程序作为 classifiers 和 actions 加载到 ingress/egress hook 点。

如何使用 tc BPF 提供的能力，参考 man8: tc-bpf

##### Hook 位置：`sch_handle_ingress()`/`sch_handle_egress()`
`sch_handle_ingress()/egress()` 会调用到 `tcf_classify()`，
- 对于 ingress，通过网络设备的 receive 方法做流量分类，这个处理位置在网卡驱动处理之后，在内核协议栈（IP 层）处理之前。
- 对于 egress，将包交给设备队列（device queue）发送之前，执行 BPF 程序。

##### 加载方式：tc 命令（背后使用 netlink）
步骤：
- 为网络设备添加分类器（classifier/qdisc）：创建一个 “clsact” qdisc
- 为网络设备添加过滤器（filter）：需要指定方向（egress/ingress）、目标文件、ELF section 等选项

例如，
```bash
$ tc qdisc add dev eth0 clsact
$ tc filter add dev eth0 egress bpf da obj toy-proxy-bpf.o sec egress
```

加载过程分为 tc 前端和内核 bpf 后端两部分，中间通过 netlink socket 通信，源码分析见 [Firewalling with BPF/XDP: Examples and Deep Dive](http://arthurchiao.art/blog/firewalling-with-bpf-xdp/)

###### 程序示例
- [Firewalling with BPF/XDP: Examples and Deep Dive](http://arthurchiao.art/blog/firewalling-with-bpf-xdp/)
- [Cracking Kubernetes Node Proxy (aka kube-proxy)](http://arthurchiao.art/blog/cracking-k8s-node-proxy/)

2. BPF_PROG_TYPE_SCHED_ACT
使用方式与 BPF_PROG_TYPE_SCHED_CLS 类似，但用作 TC action


![图片](https://static001.geekbang.org/resource/image/3c/69/3c445830476ed2f32d71e99309b26369.png?wh=1369x1049)

>   [linux流量控制参考链接](https://tldp.org/HOWTO/Traffic-Control-HOWTO/index.html)

得益于内核 v4.4 引入的 [direct-action](https://docs.cilium.io/en/v1.8/bpf/#tc-traffic-control) 模式，TC 程序可以直接在一个程序内完成分类和执行的动作，而无需再调用其他的 TC 排队规则和分类器，具体如下图所示：

<img src=image-20230103232707852.png>


同 XDP 程序相比，TC 程序可以**直接获取内核解析后的网络报文数据结构 `sk_buff`（XDP 则是 `xdp_buff`）**，并且可**在网卡的接收和发送两个方向上执行**（**XDP 则只能用于接收**）。

##### **TC 程序的执行位置**：

-   对于**接收**的网络包，TC 程序在网卡接收（GRO）之后、协议栈处理（包括 IP 层处理和 iptables 等）之前执行；
-   对于**发送**的网络包，TC 程序在协议栈处理（包括 IP 层处理和 iptables 等）之后、数据包发送到网卡队列（GSO）之前执行。

除此之外，由于 TC 运行在内核协议栈中，不需要网卡驱动程序做任何改动，因而可以挂载到任意类型的网卡设备（包括容器等使用的虚拟网卡）上。

同 XDP 程序一样，TC eBPF 程序也可以通过 Linux 命令行工具来加载到网卡上，不过相应的工具要换成 `tc`。可以通过下面的命令，分别加载接收和发送方向的 eBPF 程序：

```bash
# 创建 clsact 类型的排队规则
sudo tc qdisc add dev eth0 clsact

# 加载接收方向的 eBPF 程序
sudo tc filter add dev eth0 ingress bpf da obj tc-example.o sec ingress

# 加载发送方向的 eBPF 程序
sudo tc filter add dev eth0 egress bpf da obj tc-example.o sec egress
```



#### **套接字程序**

用于过滤、观测或重定向套接字网络包. 根据类型的不同，套接字 eBPF 程序可以挂载到套接字（socket）、控制组（cgroup ）以及网络命名空间（netns）等各个位置。

常见的套接字程序类型，以及它们的应用场景和挂载方法：

<img src="image-20230103233344938.png" alt="image-20230103233344938" style="zoom:50%;" />



#### **cgroup 程序**

用于**对 cgroup 内所有进程的网络过滤、套接字选项以及转发等进行动态控制**，它最典型的应用场景是对容器中运行的多个进程进行网络控制。

cgroup BPF 用于在 cgroup 级别对进程、socket、设备文件 （device file）等进行动态控制，
- 处理资源分配，例如 CPU、网络带宽等。
- 系统资源权限控制（allowing or denying）。
- 控制访问权限（allow or deny），程序的返回结果只有两种：
  - 放行
  - 禁止（导致随后包被丢弃）

完整的 cgroups BPF hook 列表见 前面 enum bpf_attach_type 列表，其中的 BPF_CGROUP_*。


常用类型以及应用场景：

<img src="image-20230103233456809.png" alt="image-20230103233456809" style="zoom:50%;" />


这些类型的 BPF 程序都可以通过 BPF 系统调用的 `BPF_PROG_ATTACH` 命令来进行挂载，并设置[挂载类型](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L942)为匹配的 `BPF_CGROUP_xxx` 类型。比如，在挂载 `BPF_PROG_TYPE_CGROUP_DEVICE` 类型的 BPF 程序时，需要设置 `bpf_attach_type` 为 `BPF_CGROUP_DEVICE`：

```c
union bpf_attr attr = {};
attr.target_fd = target_fd;            // cgroup文件描述符
attr.attach_bpf_fd = prog_fd;          // BPF程序文件描述符
attr.attach_type = BPF_CGROUP_DEVICE;  // 挂载类型为BPF_CGROUP_DEVICE

if (bpf(BPF_PROG_ATTACH, &attr, sizeof(attr)) < 0) {
  return -errno;
}

```

##### group BPF 通用调用栈
1. 创建 socket 时初始化其 cgroupv2 配置
```c
// https://github.com/torvalds/linux/blob/v5.10/net/core/sock.c#L1715
/**
 *    sk_alloc - All socket objects are allocated here
 *    @net: the applicable net namespace
 *    @family: protocol family
 *    @priority: for allocation (%GFP_KERNEL, %GFP_ATOMIC, etc)
 *    @prot: struct proto associated with this new sock instance
 *    @kern: is this to be a kernel socket?
 */
struct sock *sk_alloc(struct net *net, int family, gfp_t priority, struct proto *prot, int kern)
{
    struct sock *sk = sk_prot_alloc(prot, priority | __GFP_ZERO, family);
    if (sk) {
        sk->sk_family = family;
        sk->sk_prot = sk->sk_prot_creator = prot;
        sk->sk_kern_sock = kern;
        sk->sk_net_refcnt = kern ? 0 : 1;
        if (likely(sk->sk_net_refcnt)) {
            get_net(net);                          // 网络命名空间
            sock_inuse_add(net, 1);
        }

        sock_net_set(sk, net);
        refcount_set(&sk->sk_wmem_alloc, 1);

        mem_cgroup_sk_alloc(sk);                   // memory cgroup 信息单独维护
        cgroup_sk_alloc(&sk->sk_cgrp_data);        // per-socket cgroup 信息，包括了 memory cgroup 之外
                                                   // 该 socket 的 cgroup 信息
        sock_update_classid(&sk->sk_cgrp_data);
        sock_update_netprioidx(&sk->sk_cgrp_data);
        sk_tx_queue_clear(sk);
    }

    return sk;
}
```
可以看到，创建 socket 时会初始化其所属的 cgroup 信息，因此后面就能 在 cgroup 级别监听 socket 事件或拦截 socket 操作。

2. 入向（ingress）hook 处理
很多 hook 点会执行到下面两个宏来执行 cgroup BPF 代码：

```c
// include/linux/bpf-cgroup.h

#define BPF_CGROUP_RUN_PROG_INET_INGRESS(sk, skb)                  \
({                                                                 \
    int __ret = 0;                                                 \
    if (cgroup_bpf_enabled)                                        \
        __ret = __cgroup_bpf_run_filter_skb(sk, skb,               \
                            BPF_CGROUP_INET_INGRESS);              \
                                                                   \
    __ret;                                                         \
})
```

函数的定义：

```c
// https://github.com/torvalds/linux/blob/v5.10/kernel/bpf/cgroup.c#L987

int __cgroup_bpf_run_filter_skb(struct sock *sk, struct sk_buff *skb, enum bpf_attach_type type)
{
    unsigned int offset = skb->data - skb_network_header(skb);
    struct sock *save_sk;
    void *saved_data_end;
    struct cgroup *cgrp;
    int ret;

    cgrp = sock_cgroup_ptr(&sk->sk_cgrp_data); // 获取 socket cgroup 信息
    save_sk = skb->sk;
    skb->sk = sk;
    __skb_push(skb, offset);

    bpf_compute_and_save_data_end(skb, &saved_data_end);

    if (type == BPF_CGROUP_INET_EGRESS) {
        ret = BPF_PROG_CGROUP_INET_EGRESS_RUN_ARRAY(cgrp->bpf.effective[type], skb, __bpf_prog_run_save_cb);
    } else {
        ret = BPF_PROG_RUN_ARRAY(cgrp->bpf.effective[type], skb, __bpf_prog_run_save_cb);
        ret = (ret == 1 ? 0 : -EPERM);
    }
    bpf_restore_data_end(skb, saved_data_end);
    __skb_pull(skb, offset);
    skb->sk = save_sk;

    return ret;
}
```
3. 出向（egress）hook 处理
```c
// include/linux/bpf-cgroup.h

#define BPF_CGROUP_RUN_SK_PROG(sk, type)                       \
({                                                                 \
    int __ret = 0;                                                 \
    if (cgroup_bpf_enabled) {                                      \
        __ret = __cgroup_bpf_run_filter_sk(sk, type);              \
    }                                                              \
    __ret;                                                         \
})
```

函数的定义：
```c
// https://github.com/torvalds/linux/blob/v5.10/kernel/bpf/cgroup.c#L1040
int __cgroup_bpf_run_filter_sk(struct sock *sk, enum bpf_attach_type type)
{
    struct cgroup *cgrp = sock_cgroup_ptr(&sk->sk_cgrp_data); // 获取 socket cgroup 信息

    ret = BPF_PROG_RUN_ARRAY(cgrp->bpf.effective[type], sk, BPF_PROG_RUN);
    return ret == 1 ? 0 : -EPERM;
}
```

##### `BPF_PROG_TYPE_CGROUP_SKB`

**使用场景**

**场景一：在 cgroup 级别：放行/丢弃数据包**

在 IP egress/ingress 层禁止或允许网络访问。

**Hook 位置**

入向：`__sk_receive_skb/tcp_v4_rcv->tcp_filter/udp_queue_rcv_one_skb -> sk_filter_trim_cap()`

对于 ingress，上述三个函数会分别从 IP/TCP/UDP 处理逻辑里调用到 sk_filter_trim_cap()， 后者又会调用 `BPF_CGROUP_RUN_PROG_INET_INGRESS(sk, skb)`

```c
// https://github.com/torvalds/linux/blob/v5.10/net/core/filter.c#L120

/**
 *    sk_filter_trim_cap - run a packet through a socket filter
 *    @cap: limit on how short the eBPF program may trim the packet
 *
 * Run the eBPF program and then cut skb->data to correct size returned by
 * the program. If pkt_len is 0 we toss packet. If skb->len is smaller
 * than pkt_len we keep whole skb->data. This is the socket level
 * wrapper to BPF_PROG_RUN. It returns 0 if the packet should
 * be accepted or -EPERM if the packet should be tossed.
 *
 */
int sk_filter_trim_cap(struct sock *sk, struct sk_buff *skb, unsigned int cap)
{
    BPF_CGROUP_RUN_PROG_INET_INGRESS(sk, skb); // 上面有介绍
    security_sock_rcv_skb(sk, skb);

    struct sk_filter *filter = rcu_dereference(sk->sk_filter);
    if (filter) {
        struct sock *save_sk = skb->sk;
        unsigned int pkt_len;

        skb->sk = sk;
        pkt_len = bpf_prog_run_save_cb(filter->prog, skb);
        skb->sk = save_sk;
        err = pkt_len ? pskb_trim(skb, max(cap, pkt_len)) : -EPERM;
    }

    return err;
}
```

如果返回值非零，调用方（例如 `__sk_receive_skb()`）随后会将包丢弃并释放。

**出向：`ip[6]_finish_output()`**

egress 是类似的，但在 `ip[6]_finish_output()` 中。

**程序签名**

传入参数：`struct sk_buff *skb`

返回值
- 1：放行；
- 其他任何值：会使 `__cgroup_bpf_run_filter_skb()` 返回 `-EPERM`，这会进一步返回给调用方，告诉它们应该丢弃该包。

**加载方式：attach 到 cgroup 文件描述符**

根据 BPF attach 的 hook 位置，选择合适的 attach 类型：
- `BPF_CGROUP_INET_INGRESS`
- `BPF_CGROUP_INET_EGRESS`



##### `BPF_PROG_TYPE_CGROUP_SOCK`

**使用场景**

**场景一**：在 cgroup 级别：触发 socket 操作时拒绝/放行网络访问

这里的 socket 相关事件包括 BPF_CGROUP_INET_SOCK_CREATE、BPF_CGROUP_SOCK_OPS。

**程序签名**

传入参数：`struct sk_buff *skb`

**返回值**

跟前面一样，程序返回 1 表示允许访问。 返回其他值会导致 `__cgroup_bpf_run_filter_sk()` 返回 `-EPERM`，调用方收到这个返回值会将包丢弃。

**触发执行：`inet_create()`**
Socket 创建时会执行 `inet_create()`，里面会调用 `BPF_CGROUP_RUN_PROG_INET_SOCK()` ，如果该函数执行失败，socket 就会被释放。

**加载方式**：attach 到 cgroup 文件描述符


##### `BPF_PROG_TYPE_CGROUP_DEVICE`

**使用场景**

**场景一**：设备文件（device file）访问控制

**程序签名**

传入参数：`struct bpf_cgroup_dev_ctx *`
```c
// https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L4833

struct bpf_cgroup_dev_ctx {
    __u32 access_type; /* encoded as (BPF_DEVCG_ACC_* << 16) | BPF_DEVCG_DEV_* */
    __u32 major;
    __u32 minor;
};
```

字段含义：
- access_type：访问操作的类型，例如 mknod/read/write；
- major 和 minor：主次设备号；

**返回值**
- 0：访问失败（-EPERM）
- 其他值：访问成功。

**触发执行**：创建或访问设备文件时

**加载方式**：attach 到 cgroup 文件描述符

指定 attach 类型为 BPF_CGROUP_DEVICE。

**程序示例**

内核测试用例：
- [tools/testing/selftests/bpf/progs/dev_cgroup.c](https://github.com/torvalds/linux/blob/v5.10/tools/testing/selftests/bpf/progs/dev_cgroup.c)
- [tools/testing/selftests/bpf/test_dev_cgroup.c](https://github.com/torvalds/linux/blob/v5.10/tools/testing/selftests/bpf/test_dev_cgroup.c)

### 其他类 eBPF 程序

<img src="image-20230103233939074.png" alt="image-20230103233939074" style="zoom:50%;" />



## 查询跟踪点：

### 利用系统文件查看内核跟踪点：

为了方便调试，内核把所有函数以及非栈变量的地址都抽取到了 `/proc/kallsyms` ，这些符号表不仅包含了内核函数，还包含了非栈数据变量。而且，并不是所有的内核函数都是可跟踪的，只有显式导出的内核函数才可以被 eBPF 进行动态跟踪。因而，通常并不直接从内核符号表查询可跟踪点。内核[调试文件系统](https://www.kernel.org/doc/html/latest/filesystems/debugfs.html)还向用户空间提供了内核调试所需的基本信息，如内核符号列表、跟踪点、函数跟踪（ftrace）状态以及参数格式等。可以在终端中执行 `sudo ls /sys/kernel/debug` 来查询内核调试文件系统的具体信息。

比如下面命令可以查询 `execve` 系统调用的参数格式

```bash
sudo cat /sys/kernel/debug/tracing/events/syscalls/sys_enter_execve/format
```



如果碰到了 `/sys/kernel/debug` 目录不存在的错误，说明系统没有自动挂载调试文件系统。只需要执行下面的 mount 命令就可以挂载它：

```bash
sudo mount -t debugfs debugfs /sys/kernel/debug
```



有了调试文件系统，就可以从 `/sys/kernel/debug/tracing` 中找到所有内核预定义的跟踪点，进而可以在需要时把 eBPF 程序挂载到对应的跟踪点



### 利用[perf](https://man7.org/linux/man-pages/man1/perf.1.html) 来查询性能事件列表：

```bash
# 可以不带参数查询所有的性能事件，也可以加入可选的事件类型参数进行过滤
sudo perf list [hw|sw|cache|tracepoint|pmu|sdt|metric|metricgroup]
```



### **利用 bpftrace 查询跟踪点**：

bpftrace 在 eBPF 和 BCC 之上构建了一个简化的跟踪语言，通过简单的几行脚本，就可以实现复杂的跟踪功能。并且，多行的跟踪指令也可以放到脚本文件中执行（脚本后缀通常为 `.bt`）

bpftrace 会把开发的脚本借助 BCC 编译加载到内核中执行，再通过 BPF 映射获取执行的结果：

<img src="https://static001.geekbang.org/resource/image/17/fb/175853e38141433058e05770285ee5fb.png?wh=1500x1050" alt="图片" style="zoom:50%;" />

#### bpftrace 安装：

```bash
# Ubuntu 19.04
sudo apt-get install -y bpftrace

# RHEL8/CentOS8
sudo dnf install -y bpftrace
```



#### 查询内核插桩和跟踪点:

```bash
# 查询所有内核插桩和跟踪点
sudo bpftrace -l

# 使用通配符查询所有的系统调用跟踪点
sudo bpftrace -l 'tracepoint:syscalls:*'

# 使用通配符查询所有名字包含"execve"的跟踪点
sudo bpftrace -l '*execve*'

kprobe:__ia32_compat_sys_execve
kprobe:__ia32_compat_sys_execveat
kprobe:__ia32_sys_execve
kprobe:__ia32_sys_execveat
kprobe:__x32_compat_sys_execve
kprobe:__x32_compat_sys_execveat
kprobe:__x64_sys_execve
kprobe:__x64_sys_execveat
kprobe:audit_log_execve_info
kprobe:bprm_execve
kprobe:do_execveat_common.isra.0
kprobe:kernel_execve
tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_enter_execveat
tracepoint:syscalls:sys_exit_execve
tracepoint:syscalls:sys_exit_execveat
# 函数可以分为内核插桩（kprobe）和跟踪点（tracepoint）两类;
# 内核插桩属于不稳定接口，而跟踪点则是稳定接口。因而，在内核插桩和跟踪点两者都可用的情况下，应该选择更稳定的跟踪点，以保证 eBPF 程序的可移植性
```

可以加上 `-v` 参数查询函数的入口参数或返回值

```bash
# 查询系统调用execve入口参数格式
$ sudo bpftrace -lv tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_enter_execve
    int __syscall_nr
    const char * filename
    const char *const * argv
    const char *const * envp

# 查询系统调用execve返回值格式
$ sudo bpftrace -lv tracepoint:syscalls:sys_exit_execve
tracepoint:syscalls:sys_exit_execve
    int __syscall_nr
    long ret
```

>   根据函数参数以及返回值确定哪个更能满足业务需求



## 开发ebpf程序三种方式比较：

-   **bpftrace 通常用在快速排查和定位系统上，它支持用单行脚本的方式来快速开发并执行一个 eBPF 程序**。不过，bpftrace 的功能有限，不支持特别复杂的 eBPF 程序，也依赖于 BCC 和 LLVM 动态编译执行。

-   **BCC 通常用在开发复杂的 eBPF 程序中，其内置的各种小工具也是目前应用最为广泛的 eBPF 小程序**。不过，BCC 也不是完美的，它依赖于 LLVM 和内核头文件才可以动态编译和加载 eBPF 程序。

-   **libbpf 是从内核中抽离出来的标准库，用它开发的 eBPF 程序可以直接分发执行，这样就不需要每台机器都安装 LLVM 和内核头文件了**。不过，它要求内核开启 BTF 特性，需要非常新的发行版才会默认开启（如 RHEL 8.2+ 和 Ubuntu 20.10+ 等）。

libbpf 提供了加载 eBPF 程序和 map 到内核的函数。但它在可移植性方面也起着重要作用：依据 BTF 信息来调整 eBPF 代码，以补偿编译时存在的数据结构与目标机器上的数据结构之间的任何差异。实现 libbpf 库使eBPF程序能够在它们运行的目标内核上使用数据结构布局，即使该布局与编译代码的内核不同。为此，libbpf需要 Clang 生成的 BPF CO-RE 重定位信息作为编译过程的一部分。

当它编译 eBPF 程序时，它包括所谓的 BTF 重定向（relocations），这是 libbpf 在加载 BPF 程序和 map 到内核时用来知道要调整哪些内容。

可以从 linux/bpff.h 头文件中的结构体 `bpf_core_relo`的定义中了解更多关于重定位的工作原理:
```c
struct bpf_core_relo { 
    __u32 insn_off; 
    __u32 type_id;
    __u32 access_str_off;
    enum bpf_core_relo_kind kind;
};
```
eBPF程序的 CO-RE 重定位数据由每个需要重定位的指令的这些结构之一组成。假设该指令正在将寄存器设置为结构中某个字段的值。该指令的 `bpf_core_relo` 结构（由 `insn_off` 字段标识）编码了该结构的BTF类型（`type_id`字段），并指出该字段相对于该结构的访问方式（`access_str_off`）。

内核数据结构的重定位数据由 Clang 自动生成并编码在 ELF 对象文件中。 可以在 vmlinux.h 文件的开头附近找到以下一行，它导致 Clang 执行此操作：
```c
#pragma clang attribute push (__attribute__((preserve_access_index)), \
    apply_to = record)
```
preserve_access_index 属性告诉 Clang 为类型定义生成 BPF CO-RE 重定位。clang 属性 push 表示，该属性应该应用于直到出现 clang 的 pop 属性的所有定义。这意味着Clang为vmlinux.h中定义的所有类型生成了重定位信息。




eBPF 程序段(部分)：
使用 libbpf 要求每个 eBPF 程序都用定义程序类型的 `SEC()` 宏标记，如下所示：
```c
SEC("kprobe")
```
这会在编译的 **ELF** 对象中产生一个名为 kprobe 的段（部分），因此 libbpf 知道将其作为 `BPF_PROG_TYPE_KPROBE` 加载。
根据程序的类型，也可以使用部分名称来指定程序将被附加到什么事件。libbpf库将使用这些信息来自动设置附件，而不是在用户空间的代码中明确地进行设置。因此，举例来说，如果要在基于ARM的机器上自动附加到execve系统调用的kprobe，可以这样指定部分：
```c
SEC("kprobe/__arm64_sys_execve")
```
这需要提前知道该架构上 `syscall` 的函数名（或者通过查看目标机器上的/proc/kallsyms文件来弄清楚，该文件列出了所有的内核符号，包括其函数名）。但是libbpf可以通过 `k(ret)syscall` 部分的名称使其变得更加简单，它告诉加载器自动附加到架构特定函数中的kprobe：
```c
SEC("ksyscall/execve")
```
SEC定义声明 eBPF 程序应该附加到哪里，然后跟的是程序本身。


### 案例1: 排查短时进程

利用内核跟踪点排查短时进程：

​	在排查系统 CPU 使用率高的问题时，会出现明明通过 `top` 命令发现系统的 CPU 使用率（特别是用户 CPU 使用率）特别高，但通过 `ps`、`pidstat` 等工具都找不出 CPU 使用率高的进程的情况，

一般情况下，这类问题很可能是以下两个原因导致的：

-   第一，应用程序里面直接调用其他二进制程序，并且这些程序的运行时间很短，通过 `top` 工具不容易发现；
-   第二，应用程序自身在不停地崩溃重启中，且重启间隔较短，启动过程中资源的初始化导致了高 CPU 使用率。

使用 `top`、`ps` 等性能工具很难发现这类短时进程，这是因为它们都只会按照给定的时间间隔采样，而不会实时采集到所有新创建的进程，**利用 eBPF 的事件触发机制，跟踪内核每次新创建的进程**，这样就可以揪出这些短时进程。

要跟踪内核新创建的进程，首先得找到要跟踪的内核函数或跟踪点。创建一个新进程通常需要调用 `fork()` 和 `execve()` 这两个标准函数，它们的调用过程如下图所示：

<img src="https://static001.geekbang.org/resource/image/ef/1c/efda99288b5366ca24a00f374c6fba1c.jpg?wh=1920x2645" alt="图片" style="zoom:25%;" />

我们要关心的主要是新创建进程的基本信息，而像进程名称和参数等信息都在 `execve()` 的参数里，所以我们就要找出 `execve()` 所对应的内核函数或跟踪点，借助刚才提到的 `bpftrace` 工具，可以执行下面的命令，查询所有包含 `execve` 关键字的跟踪点：

```bash
# sudo bpftrace -l '*execve*'
kprobe:__ia32_compat_sys_execve
kprobe:__ia32_compat_sys_execveat
kprobe:__ia32_sys_execve
kprobe:__ia32_sys_execveat
kprobe:__x32_compat_sys_execve
kprobe:__x32_compat_sys_execveat
kprobe:__x64_sys_execve
kprobe:__x64_sys_execveat
kprobe:audit_log_execve_info
kprobe:bprm_execve
kprobe:do_execveat_common.isra.0
kprobe:kernel_execve
tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_enter_execveat
tracepoint:syscalls:sys_exit_execve
tracepoint:syscalls:sys_exit_execveat
```

发现这些函数可以分为内核插桩（kprobe）和跟踪点（tracepoint）两类, 内核插桩属于不稳定接口，而跟踪点则是稳定接口。因而，排除掉 `kprobe` 类型之后剩下的就是想要的 eBPF 跟踪点，`sys_enter_` 和 `sys_exit_` 分别表示在系统调用的入口和出口执行。

只有跟踪点的列表还不够，因为还想知道具体启动的进程名称、命令行选项以及返回值，可以通过 bpftrace 来查询 

```bash
# 查询sys_enter_execve入口参数
$ sudo bpftrace -lv tracepoint:syscalls:sys_enter_execve
tracepoint:syscalls:sys_enter_execve
    int __syscall_nr
    const char * filename
    const char *const * argv
    const char *const * envp

# 查询sys_exit_execve返回值
$ sudo bpftrace -lv tracepoint:syscalls:sys_exit_execve
tracepoint:syscalls:sys_exit_execve
    int __syscall_nr
    long ret

# 查询sys_enter_execveat入口参数
$ sudo bpftrace -lv tracepoint:syscalls:sys_enter_execveat
tracepoint:syscalls:sys_enter_execveat
    int __syscall_nr
    int fd
    const char * filename
    const char *const * argv
    const char *const * envp
    int flags

# 查询sys_exit_execveat返回值
$ sudo bpftrace -lv tracepoint:syscalls:sys_exit_execveat
tracepoint:syscalls:sys_exit_execveat
    int __syscall_nr
    long ret
```

### 三种方式对应的解决办法：

#### bpftrace 方法：

先忽略返回值，只看入口参数。打开一个终端，执行下面的 bpftrace 命令：

```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve,tracepoint:syscalls:sys_enter_execveat { printf("%-6d %-8s", pid, comm); join(args->argv);}'
```

这个命令中的具体内容含义如下：

-   `bpftrace -e` 表示直接从后面的字符串参数中读入 bpftrace 程序（除此之外，它还支持从文件中读入 bpftrace 程序）；
-   `tracepoint:syscalls:sys_enter_execve,tracepoint:syscalls:sys_enter_execveat` 表示用逗号分隔的多个跟踪点，其后的中括号表示跟踪点的处理函数；
-   `printf()` 表示向终端中打印字符串，其用法类似于 C 语言中的 `printf()` 函数；
-   `pid` 和 `comm` 是 bpftrace 内置的变量，分别表示进程 PID 和进程名称（你可以在其[官方文档](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#1-builtins)中找到其他的内置变量）；
-   `join(args->argv)` 表示把字符串数组格式的参数用空格拼接起来，再打印到终端中。对于跟踪点来说，你可以使用 `args->参数名` 的方式直接读取参数（比如这里的 `args->argv` 就是读取系统调用中的 `argv` 参数）。

在另一个终端中执行 `ls` 命令，然后你会在第一个终端中看到如下的输出：

```plain
Attaching 2 probes...
157286 zsh     ls --color=tty
```

可以发现，当前系统使用了 `zsh` 终端，在 `zsh` 终端中执行了`ls` 命令。现在已经可以通过一个简单的单行命令来跟踪短时进程问题了。不过，这个程序还不够完善，因为它的返回值还没有处理。那么，如何处理返回值呢？

一个最简单的思路就是在系统调用的入口把参数保存到 BPF 映射中，然后再在系统调用出口获取返回值后一起输出。比如，你可以尝试执行下面的命令，把新进程的参数存入哈希映射中：

```bash
# 其中，tid表示线程ID，@execs[tid]表示创建一个哈希映射
$ sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve,tracepoint:syscalls:sys_enter_execveat {@execs[tid] = join(args->argv);}'
```

很遗憾，这条命令并不能正常运行。根据下面的错误信息，你可以发现，`join()` 这个内置函数没有返回字符串，不能用来赋值：

```plain
stdin:1:92-108: ERROR: join() should not be used in an assignment or as a map key
```

实际上，在 bpftrace 的 GitHub 页面上，已经有其他用户汇报了同样的[问题](https://github.com/iovisor/bpftrace/issues/1390)，并且到现在还是没有解决。可见，bpftrace 本身并不适用于所有的 eBPF 应用。如果是复杂的应用，还是推荐使用 BCC 或者 libbpf 开发。

#### BCC 方法：

使用 BCC 开发的 eBPF 程序包含两部分：

-   第一部分是用 C 语言开发的 eBPF 程序。在 eBPF 程序中，可以利用 BCC 提供的[库函数和宏定义](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)简化你的处理逻辑。
-   第二部分是用 Python 语言开发的前端界面，其中包含 eBPF 程序加载、挂载到内核函数和跟踪点，以及通过 BPF 映射获取和打印执行结果等部分。在前端程序中，同样可以利用 BCC 库来访问 BPF 映射。

BCC 依赖于 LLVM 和内核头文件才可以动态编译和加载 eBPF 程序，而出于安全策略的需要，在生产环境中通常又不允许安装这些开发工具。这个难题应该怎么克服呢？

-   一种很容易想到的方法是把 BCC 和开发工具都安装到容器中，容器本身不提供对外服务，这样可以降低安全风险。
-   另外一种方法就是参考内核中的 [eBPF 示例](https://elixir.bootlin.com/linux/v5.13/source/samples/bpf)，开发一个匹配当前内核版本的 eBPF 程序，并编译为字节码，再分发到生产环境中。
-   除此之外，如果内核已经支持了 BPF 类型格式 (BTF)，推荐使用从内核源码中抽离出来的 libbpf 进行开发，这样可以借助 BTF 和 CO-RE 获得更好的移植性。

**示例代码：**

```c
/* 跟踪执行系统调用 */
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>
#include <linux/fs.h>

// 定义参数长度和参数个数常量
// consts for arguments (确保堆栈大小限制在512以下)
#define ARGSIZE 64
#define TOTAL_MAX_ARGS 5
#define FULL_MAX_ARGS_ARR (TOTAL_MAX_ARGS * ARGSIZE)
#define LAST_ARG (FULL_MAX_ARGS_ARR - ARGSIZE)

// perf event map (向用户空间共享数据) and hash map (在跟踪点tracepoints之间共享数据)
struct data_t {
	u32 pid;
	char comm[TASK_COMM_LEN];
	int retval;
	unsigned int args_size;
	char argv[FULL_MAX_ARGS_ARR];
};
// 定义了一个性能事件映射
BPF_PERF_OUTPUT(events);
// 定义了一个哈希映射，其键为 32 位的进程 PID，而值则是进程基本信息 data_t
BPF_HASH(tasks, u32, struct data_t);

// 从用户空间读取字符串的辅助函数
static int __bpf_read_arg_str(struct data_t *data, const char *ptr)
{
	if (data->args_size > LAST_ARG) {
		return -1;
	}

	int ret = bpf_probe_read_user_str(&data->argv[data->args_size], ARGSIZE,
					  (void *)ptr);
	if (ret > ARGSIZE || ret < 0) {
		return -1;
	}
	// increase the args size. the first tailing '\0' is not counted and hence it
	// would be overwritten by the next call.
	data->args_size += (ret - 1);

	return 0;
}

// 定义sys_enter_execve跟踪点处理函数
TRACEPOINT_PROBE(syscalls, sys_enter_execve)
{
	// variables definitions
	unsigned int ret = 0;
	const char **argv = (const char **)(args->argv);

	// 获取pid和进程名称
	struct data_t data = { };
	u32 pid = bpf_get_current_pid_tgid(); // 取低32位为进程PID
	data.pid = pid;
	bpf_get_current_comm(&data.comm, sizeof(data.comm));

	// 获取第一个参数（即可执行文件的名字）
	if (__bpf_read_arg_str(&data, (const char *)argv[0]) < 0) {
		goto out;
	}
	// 获取其他参数（限定最多5个 跳过第一个参数，因为它已经被读取）
#pragma unroll // 告诉编译器，把源码中的循环自动展开。这就避免了最终的字节码中包含循环
	for (int i = 1; i < TOTAL_MAX_ARGS; i++) {
		if (__bpf_read_arg_str(&data, (const char *)argv[i]) < 0) {
			goto out;
		}
	}

 out:
	// 存储到哈希映射中
	tasks.update(&pid, &data);
	return 0;
}

// 定义sys_exit_execve跟踪点处理函数
TRACEPOINT_PROBE(syscalls, sys_exit_execve)
{
	// 从哈希映射中查询进程基本信息
	u32 pid = bpf_get_current_pid_tgid();
	struct data_t *data = tasks.lookup(&pid);

	// 填充返回值并提交到性能事件映射中
	if (data != NULL) {
		data->retval = args->ret;
		events.perf_submit(args, data, sizeof(struct data_t));

		// 最后清理进程信息
		tasks.delete(&pid);
	}

	return 0;
}
```

-   `struct data_t` 定义了一个包含进程基本信息的数据结构，它将用在哈希映射的值中（其中的参数大小 `args_size` 会在读取参数内容的时候用到）；
-   `BPF_PERF_OUTPUT(events)` 定义了一个性能事件映射；
-   `BPF_HASH(tasks, u32, struct data_t)` 定义了一个哈希映射，其键为 32 位的进程 PID，而值则是进程基本信息 `data_t`。
-   `RACEPOINT_PROBE(category, event)` 来定义一个跟踪点处理函数。BCC 会将所有的参数放入 `args` 这个变量中，这样使用 `args-><参数名>` 就可以访问跟踪点的参数值
-   `bpf_probe_read_user_str()` 返回的是包含字符串结束符 `\0` 的长度。为了拼接所有的字符串，在计算已读取参数长度的时候，需要把 `\0` 排除在外。
-   `&data->argv[data->args_size]` 用来获取要存放参数的位置指针，这是为了把多个参数拼接到一起。
-   在调用 `bpf_probe_read_user_str()` 前后，需要对指针位置和返回值进行校验，这可以帮助 eBPF 验证器获取指针读写的边界
-   因为 eBPF 在老版本内核中并不支持循环（有界循环在 **5.3** 之后才支持），要访问字符串数组，还需要一个小技巧：使用 `#pragma unroll` 告诉编译器，把源码中的循环自动展开。这就避免了最终的字节码中包含循环

eBPF 程序开发完成后，最后一步就是为它增加一个 Python 前端。**Python 前端逻辑需要 eBPF 程序加载、挂载到内核函数和跟踪点，以及通过 BPF 映射获取和打印执行结果等几个步骤**。c 中已经使用了 `TRACEPOINT_PROBE` 宏定义，来定义 eBPF 跟踪点处理函数，BCC 在加载字节码的时候，会自动把它挂载到正确的跟踪点上，所以挂载的步骤就可以忽略

```python
#!/usr/bin/env python3
# 引入库函数
from bcc import BPF
from bcc.utils import printb

# 1) 加载eBPF代码
b = BPF(src_file="execsnoop.c")

# 2) 输出头
print("%-6s %-16s %-3s %s" % ("PID", "COMM", "RET", "ARGS"))

# 3) 定义性能事件打印函数
def print_event(cpu, data, size):
    # BCC自动根据"struct data_t"生成数据结构
    event = b["events"].event(data)
    printb(b"%-6d %-16s %-3d %-16s" % (event.pid, event.comm, event.retval, event.argv))

# 4) 绑定性能事件映射和输出函数，并从映射中循环读取数据
b["events"].open_perf_buffer(print_event)
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```

**测试：**

```bash
$ sudo python3 execsnoop.py
PID    COMM             RET ARGS
249134 zsh              0   ls--color=tty
```



#### libbpf 方法：

使用 libbpf 开发 eBPF 程序也是分为两部分：

-   第一，内核态的 eBPF 程序；
-   第二，用户态的加载、挂载、映射读取以及输出程序等

**在 eBPF 程序中，由于内核已经支持了 BTF，不再需要引入众多的内核头文件来获取内核数据结构的定义。** 取而代之的是一个通过 bpftool 生成的 **`vmlinux.h`** 头文件，其中包含了内核数据结构的定义。

这样，使用 libbpf 开发 eBPF 程序就可以通过以下四个步骤完成：

1.  使用 bpftool 生成内核数据结构定义头文件。BTF 开启后，可以在系统中找到 `/sys/kernel/btf/vmlinux` 这个文件，bpftool 正是从它生成了内核数据结构头文件。
2.  开发 eBPF 程序部分。为了方便后续通过统一的 Makefile 编译，eBPF 程序的源码文件一般命名为 `<程序名>.bpf.c`。
3.  编译 eBPF 程序为字节码，然后再调用 `bpftool gen skeleton` 为 eBPF 字节码生成脚手架头文件（Skeleton Header）。这个头文件包含了 eBPF 字节码以及相关的加载、挂载和卸载函数，可在用户态程序中直接调用。
4.  最后就是用户态程序引入上一步生成的头文件，开发用户态程序，包括 eBPF 程序加载、挂载到内核函数和跟踪点，以及通过 BPF 映射获取和打印执行结果等。

通常，这几个步骤里面的编译、库链接、执行 `bpftool` 命令等，都可以放到 Makefile 中，这样就可以通过一个 `make` 命令去执行所有的步骤。比如，下面是一个简化版本的 Makefile：

```makefile
# 生成得项目名(可以是多个)
APPS = target 

# 声明工具位置
bpftool = $(shell which bpftool || ../tools/bpftool)

# 引用 https://github.com/libbpf/libbpf 第三方库
LIBBPF_SRC := $(abspath ./libbpf/src)
LIBBPF_OBJ := $(abspath libbpf/libbpf.a)

# 声明依赖库位置
INCLUDES := -Ilibbpf/usr/include -I../libbpf/include/uapi -I/usr/include/x86_64-linux-gnu -I.

# 编译目标
.PHONY: all
all: $(APPS)

# 由于寄存器是特定的架构，结构的定义取决于所运行的架构。这意味着，如果想使用这些宏，需要告诉编译器目标架构是什么
# 可以通过设置-D __TARGET_ARCH_($ARCH)来做到这一点，其中$ARCH是像arm64、amd64等这样的体系结构名称；
# 如果不使用宏，则需要特定于体系结构的代码来访问寄存器信息
ARCH = $(shell uname -m | sed 's/x86_64/x86/' | sed 's/aarch64/arm64/')
%.bpf.o: %.bpf.c vmlinux.h
	clang \
	    -target bpf \
        -D __TARGET_ARCH_$(ARCH) \
	    -Wall \
	    -O2 -g -o $@ -c $<
	llvm-strip -g $@


# clang 的参数 -target bpf 表示要生成 BPF 字节码，
# -D__TARGET_ARCH_x86_64 表示目标的体系结构是 x86_64
# -I 则是引入头文件路径
# bpftool gen skeleton 为 eBPF 字节码生成脚手架头文件（Skeleton Header）*.skel.h。这个头文件包含了 eBPF 字节码以及相关的加载、挂载和卸载函数，可在用户态程序中直接调用
$(APPS): %: %.bpf.c $(LIBBPF_OBJ) $(wildcard %.h)
	clang -g -O2 -target bpf -D__TARGET_ARCH_x86 $(INCLUDES) -c $@.bpf.c -o $@.bpf.o
	$(bpftool) gen skeleton $@.bpf.o > $@.skel.h
	clang -g -O2 -Wall $(INCLUDES) -c $@.c -o $@.o
	clang -Wall -O2 -g $@.o -static $(LIBBPF_OBJ) -lelf -lz -o $@

libbpf: $(LIBBPF_OBJ)

$(LIBBPF_OBJ): $(wildcard $(LIBBPF_SRC)/*.[ch] $(LIBBPF_SRC)/Makefile)
	make -C $(LIBBPF_SRC) BUILD_STATIC_ONLY=1 OBJDIR=$(dir $@) DESTDIR=$(dir $@) install

# 生成内核数据结构的头文件
vmlinux:
	$(bpftool) btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h

clean:
	rm -rf $(APPS) *.o
```

执行 `make vmlinux` 命令就可以生成 `vmlinux.h` 文件，再执行 `make` 就可以编译 `APPS` 里面配置的所有 eBPF 程序（多个程序之间以空格分隔）



1. **生成内核数据结构的头文件：**

```bash
sudo bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

如果命令执行失败了，并且错误说 BTF 不存在，那说明当前系统内核没有开启 BTF 特性。这时候，需要开启 `CONFIG_DEBUG_INFO_BTF=y` 和 `CONFIG_DEBUG_INFO=y` 这两个编译选项，然后重新编译和安装内核。

2. **eBPF 程序定义:**

开发 eBPF 程序，包括定义哈希映射、性能事件映射以及跟踪点的处理函数等，而对这些数据结构和跟踪函数的定义都可以通过 `SEC()` 宏定义来完成。在编译时，**通过 `SEC()` 宏定义的数据结构和函数会放到特定的 ELF 段中，这样后续在加载 BPF 字节码时，就可以从这些段中获取所需的元数据。**

```c
// 包含头文件
#include "vmlinux.h"			// 包含了内核数据结构
#include <bpf/bpf_helpers.h>	// 包含 BPF 辅助函数

// 定义进程基本信息数据结构
struct event {
    char comm[TASK_COMM_LEN];
    pid_t pid;
    int retval;
    int args_count;
    unsigned int args_size;
    char args[FULL_MAX_ARGS_ARR];
};

// 定义哈希映射
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __uint(max_entries, 10240);
    __type(key, pid_t);
    __type(value, struct event);
} execs SEC(".maps");

// 定义性能事件映射
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} events SEC(".maps");

// 入口跟踪点处理 (sys_enter_execve跟踪点)
SEC("tracepoint/syscalls/sys_enter_execve")
int tracepoint__syscalls__sys_enter_execve(struct trace_event_raw_sys_enter
                       *ctx)
{
    struct event *event;
    const char **args = (const char **)(ctx->args[1]);
    const char *argp;

    // 查询PID
    u64 id = bpf_get_current_pid_tgid();
    pid_t pid = (pid_t) id;

    // 保存一个空的event到哈希映射中
    if (bpf_map_update_elem(&execs, &pid, &empty_event, BPF_NOEXIST)) {
        return 0;
    }
    event = bpf_map_lookup_elem(&execs, &pid);
    if (!event) {
        return 0;
    }

    // 初始化event变量
    event->pid = pid;
    event->args_count = 0;
    event->args_size = 0;

    // 查询第一个参数
    unsigned int ret = bpf_probe_read_user_str(event->args, ARGSIZE,
                           (const char *)ctx->args[0]);
    if (ret <= ARGSIZE) {
        event->args_size += ret;
    }

    // 查询其他参数
    event->args_count++;
    #pragma unroll
    for (int i = 1; i < TOTAL_MAX_ARGS; i++) {
        bpf_probe_read_user(&argp, sizeof(argp), &args[i]);
        if (!argp)
            return 0;

        if (event->args_size > LAST_ARG)
            return 0;

        ret =
            bpf_probe_read_user_str(&event->args[event->args_size],
                        ARGSIZE, argp);
        if (ret > ARGSIZE)
            return 0;

        event->args_count++;
        event->args_size += ret;
    }

    // 再尝试一次，确认是否还有未读取的参数
    bpf_probe_read_user(&argp, sizeof(argp), &args[TOTAL_MAX_ARGS]);
    if (!argp)
        return 0;

    // 如果还有未读取参数，则增加参数数量（用于输出"..."）
    event->args_count++;

    return 0;
}

// 出口跟踪点处理(sys_exit_execve跟踪点)
SEC("tracepoint/syscalls/sys_exit_execve")
int tracepoint__syscalls__sys_exit_execve(struct trace_event_raw_sys_exit *ctx)
{
    u64 id;
    pid_t pid;
    int ret;
    struct event *event;

    // 从哈希映射中查询进程基本信息
    id = bpf_get_current_pid_tgid();
    pid = (pid_t) id;
    event = bpf_map_lookup_elem(&execs, &pid);
    if (!event)
        return 0;

    // 更新返回值和进程名称
    ret = ctx->ret;
    event->retval = ret;
    bpf_get_current_comm(&event->comm, sizeof(event->comm));

    // 提交性能事件
    size_t len = EVENT_SIZE(event);
    if (len <= sizeof(*event))
        bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, event,
                      len);

    // 清理哈希映射
    bpf_map_delete_elem(&execs, &pid);
    return 0;
}

// 定义许可证（前述的BCC默认使用GPL）
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```

-   头文件 `vmlinux.h` 包含了内核数据结构，而 `bpf/bpf_helpers.h` 包含 BPF 辅助函数；
-   `struct event` 定义了进程基本信息数据结构，它会用在后面的哈希映射中；
-   `SEC(".maps")` 定义了哈希映射和性能事件映射；
-   `SEC("tracepoint/<跟踪点名称>")` 定义了跟踪点处理函数，系统调用跟踪点的格式是 `tracepoint/syscalls/<系统调用名称>"`。以后需要定义内核插桩和用户插桩的时候，也是以类似的格式定义，比如 `kprobe/do_unlinkat` 或 `uprobe/func`；
-   最后的 `SEC("license")` 定义了 eBPF 程序的许可证。在上述的 BCC eBPF 程序中，并没有定义许可证，这是因为 BCC 自动使用了 GPL 许可。


##### SEC 宏说明：
SEC 是 eBPF 程序的关键字，用于为 eBPF 程序中的函数或变量设置特定功能或属性

1. `SEC("license")`:  定义了 eBPF 程序的许可证, 许可证类型必须与内核符号匹配以允许程序附加到内核。对于GPL许可证来说，任何基于BSD或专有许可证的代码都不允许将其代码添加到GPL密集应用程序中，因此它限制了附加到eBPF的程序开源代码的类型和分发方式。
```bash
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```
2. `SEC("xdp")、SEC("tracepoint/<跟踪点名称>")`：通过 eBPF 的钩子机制将函数与内核中的特定挂载点关联起来，**指明这个函数的运行时机**，例如这里使用 `"xpd"` 声明了这个函数是在 xdp 挂载点上运行。其作用相当于定义一个特殊的注解，告诉eBPF程序运行控制流从哪里开始和结束。这些注解对于保证安全性和可移植性非常重要。
```c
int SEC("xdp") ip_handler(struct xdp_md *ctx, __be32 addr, __be16 port)

SEC("tracepoint/syscalls/sys_exit_execve")`
int tracepoint__syscalls__sys_exit_execve(struct trace_event_raw_sys_exit *ctx)

SEC("xdp_prog")
int xdp_filter(struct xdp_md *ctx) 
```
3. `SEC('map')`: 意味着该变量属于 "maps" 类型，这是一个特殊的标签，它告诉内核如何正确的映射该变量，在maps（即BPF_MAP_TYPE_HASH）程序加载时，内核使用map定义创建并确定map的大小、键/值的大小等重要参数
```c
struct bpf_map_def SEC("maps") portmap {}

struct {
} my_map SEC(".maps");
```

- `SEC("socket_filter")`: 表示这是一个 socket filter，必须以 BPF_PROG_TYPE_SOCKET_FILTER 类型加载。
- `SEC("kprobe")` 和 `SEC("kretprobe")`: 表示在内核中挂钩某个代码点，在函数被调用前后执行 eBPF 程序。
- `SEC("uprobe")` 和 `SEC("uretprobe")`: 表示在用户态进程中挂钩某个函数，在函数被调用前后执行 eBPF 程序。
- `SEC("tracepoint")`: 表示在系统跟踪点上执行 eBPF 程序，其中 tracepoint 的名称是作为 SEC 函数参数传递的。
- `SEC("perf_event")`: 表示一个性能分析事件触发器，用于对性能进行分析。
- `SEC("cgroup_skb/sock")`: 分别表示对包和套接字访问权限控制。



##### `SEC(".maps")` 和 `SEC("maps")` 的区别：
`SEC(".maps")` 和 `SEC("maps")` 都可以在代码中创建 BPF Maps，但是前者创建的Map存储于`.bss`段（静态内存区），后者则为Map分配内存于映射段。所以，应该区分它们的应用场景。具体来说：

- `SEC(".maps")`：主要用于将Map声明为**全局变量**。由于Map存储于`.bss`段，因此创建该类型的Map时只需要在代码中定义即可，系统会自动分配内存空间。例如：
```c
struct bpf_map_def SEC("maps") mymap = {
    .type = BPF_MAP_TYPE_HASH,
    .key_size = sizeof(int),
    .value_size = sizeof(int),
    .max_entries = 100,
};
```
在上面的示例中，将一个哈希表声明为mymap全局变量，并将它存储于 `.bss` 段。当程序执行时，系统会自动分配内存空间。

- `SEC("maps")`：主要用于在代码中直接为Map分配内存并初始化它。通常情况下，该类型的Map是程序运行期间的**临时变量**。例如：
```c
int key = 0;
struct bpf_map_def *map = bpf_map_lookup_elem(&mapptr, &key);
if (map) {
        SEC("maps")
        struct leafy_btree_node leaf = {};
        ...
}
```

在上述伪代码中，使用bpf_map_lookup_elem()函数查找内存中已有的Map。如果Map存在就需要对它新增一条记录。因此，使用SEC("maps")确保Map可以被正确的分配和使用。

**综上所述，SEC(".maps")主要用于全局变量的声明，而 SEC("maps") 更多的是用于编写具体逻辑时，为 Map 提供临时存储**。


##### 如何确定系统许可证类型：

可以使用uname -a命令获取当前Linux内核的版本信息，一般情况下在输出信息中会有许可证类型。例如：
```bash
# uname -a
Linux root 5.15.0-60-generic #66-Ubuntu SMP Fri Jan 20 14:29:49 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```
该系统使用的是GNU许可证（GPL）。如果看到类似 "FreeBSD"、"OpenBSD" 等字样，则表明该系统使用的是 BSD 许可证。



有了基本的程序结构，接下来就是**实现系统调用入口和出口跟踪点的处理函数**



**Libbpf 和 bcc 区别：**

Libbpf 它的处理逻辑跟上述的 BCC 程序基本上是相同的。不过，详细对比一下，会发现它们之间还是有不同的，不同点主要在两个方面：

-   第一，函数名的定义格式不同。BCC 程序使用的是 `TRACEPOINT_PROBE` 宏，而 libbpf 程序用的则是 `SEC` 宏。
-   第二，映射的访问方法不同。BCC 封装了很多更易用的映射访问函数（如 `tasks.lookup()`），而 libbpf 程序则需要调用 BPF 辅助函数（比如查询要使用 `bpf_map_lookup_elem()`）。

到这里，新建一个目录，并把上述代码存入 `execsnoop.bpf.c` 文件中，eBPF 部分的代码也就开发好了。



3. **编译并生成脚手架头文件**：

有了 eBPF 程序，执行下面的命令，就可以使用 clang 和 bpftool 将其编译成 BPF 字节码，然后再生成其脚手架头文件 `execsnoop.skel.h` （注意，脚手架头文件的名字一般定义为 `<程序名>.skel.h`）

```bash
clang -g -O2 -target bpf -D__TARGET_ARCH_x86_64 -I/usr/include/x86_64-linux-gnu -I. -c execsnoop.bpf.c -o execsnoop.bpf.o
bpftool gen skeleton execsnoop.bpf.o > execsnoop.skel.h
```

其中

-   clang 的参数 `-target bpf` 表示要生成 BPF 字节码，
-   `-D__TARGET_ARCH_x86_64` 表示目标的体系结构是 x86_64
-   `-I` 则是引入头文件路径。

命令执行后，脚手架头文件会放到 `execsnoop.skel.h` 中，这个头文件包含了 BPF 字节码和相关的管理函数。因而，当用户态程序引入这个头文件并编译之后，只需要分发最终用户态程序生成的二进制文件到生产环境即可（如果用户态程序使用了其他的动态库，还需要分发动态库）。

4. **开发用户态程序**：

有了脚手架头文件之后，还剩下最后一步，也就是用户态程序的开发。同 BCC 的 Python 前端程序类似，libbpf 用户态程序也需要 eBPF 程序加载、挂载到跟踪点，以及通过 BPF 映射获取和打印执行结果等几个步骤。虽然 C 语言听起来可能比 Python 语言麻烦一些，但实际上，这几个步骤都可以通过脚手架头文件中自动生成的函数来完成。

**用户态程序的一个基本框架：**

```c
// Based on execsnoop(8) from BCC by Brendan Gregg and others.
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <sys/resource.h>
#include <bpf/libbpf.h>
#include <time.h>
#include "execsnoop.h"
// 引入脚手架头文件
#include "execsnoop.skel.h"


// libbpf错误和调试信息回调的处理程序
static int libbpf_print_fn(enum libbpf_print_level level, const char *format,
			   va_list args)
{
#ifdef DEBUGBPF
	return vfprintf(stderr, format, args);
#else
	return 0;
#endif
}

// 丢失事件的处理程序
void handle_lost_events(void *ctx, int cpu, __u64 lost_cnt)
{
	fprintf(stderr, "Lost %llu events on CPU #%d!\n", lost_cnt, cpu);
}

// 用于打印参数的Heandler (参数通过 '\0' 分割).
static void print_args(const struct event *e)
{
	int args_counter = 0;

	for (int i = 0; i < e->args_size && args_counter < e->args_count; i++) {
		char c = e->args[i];
		if (c == '\0') {
			args_counter++;
			putchar(' ');
		} else {
			putchar(c);
		}
	}
	if (e->args_count > TOTAL_MAX_ARGS) {
		fputs(" ...", stdout);
	}
}

// 为每个性能事件处理程序
void handle_event(void *ctx, int cpu, void *data, __u32 data_sz)
{
	const struct event *e = data;
	printf("%-16s %-6d %3d ", e->comm, e->pid, e->retval);
	print_args(e);
	putchar('\n');
}

// 提升RLIMIT_MEMLOCK，允许BPF子系统做任何它需要的事情
static void bump_memlock_rlimit(void)
{
	struct rlimit rlim_new = {
		.rlim_cur = RLIM_INFINITY,
		.rlim_max = RLIM_INFINITY,
	};

	if (setrlimit(RLIMIT_MEMLOCK, &rlim_new)) {
		fprintf(stderr, "Failed to increase RLIMIT_MEMLOCK limit!\n");
		exit(1);
	}
}

// C语言主函数
int main(int argc, char **argv)
{
    // 定义BPF程序和性能事件缓冲区
    struct execsnoop_bpf *skel;
    struct perf_buffer_opts pb_opts;
    struct perf_buffer *pb = NULL;
    int err;

    // 1. 设置调试输出函数
    libbpf_set_print(libbpf_print_fn);

    // 2. 增大 RLIMIT_MEMLOCK（默认值通常太小，不足以存入BPF映射的内容）
    bump_memlock_rlimit();

    // 3. 初始化BPF程序
    skel = execsnoop_bpf__open();
	if (!skel) {
		fprintf(stderr, "Failed to open BPF skeleton\n");
		return 1;
	}
    
    // 4. 加载BPF字节码
    err = execsnoop_bpf__load(skel);
	if (err) {
		fprintf(stderr, "Failed to load and verify BPF skeleton\n");
		goto cleanup;
	}
    
    // 5. 挂载BPF字节码到跟踪点
    err = execsnoop_bpf__attach(skel);
	if (err) {
		fprintf(stderr, "Failed to attach BPF skeleton\n");
		goto cleanup;
	}
    
    // 6. 配置性能事件回调函数
    pb_opts.sample_cb = handle_event;
	err = libbpf_get_error(pb);
	if (err) {
		pb = NULL;
		fprintf(stderr, "failed to open perf buffer: %d\n", err);
		goto cleanup;
	}

	printf("%-16s %-6s %3s %s\n", "COMM", "PID", "RET", "ARGS");err = libbpf_get_error(pb);
	if (err) {
		pb = NULL;
		fprintf(stderr, "failed to open perf buffer: %d\n", err);
		goto cleanup;
	}

	printf("%-16s %-6s %3s %s\n", "COMM", "PID", "RET", "ARGS");
    
    pb = perf_buffer__new(bpf_map__fd(skel->maps.events), 64, &pb_opts);

    // 7. 从缓冲区中循环读取数据
    while ((err = perf_buffer__poll(pb, 100)) >= 0);
	printf("Error polling perf buffer: %d\n", err);

 cleanup:
	perf_buffer__free(pb);
	execsnoop_bpf__destroy(skel);
	return err != 0;
}
```

编译为可执行文件：

```bash
clang -g -O2 -Wall -I . -c execsnoop.c -o execsnoop.o
clang -Wall -O2 -g execsnoop.o -static -lbpf -lelf -lz -o execsnoop
```

最后，执行 `execsnoop`，就可以得到如下的结果：

```bash
$ sudo ./execsnoop
COMM             PID    RET ARGS
sh               276871   0 /bin/sh -c which ps
which            276872   0 /usr/bin/which ps
```

可以**直接把这个文件复制到开启了 BTF 的其他机器中，无需安装额外的 LLVM 开发工具和内核头文件，也可以直接执行**。

如果命令失败，并且你看到如下的错误，这说明当前机器没有开启 BTF，需要重新编译内核开启 BTF 才可以运行：

```bash
Failed to load and verify BPF skeleton
```

>   [代码参考](https://github.com/feiskyer/ebpf-apps/blob/main/bpf-apps/execsnoop.c)



在不支持 BTF 的机器中，如果不想在运行 eBPF 时依赖于 LLVM 编译和内核头文件，还可以参考内核中的 [BPF 示例](https://elixir.bootlin.com/linux/v5.13/source/samples/bpf)，直接引用内核源码中的 `tools/lib/bpf/` 库，以及内核头文件中的数据结构，来开发 eBPF 程序。



### 案例2: 使用eBPF排查应用程序:

在跟踪内核的状态之前，需要利用内核提供的调试信息查询内核函数、内核跟踪点以及性能事件等。类似地，在跟踪应用进程之前，也需要知道**这个进程所对应的二进制文件中提供了哪些可用的跟踪点**，那就是**应用程序二进制文件中的调试信息**。

在静态语言的编译过程中，通常你可以加上 `-g` 选项保留调试信息，源代码中的函数、变量以及它们对应的代码行号等信息，就以 [DWARF](https://dwarfstd.org/)（Debugging With Attributed Record Formats，Linux 和类 Unix 平台最主流的调试信息格式）格式存储到了编译后的二进制文件中. 有了调试信息，就可以通过 [readelf](https://man7.org/linux/man-pages/man1/readelf.1.html)、[objdump](https://man7.org/linux/man-pages/man1/objdump.1.html)、[nm](https://man7.org/linux/man-pages/man1/nm.1.html) 等工具，查询可用于跟踪的函数、变量等符号列表。比如，使用 `readelf` 命令，查询二进制文件的基本信息。在终端中执行下面的命令，就可以查询 [libc](https://www.gnu.org/software/libc/) 动态链接库中的符号表

```bash
# 查询符号表（RHEL8系统中请把动态库路径替换为/usr/lib64/libc.so.6）
readelf -Ws /usr/lib/x86_64-linux-gnu/libc.so.6

# 查询USDT信息（USDT信息位于ELF文件的notes段）
readelf -n /usr/lib/x86_64-linux-gnu/libc.so.6
```

bpftrace 工具也可以用来查询 uprobe 和 USDT 跟踪点，其查询格式如下所示（同样支持 `*` 通配符过滤）：

```bash
# 查询uprobe（RHEL8系统中请把动态库路径替换为/usr/lib64/libc.so.6）
bpftrace -l 'uprobe:/usr/lib/x86_64-linux-gnu/libc.so.6:*'

# 查询USDT
bpftrace -l 'usdt:/usr/lib/x86_64-linux-gnu/libc.so.6:*'
```

同内核跟踪点类似，也可以加上 `-v` 选项**查询用户探针的参数格式**。不过**想要通过二进制文件查询符号表和参数定义，必须在编译的时候保留 DWARF 调试信息**。

除了符号表之外，理论上你可以把 uprobe 插桩到二进制文件的任意地址。不过这要求你对应用程序 ELF 格式的地址空间非常熟悉，并且具体的地址会随着应用的迭代更新而发生变化。所以，在需要跟踪地址的场景中，一定要记得去 ELF 二进制文件动态获取地址信息。

另外**uprobe 是基于文件的。当文件中的某个函数被跟踪时，除非对进程 PID 进行了过滤，默认所有使用到这个文件的进程都会被插桩。**



编程语言进行归类，按照其运行原理，大致上可以分为三类：

-   第一类是 C、C++、Golang 等编译为机器码后再执行的**编译型语言**。这类编程语言开发的程序，通常会编译成 ELF 格式的二进制文件，包含了保存在寄存器或栈中的函数参数和返回值，因而可以直接通过二进制文件中的符号进行跟踪。
-   第二类是 Python、Bash、Ruby 等通过解释器语法分析之后再执行的**解释型语言**。这类编程语言开发的程序，无法直接从语言运行时的二进制文件中获取应用程序的调试信息，通常需要跟踪解释器的函数，再从其参数中获取应用程序的运行细节。
-   最后一类是 Java、.Net、JavaScript 等先编译为字节码，再由即时编译器（JIT）编译为机器码执行的**即时编译型语言**。同解释型语言类似，这类编程语言无法直接从语言运行时的二进制文件中获取应用程序的调试信息。跟踪 JIT 编程语言开发的程序是最困难的，因为 JIT 编译的状态只存在于内存中。

大部分编译型语言遵循 [ABI（Application Binary Interface）](https://en.wikipedia.org/wiki/Application_binary_interface) 调用规范，函数的参数和返回值都存放在寄存器中。而 Go 1.17 之前使用的是 [Plan 9](https://9p.io/sys/doc/asm.html) 调用规范，函数的参数和返回值都存放在堆栈中；直到 1.17， Go 才从 Plan 9 切换到 ABI 调用规范。所以，在跟踪函数参数和返回值时，你需要**首先区分编程语言的调用规范**，然后再去寄存器或堆栈中读取函数的参数和返回值。



#### **编译型语言：**

​		**调试信息并非一定要内置于最终分发的应用程序二进制文件中，它们也可以放到独立的调试文件存储**。为了减少应用程序二进制文件的大小，通常会把调试信息从二进制文件中剥离出来，保存到 `<应用名>.debuginfo` 或者 `<build-id>.debug` 文件中，后续排查问题需要用到时再安装。每次编译和发布应用程序之前，通常都需要执行一下 `strip` 命令，把调试信息删除后才发布到生产环境中。



示例：**使用的 Bash 为例（Bash 是一个典型的 C 语言程序），看看如何跟踪编译型语言应用程序的执行状态**

在跟踪 Bash 之前，首先执行下面的命令，安装它的调试信息：

```bash
# Ubuntu
sudo apt install bash-dbgsym

# RHEL
sudo debuginfo-install bash
```

有了 Bash 调试信息之后，再执行下面的几步，查询 Bash 的符号表：

```bash
# 第一步，查询 Build ID（用于关联调试信息）
readelf -n /usr/bin/bash | grep 'Build ID'
# 输出示例为：
#     Build ID: 7b140b33fd79d0861f831bae38a0cdfdf639d8bc

# 第二步，找到调试信息对应的文件（调试信息位于目录/usr/lib/debug/.build-id中，上一步中得到的Build ID前两个字母为目录名）
ls /usr/lib/debug/.build-id/7b/140b33fd79d0861f831bae38a0cdfdf639d8bc.debug

# 第三步，从调试信息中查询符号表
readelf -Ws /usr/lib/debug/.build-id/7b/140b33fd79d0861f831bae38a0cdfdf639d8bc.debug
```

参考 Bash 的[源代码](https://git.savannah.gnu.org/cgit/bash.git/tree/lib/readline/readline.c?h=bash-5.1#n352)，每条 Bash 命令在运行前，都会调用 `char* readline (const char *prompt)` 函数读取用户的输入，然后再去解析执行（Bash 自身是使用编译型语言 C 开发的，而 Bash 语言则是一种解释型语言）。

注意，`readline` 函数的参数是命令行提示符（通过环境变量 `PS1`、`PS2` 等设置），而返回值才是用户的输入。因而，只需要跟踪 `readline` 函数的返回值，也就是使用 `uretprobe` 跟踪。

1.   **使用bpftrace 实现:** 

```bash
sudo bpftrace -e 'uretprobe:/usr/bin/bash:readline { printf("User %d executed \"%s\" command\n", uid, str(retval)); }'
```

这个命令中具体内容的作用如下：

-   `uretprobe:/usr/bin/bash:readline` 设置跟踪类型为 `uretprobe`，跟踪的二进制文件为 `/usr/bin/bash`，跟踪符号为 `readline`；
-   中括号里的内容为 uretprobe 的处理函数；
-   处理函数中，`uid` 和 `retval` 是两个内置变量，分别表示用户 UID 以及返回值；
-   `str` 用于从指针中读取字符串， `str(retval)` 就是 Bash 中输入命令的字符串；
-   `printf` 用于向终端中打印一个字符串。

打开一个终端，并在新终端中执行 `ps` 命令，然后就会在第一个终端中看到如下的输出（即 UID 为 1000 的用户执行了 `ps` 命令）：

```plain
Attaching 1 probe...
User 1000 executed "ps" command
```



2.   **使用 bcc 实现:**

BCC 的使用可以分为两部分：

-   第一部分是用 C 语言开发的 eBPF 程序。在 eBPF 程序中，可以利用 BCC 提供的[库函数和宏定义](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md)简化你的处理逻辑。
-   第二部分是用 Python 语言开发的前端界面，其中包含 eBPF 程序加载、挂载到内核函数和跟踪点，以及通过 BPF 映射获取和打印执行结果等部分。在前端程序中，同样可以利用 BCC 库来访问 BPF 映射。

对于第一部分来说，我们要做的就是从 uretprobe 的处理函数中获取 `readline` 的返回值，然后提交到性能事件映射中。对于 BCC 的 uretprobe 来说，其官方文档链接就是 [uretprobes](https://github.com/iovisor/bcc/blob/master/docs/reference_guide.md#5-uretprobes)。根据这里的文档，uretprobe 处理函数的定义格式应该为 `function_name(struct pt_regs *ctx)`，而返回值可以通过宏 `PT_REGS_RC(ctx)` 来获取（实际上，kretprobe 也是采用相同的格式）。

```c
// 包含头文件
#include <uapi/linux/ptrace.h>

// 定义数据结构和性能事件映射
struct data_t {
    u32 uid;
    char command[64];
};
BPF_PERF_OUTPUT(events);

// 定义uretprobe处理函数
int bash_readline(struct pt_regs *ctx)
{
    // 查询uid
    struct data_t data = { };
    data.uid = bpf_get_current_uid_gid();

    // 从PT_REGS_RC(ctx)读取返回值
    bpf_probe_read_user(&data.command, sizeof(data.command), (void *)PT_REGS_RC(ctx));

    // 提交性能事件
    events.perf_submit(ctx, &data, sizeof(data));
    return 0;
}
```

有了 eBPF 程序之后，第二部分的 Python 前端也比较直观，代码如下所示：

```python
# 引入BCC库
from bcc import BPF
from time import strftime

# 加载eBPF 程序
b = BPF(src_file="bashreadline.c")

# 挂载uretprobe
b.attach_uretprobe(name="/usr/bin/bash", sym="readline", fn_name="bash_readline")

# 定义性能事件回调（输出时间、UID以及Bash中执行的命令）
def print_event(cpu, data, size):
    event = b["events"].event(data)
    print("%-9s %-6d %s" % (strftime("%H:%M:%S"), event.uid, event.command.decode("utf-8")))

# 打印头
print("%-9s %-6s %s" % ("TIME", "UID", "COMMAND"))

# 绑定性能事件映射和输出函数，并从映射中循环读取数据
b["events"].open_perf_buffer(print_event)
while 1:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```

挂载 uretprobe 的时候调用了 `b.attach_uretprobe()` ，并在其参数中传入了二进制文件的路径 `name="/usr/bin/bash"` ，以及要跟踪的符号 `sym="readline"`。

在 BCC 的内部，当挂载 uprobe 或者 uretprobe 到用户程序时，由于同一个符号可能出现多次，BCC 会首先查询该符号的所有地址，然后把 eBPF 程序挂载到所有不同的地址上。

执行 `sudo python3 bashreadline.py` ，就可以运行这个跟踪程序。打开一个新终端并运行 `ls` 命令，回到第一个终端，就可以看到如下的输出：

```plain
TIME      UID    COMMAND
07:17:07  1000   ls
```

从这里的例子中，可以发现：编译型语言应用程序的跟踪与内核的跟踪是类似的，只不过是把跟踪类型从 kprobe 换成了 uprobe 或者 USDT。不同的地方在于符号信息：应用程序的符号信息可以存放在 ELF 二进制文件中，也可以以单独文件的形式，放到调试文件中；而内核的符号信息除了可以存放到内核二进制文件中之外，还会以 `/proc/kallsyms` 和 `/sys/kernel/debug` 等形式暴露到用户空间。



#### 跟踪解释型语言

根据 Python [文档](https://docs.python.org/zh-cn/3/howto/instrumentation.html)，为其开启 USDT 跟踪点（编译选项为 `--with-dtrace`）之后，Python3 二进制文件中就会包含一系列的 USDT 跟踪点。这些跟踪点也可以通过 bpftrace 查询到：

```bash
$ bpftrace -l '*:/usr/bin/python3:*'
usdt:/usr/bin/python3:python:audit
usdt:/usr/bin/python3:python:function__entry
usdt:/usr/bin/python3:python:function__return
usdt:/usr/bin/python3:python:gc__done
usdt:/usr/bin/python3:python:gc__start
usdt:/usr/bin/python3:python:import__find__load__done
usdt:/usr/bin/python3:python:import__find__load__start
usdt:/usr/bin/python3:python:line
```

其中，跟函数调用相关的正是 `function__entry` 和 `function__return`，因而它们就可以用来跟踪函数的调用过程。根据 Python [文档](https://docs.python.org/zh-cn/3/howto/instrumentation.html#available-static-markers)，这两个函数的定义格式为：

```python
// 三个参数分别是文件名、函数名和行号
function__entry(str filename, str funcname, int lineno)
function__return(str filename, str funcname, int lineno)
```

比如，对 `function__entry` 来说，执行下面的 bpftrace 单行命令，就可以跟踪 Python 函数的调用信息：

```bash
sudo bpftrace -e 'usdt:/usr/bin/python3:function__entry { printf("%s:%d %s\n", str(arg0), arg2, str(arg1))}'
```

在这个命令中， `arg0`、 `arg1` 、 `arg2` 表示函数的三个参数。这条命令的含义就是把这几个参数拼接成 `文件名:行号 函数名` 的格式，然后再打印到终端上。

打开一个新终端，并执行下面的 Python 命令开启一个 http 服务：

```bash
python3 -m http.server 8080
```

然后，再回到第一个终端，就可以看到如下的输出：

```bash
...
/usr/lib/python3.9/socketserver.py:254 service_actions
/usr/lib/python3.9/selectors.py:403 select
/usr/lib/python3.9/socketserver.py:254 service_actions
/usr/lib/python3.9/selectors.py:403 select
```



<img src="https://static001.geekbang.org/resource/image/80/50/80c2a8fe3b36ee2fb033b4332431f750.jpg?wh=2284x2197" alt="img" style="zoom:25%;" />

**eBPF 提供了大量专用于网络的 eBPF 程序类型，包括 XDP 程序、TC 程序、套接字程序以及 cgroup 程序等**。这些类型的程序涵盖了从网卡（如卸载到硬件网卡中的 XDP 程序）到网卡队列（如 TC 程序）、封装路由（如轻量级隧道程序）、TCP 拥塞控制、套接字（如 sockops 程序）等内核协议栈，再到同属于一个 cgroup 的一组进程的网络过滤和控制，而这些都是内核协议栈的核心组成部分（如上图中绿色部分所示）



## 如何跟踪定位协议栈

**案例1:** 访问网站进行协议跟踪

### 如何跟踪内核网络协议栈

​		**难点**：因为**不清楚内核中都有哪些函数和跟踪点可以拿来跟踪**。而即使通过源码查询到了一系列的内核函数，还是没有一个清晰的思路把这些内核函数与所碰到的网络问题关联起来。

​		**解决方法: 通过跟踪调用栈，根据调用栈回溯路径，找出导致某个网络事件发生的整个流程，进而就可以再根据这些流程中的内核函数进一步跟踪**。既然是调用栈的**回溯**，只有知道了最接近整个执行逻辑结尾的函数，才有可能开始这个回溯过程。对 Linux 网络丢包问题来说，内核协议栈执行的结尾，当然就是释放最核心的 SKB（Socket Buffer）数据结构。查询内核 [SKB 文档](https://www.kernel.org/doc/htmldocs/networking/ch01s02.html)，可以发现，内核中释放 SKB 相关的函数有两个：

-   第一个，[kfree_skb](https://www.kernel.org/doc/htmldocs/networking/API-kfree-skb.html) ，它经常在网络异常丢包时调用；
-   第二个，[consume_skb](https://www.kernel.org/doc/htmldocs/networking/API-consume-skb.html) ，它在正常网络连接完成时调用。

这两个函数除了使用场景的不同，其功能和实现流程都是一样的，即都是检查 SKB 的引用计数，当引用计数为 0 时释放其内核内存。所以，要跟踪网络丢包的执行过程，也就可以跟踪 `kfree_skb` 的内核调用栈。

为了方便调用栈的跟踪，**bpftrace 提供了 [kstack](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#7-kstack-stack-traces-kernel) 和 [ustack](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md#8-ustack-stack-traces-user) 这两个内置变量，分别用于获取内核和进程的调用栈**。打开一个终端，执行下面的命令就可以跟踪 `kfree_skb` 的内核调用栈了：

```plain
# sudo bpftrace -l '*kfree_skb*'
kprobe:__dev_kfree_skb_any
kprobe:__dev_kfree_skb_irq
kprobe:__kfree_skb
kprobe:__kfree_skb_defer
kprobe:__traceiter_kfree_skb
kprobe:kfree_skb_list
kprobe:kfree_skb_partial
kprobe:kfree_skb_reason
kprobe:kfree_skbmem
kprobe:net_dm_packet_trace_kfree_skb_hit
kprobe:rtnl_kfree_skbs
kprobe:trace_kfree_skb_hit
tracepoint:skb:kfree_skb

# sudo bpftrace -lv kprobe:__kfree_skb
kprobe:__kfree_skb

# sudo bpftrace -e 'kprobe:__kfree_skb /comm=="curl"/ {printf("kstack: %s\n", kstack);}'
```

这个命令中的具体内容含义如下：

-   `kprobe:kfree_skb` 指定跟踪的内核函数为 `kfree_skb`；
-   紧随其后的 `/comm=="curl"/` ，表示只跟踪 `curl` 进程，这是为了过滤掉其他不相关的进程操作；
-   最后的 `printf()` 函数就是把 **内核协议栈(kstack)** 打印到终端中。

打开一个新终端，并在终端中执行 `curl time.geekbang.org` 命令，然后回到第一个终端，就可以看到如下的输出：

```bash
Attaching 1 probe...
kstack: 
        __kfree_skb+1
        tcp_ack+1073
        tcp_rcv_established+854
        tcp_v4_do_rcv+343
        tcp_v4_rcv+3630
        ip_protocol_deliver_rcu+60
        ip_local_deliver_finish+72
        ip_local_deliver+251
        ip_sublist_rcv_finish+107
        ip_sublist_rcv+380
        ip_list_rcv+249
        __netif_receive_skb_list_core+536
        netif_receive_skb_list_internal+398
        napi_complete_done+122
        e1000_clean+122
        __napi_poll+51
        net_rx_action+294
        __do_softirq+217
        irq_exit_rcu+148
        common_interrupt+74
        asm_common_interrupt+39

kstack: 
        __kfree_skb+1
        tcp_v4_do_rcv+343
        tcp_v4_rcv+3630
        ip_protocol_deliver_rcu+60
        ip_local_deliver_finish+72
        ip_local_deliver+251
        ip_sublist_rcv_finish+107
        ip_sublist_rcv+380
        ip_list_rcv+249
        __netif_receive_skb_list_core+536
        netif_receive_skb_list_internal+398
        napi_complete_done+122
        e1000_clean+122
        __napi_poll+51
        net_rx_action+294
        __do_softirq+217
        irq_exit_rcu+148
        common_interrupt+74
        asm_common_interrupt+39

kstack: 
        __kfree_skb+1
        tcp_recvmsg+121
        inet_recvmsg+92
        sock_recvmsg+113
        __sys_recvfrom+229
        __x64_sys_recvfrom+36
        do_syscall_64+92
        entry_SYSCALL_64_after_hwframe+97
```

这个输出包含了多个调用栈，每个调用栈从下往上就是 `kfree_skb` 被调用过程中的各个函数（**函数名后的数字表示调用点相对函数地址的偏移**），它们都是从系统调用（`entry_SYSCALL_64`）开始，通过一系列的内核函数之后，最终调用到了跟踪函数。

输出中包含多个调用栈，是因为同一个内核函数是有可能在多个地方调用的。因此，需要对它进一步改进，加上网络信息的过滤，并把源 IP 和目的 IP 等基本信息也打印出来。比如，访问一个网址，只需要关心 TCP 协议，而其他协议相关的内核栈就可以忽略掉。

`kfree_skb` 函数的定义格式如下所示，它包含一个 `struct sk_buff` 类型的参数，这样可以从中获取协议、源 IP 和目的 IP 等基本信息：

```c++
void kfree_skb(struct sk_buff * skb);
```

由于需要添加数据结构读取的过程，为了更好的可读性，可以把这些过程放入一个脚本文件中，通常后缀为 `.bt`。下面就是一个改进了的跟踪程序：

```c++
kprobe:__kfree_skb /comm=="curl"/
{
  // 1. 第一个参数是 struct sk_buff
  $skb = (struct sk_buff *)arg0;

  // 2. 从网络头中获取源IP和目的IP
  $iph = (struct iphdr *)($skb->head + $skb->network_header);
  $sip = ntop(AF_INET, $iph->saddr);
  $dip = ntop(AF_INET, $iph->daddr);

  // 3. 只处理TCP协议
  if ($iph->protocol == IPPROTO_TCP)
  {
    // 4. 打印源IP、目的IP和内核调用栈
    printf("TCP proto, SKB dropped: %s->%s, kstack: %s\n", $sip, $dip, kstack);
  } else {
    printf("%d proto, SKB dropped: %s->%s, kstack: %s\n", $iph->protocol, $sip, $dip, kstack);
  }
}
```

-   第1处是把 bpftrace 的内置参数 `arg0` 转换成 SKB 数据结构 `struct sk_buff *`（注意使用指针）。
-   第2处是从 SKB 数据结构中获取网络头之后，再从中拿到源IP和目的IP，最后再调用内置函数 `ntop()` ，把整数型的 IP 数据结构转换为可读性更好的字符串格式。
-   第3处是对网络协议进行了过滤，只保留TCP协议。
-   第4处是向终端中打印刚才获取的源IP和目的IP，同时也打印内核调用栈。

直接把这个脚本保存到文件中，bpftrace 并不能直接运行, 因为，**bpftrace 在将上述脚本编译为 BPF 字节码的过程中，找不到这些类型的定义**。在内核支持 BTF 之前，这其实是所有 eBPF 程序开发过程中都会遇到的一个问题。要解决这个问题，需要把所需的内核头文件引入进来。

**如何找出这些数据结构的头文件呢**？通常使用下面的两种方法：

-   第一种方法是在内核源码目录中，通过查找的方式，找出定义了这些数据结构的头文件（后缀为 `.h`）。
-   另外一种方法是到 https://elixir.bootlin.com/ 上选择正确的内核版本后，再搜索数据结构的名字。

在脚本文件中加入这些类型定义的头文件：

```c++
#include <linux/skbuff.h>
#include <linux/ip.h>
#include <linux/socket.h>
#include <linux/netdevice.h>
```

然后，保存到文件 `dropwatch.bt`中（也可以在 [GitHub](https://github.com/feiskyer/ebpf-apps/blob/main/bpftrace/dropwatch.bt) 中找到完整的程序），就可以通过 `sudo bpftrace dropwatch.bt` 来运行了。



### 如何排查网络丢包问题

可以执行下面的 `nslookup`命令，查询到极客时间的 IP 地址，然后再执行`iptables` 命令，禁止访问极客时间的 80 端口：

```bash
# 首先查询极客时间的IP
$ apt install dnsutils -y & yum install bind-utils -y 
$ nslookup time.geekbang.org
Server:        127.0.0.53
Address:    127.0.0.53#53

Non-authoritative answer:
Name:    time.geekbang.org
Address: 39.106.233.176

# 然后增加防火墙规则阻止80端口
$ sudo iptables -I OUTPUT -d 39.106.233.176/32 -p tcp -m tcp --dport 80 -j DROP
```

防火墙规则加好之后，在终端一中启动跟踪脚本：

```bash
$ sudo bpftrace dropwatch.bt
```

然后，新建一个终端，访问极客时间，应该会看到超时的错误：

```bash
$ curl --connect-timeout 1 39.106.233.176
curl: (28) Connection timed out after 1000 milliseconds
```

返回第一个终端，你就可以看到 eBPF 程序已经成功跟踪到了内核丢包的调用栈信息，如下所示：

```bash
SKB dropped: 192.168.1.129->39.106.233.176, kstack:
        kfree_skb+1
        __ip_local_out+219
        ip_local_out+29
        __ip_queue_xmit+367
        ip_queue_xmit+21
        __tcp_transmit_skb+2237
        tcp_connect+1009
        tcp_v4_connect+951
        __inet_stream_connect+206
        inet_stream_connect+59
        __sys_connect_file+95
        __sys_connect+162
        __x64_sys_connect+24
        do_syscall_64+97
        entry_SYSCALL_64_after_hwframe+68
```

第一行输出中成功拿到了源 IP 和目的 IP，而接下来的每一行中都包含了指令地址、函数名以及函数地址偏移。从下往上看这个调用栈，最后调用 `kfree_skb` 函数的是 `__ip_local_out`，那么 `__ip_local_out` 这个函数又是干什么的呢？根据函数名可以大致猜测出，它是用于向外发送网络包的，但具体的步骤就不太确定了。所以，这时候就需要去参考一下内核源代码。

这里推荐使用 https://elixir.bootlin.com/ 这个网站来查看内核源码，因为它不仅列出了所有内核版本的源代码，还提供了交叉引用的功能。在源码文件中点击任意函数或类型，它就可以自动跳转到其定义和引用的位置。

比如，对于 `__ip_local_out` 函数的定义和引用，就可以通过 https://elixir.bootlin.com/linux/v5.13/A/ident/__ip_local_out （请注意把 v5.13 替换成你的内核版本）这个网址来查看。

<img src="https://static001.geekbang.org/resource/image/98/cc/98891ed6bf8f2d6e273299bba6a3edcc.png?wh=856x614" alt="图片" style="zoom:50%;" />

查询的结果分为三个部分:

-   头文件中的函数声明

-   源码文件中的函数定义

-   其他文件的引用

点击中间部分（即上图红框中的第一个链接），就可以跳转到源码的定义位置。

```c
int __ip_local_out(struct net *net, struct sock *sk, struct sk_buff *skb)
{
    struct iphdr *iph = ip_hdr(skb);

  /* 计算总长度 */
    iph->tot_len = htons(skb->len);
  /* 计算校验和 */
    ip_send_check(iph);

    /* L3主设备处理 */
    skb = l3mdev_ip_out(sk, skb);
    if (unlikely(!skb))
        return 0;

  /* 设置IP协议 */
    skb->protocol = htons(ETH_P_IP);

  /* 调用NF_INET_LOCAL_OUT钩子 */
    return nf_hook(NFPROTO_IPV4, NF_INET_LOCAL_OUT,
               net, sk, skb, NULL, skb_dst(skb)->dev,
               dst_output);
}
```

从这个代码来看，`__ip_local_out` 函数的主要流程就是计算总长度和校验和，再设置 L3 主设备和协议等属性后，最终调用 `nf_hook`。

而 `nf` 就是 netfilter 的缩写，可以将其理解为调用 iptables 规则。再根据 `NF_INET_LOCAL_OUT`参数，可以知道接下来调用了 OUTPUT 链（chain）的钩子。

知道了发生丢包的问题来源，接下来再去定位 iptables 就比较容易了。在终端中执行下面的 iptables 命令，就可以查询 OUTPUT 链的过滤规则：

```bash
sudo iptables -nvL OUTPUT
```

命令执行后，可以看到类似下面的输出。可以看到，正是之前加入的 iptables 规则导致了丢包：

```bash
Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    1   180 DROP       tcp  --  *      *       0.0.0.0/0            39.106.233.176       tcp dpt:80
```

清楚了问题的根源，要解决它当然就很简单了。只要执行下面的命令，把导致丢包的 iptables 规则删除即可：

```bash
sudo iptables -D OUTPUT -d 39.106.233.176/32 -p tcp -m tcp --dport 80 -j DROP
```



### 如何根据函数偏移快速定位源码

在内核栈的输出中，每一个函数的输出格式都是`函数名+偏移量`，而这儿的偏移就是调用下一个函数的位置。Linux 内核维护了一个 [faddr2line](https://github.com/torvalds/linux/blob/master/scripts/faddr2line) 脚本，根据`函数名+偏移量`输出源码文件名和行号。可以点击[这里](https://github.com/torvalds/linux/blob/master/scripts/faddr2line)，把它下载到本地，然后执行下面的命令加上可执行权限：

```bash
chmod +x faddr2line
```

在使用这个脚本之前，需要注意两个前提条件：

-   第一，带有调试信息的内核文件，一般名字为 vmlinux（注意，/boot 目录下面的 vmlinz 是压缩后的内核，不可以直接拿来使用）。
-   第二，系统中需要安装 `awk`、`readelf`、`addr2line`、`size`、`nm` 等命令。

对于第二个条件，这些命令都包含在 [binutils](https://www.gnu.org/software/binutils/) 软件包中，只需要执行 `apt` 或者 `dnf` 命令安装即可。

第一个条件中的内核调试信息, 对于 Ubuntu 来说，执行下面的命令安装调试信息：

```bash
codename=$(lsb_release -cs)
sudo tee /etc/apt/sources.list.d/ddebs.list << EOF
deb http://ddebs.ubuntu.com/ ${codename}      main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-updates  main restricted universe multiverse
EOF
sudo apt-get install -y ubuntu-dbgsym-keyring
sudo apt-get update
sudo apt-get install -y linux-image-$(uname -r)-dbgsym
```

而对于 RHEL8 等系统，则可以执行下面的命令：

```bash
sudo debuginfo-install kernel-$(uname -r)
```

调试信息安装好之后，相关的调试文件会放到 `/usr/lib/debug` 目录下。不同发行版的目录结构是不同的，可以使用下面的命令来搜索 `vmlinux` 开头的文件：

```bash
find /usr/lib/debug/ -name 'vmlinux*'
```

查找到的文件路径，接下来，就可以执行下面的命令，对刚才内核栈中的 `__ip_local_out+219` 进行定位：

```plain
faddr2line /usr/lib/debug/boot/vmlinux-5.13.0-22-generic __ip_local_out+219
```

命令执行后，可以得到下面的输出：

```plain
__ip_local_out+219/0x150:
nf_hook at include/linux/netfilter.h:256
(inlined by) __ip_local_out at net/ipv4/ip_output.c:115
```

这个输出中的具体内容含义如下：

-   第二行表示 `nf_hook`的定义位置在 `netfilter.h` 的`156`行。
-   第三行表示 `net/ipv4/ip_output.c` 的 `115`行调用了 `kfree_skb` 函数。但是，由于 `nf_hook` 是一个内联函数，所以行号`115`实际上是内联函数 `nf_hook` 的调用位置。

对比一下上一个模块查找的[内核源码](https://elixir.bootlin.com/linux/v5.13/source/net/ipv4/ip_output.c#L115)，`net/ipv4/ip_output.c` 的 115 号也刚好是调用 `nf_hook` 的位置：而再点击 [nf_hook](https://elixir.bootlin.com/linux/v5.13/source/include/linux/netfilter.h#L205) 继续去看它的定义，可以发现这的确是个内联函数：

```c++
static inline int nf_hook(...)
```

>   小提示：内联是一种常用的编程优化技术，它告诉编译器把指定函数展开之后再进行编译，这样就省去了函数调用的开销。对频繁调用的小函数来说，这就可以大大提高程序的运行效率。

有了 faddr2line 工具，在以后排查内核协议栈时，可以根据栈中`函数名+偏移量`，直接定位到源代码的位置。这样就可以直接去内核源码或 [elixir.bootlin.com](https://elixir.bootlin.com/) 网站中查找相关函数的实现逻辑，进而更深层次地理解内核的实现原理。



## 开发套接字转发 eBPF 程序

套接字 eBPF 程序工作在内核空间中，无需把网络数据发送到用户空间就能完成转发。

使用套接字映射转发网络包需要以下几个步骤：

1.  **创建**套接字映射；
2.  在 BPF_PROG_TYPE_SOCK_OPS 类型的 eBPF 程序中，将新创建的套接字**存入套接字映射**中；
3.  在流解析类的 eBPF 程序（如 BPF_PROG_TYPE_SK_SKB 或 BPF_PROG_TYPE_SK_MSG ）中，从套接字映射中提取套接字信息，并调用 BPF 辅助函数**转发**网络包；
4.  **加载并挂载** eBPF 程序到套接字事件。



### 1. 创建套接字映射

以 `BPF_MAP_TYPE_SOCKHASH` 类型的套接字映射为例，它的值总是套接字文件描述符，而键则需要我们去定义。比如，可以定义一个包含 IP 协议五元组的结构体，作为套接字映射的键类型：

```c++
struct sock_key
{
    __u32 sip;    //源IP
    __u32 dip;    //目的IP
    __u32 sport;  //源端口
    __u32 dport;  //目的端口
    __u32 family; //协议
};
```

有了键类型之后，就可以使用 `SEC` 关键字来定义套接字映射了，如下所示：

```c++
#include <linux/bpf.h>

struct bpf_map_def SEC("maps") sock_ops_map = {
    .type = BPF_MAP_TYPE_SOCKHASH,
    .key_size = sizeof(struct sock_key),
    .value_size = sizeof(int),
    .max_entries = 65535,
    .map_flags = 0,
};
```



### 2. 更新套接字映射

套接字映射准备好之后，第二步就是在 `BPF_PROG_TYPE_SOCK_OPS` 类型的 eBPF 程序中跟踪套接字事件，并把套接字信息保存到 SOCKHASH 映射中。

参考内核中 `BPF_PROG_TYPE_SOCK_OPS` 程序类型的[定义格式](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf_types.h#L29)，它的参数格式为 [struct bpf_sock_ops](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5506)：

```c++
#define BPF_PROG_TYPE(_id, _name, prog_ctx_type, kern_ctx_type)

BPF_PROG_TYPE(BPF_PROG_TYPE_SOCK_OPS, sock_ops,
    struct bpf_sock_ops, struct bpf_sock_ops_kern)
```

因此可以使用如下的格式来定义这个 eBPF 程序：

```c++
SEC("sockops")
int bpf_sockmap(struct bpf_sock_ops *skops)
{
  // TODO: 添加套接字映射更新操作
}
```

在添加具体的套接字映射更新逻辑之前，还需要先从 `struct bpf_sock_ops`中获取作为键类型的五元组。参考内核中 [struct bpf_sock_ops](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5506) 的定义：

```c++
struct bpf_sock_ops {
    __u32 family;
    __u32 remote_ip4;	/* Stored in network byte order */
    __u32 local_ip4;	/* Stored in network byte order */
    __u32 remote_port;/* Stored in network byte order */
    __u32 local_port;	/* stored in host byte order */
    ...
}
```

因此，可以直接使用它们来定义映射中所需要的键。下面就是 `sock_key` 的定义方法，注意这里把 `local_port` 转换为了同其他字段一样的网络字节序：

```c++
struct sock_key key = {
    .dip = skops->remote_ip4,
    .sip = skops->local_ip4,
    .sport = bpf_htonl(skops->local_port),
    .dport = skops->remote_port,
	.family = skops->family,
};
```

**有了键之后，还不能立刻就去更新套接字映射**。这是因为 `BPF_PROG_TYPE_SOCK_OPS` 程序跟踪了所有类型的套接字操作，而需要把新创建的套接字更新到映射中。

`struct bpf_sock_ops` 中包含的 `op` 字段可用于判断套接字操作类型，其定义格式可以参考[这里的内核头文件](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5638)。内核头文件中已经为每种操作的具体含义加了详细的注释，对于新创建的连接，可以使用以下两个状态（即主动连接和被动连接）作为判断条件：

```c++
/* skip if it is not established op */
if (skops->op != BPF_SOCK_OPS_PASSIVE_ESTABLISHED_CB && skops->op != BPF_SOCK_OPS_ACTIVE_ESTABLISHED_CB) {
      return BPF_OK;
}
```

到这里，说明套接字已经属于新创建的连接了，所以接下来就是**调用 BPF 辅助函数去更新套接字映射**，如下所示：

```c++
bpf_sock_hash_update(skops, &sock_ops_map, &key, BPF_NOEXIST);
```

其中，`BPF_NOEXIST` 表示键不存在的时候才添加新元素。

### 3. 套接字转发

套接字转发可以使用 `BPF_PROG_TYPE_SK_MSG` 类型的 eBPF 程序，捕获套接字中的发送数据包，并根据上述的套接字映射进行转发。根据内核头文件中的[定义格式](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf_types.h#L33)，它的参数格式为 [struct sk_msg_md](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5328).

注意一点：`BPF_PROG_TYPE_SK_MSG` 跟 `BPF_PROG_TYPE_SOCK_OPS` 属于不同的 eBPF 程序。虽然你可以把多个 eBPF 程序放入同一个源码文件，并编译到同一个字节码文件(即 `文件名.o`）中，但由于它们的加载和挂载格式都是不同的，推荐你把不同的 eBPF 程序放入不同的文件中，这样管理起来更为方便。

创建一个新的文件（如 `sockredir.bpf.c`），用于保存 BPF_PROG_TYPE_SK_MSG 程序。添加如下的代码，就定义了一个名为 `bpf_redir` 的 eBPF 程序：

```c++
SEC("sk_msg")
int bpf_redir(struct sk_msg_md *msg)
{
    //TODO: 添加套接字转发逻辑
}
```

### 4. 加载 eBPF 程序

得到套接字映射更新和转发这两个 BPF 字节码之后，还需要把它们加载到内核之中，再挂载到特定的内核事件之后才会生效。

>   可以在 [GitHub](https://github.com/feiskyer/ebpf-apps/blob/main/loadbalancer/sockops/sockredir.bpf.c) 中找到完整的代码



## 开发 XDP eBPF 程序

libbpf 的使用通常分为以下几步：

1.  开发 eBPF 程序，并把源文件命名为 `<程序名>.bpf.c`；
2.  编译 eBPF 程序为字节码，然后再调用 `bpftool gen skeleton` 为 eBPF 字节码生成脚手架头文件；
3.  开发用户态程序，引入生成的脚手架头文件后，加载 eBPF 程序并挂载到相应的内核事件中。



### 1. 开发一个运行在内核态的 eBPF 程序

参考内核中 BPF_PROG_TYPE_XDP 程序类型的[定义格式](https://elixir.bootlin.com/linux/v5.13/source/include/linux/bpf_types.h#L11)，它的参数类型为 [struct xdp_md](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5283)：


因为eBPF程序是通过eBPF虚拟机来执行的。虚拟机需要知道一些信息和规则，这些都被包含在定义为结构体的 `xdp_md` 上下文中。

这个结构体包含关于网络数据包，如数据包长度、数据指针、接口索引等信息。因此，当加载eBPF程序时，传递 `struct xdp_md *ctx` 将使程序能够访问这些内存区域，并从中获得所需的环境变量，以便在程序中进行操作。


```c++
#define BPF_PROG_TYPE(_id, _name, prog_ctx_type, kern_ctx_type)

BPF_PROG_TYPE(BPF_PROG_TYPE_XDP, xdp,
       struct xdp_md, struct xdp_buff)
```

因而可以使用如下的格式来定义这个 XDP 程序：

```c++
SEC("xdp")
int xdp_proxy(struct xdp_md *ctx)
{
  // TODO: 添加XDP负载均衡逻辑
}
```

**`SEC("xdp")` 表示程序的类型为 XDP 程序。可以在 libbpf 中 [section_defs](https://github.com/libbpf/libbpf/blob/master/src/libbpf.c#L8599-L8675)找到所有 eBPF 程序类型对应的段名称格式**。

参考 `linux/bpf.h` 头文件中 [struct xdp_md](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/bpf.h#L5283) 的定义格式

```c++
struct xdp_md {
    __u32 data;
    __u32 data_end;
    __u32 data_meta;
    /* Below access go through struct xdp_rxq_info */
    __u32 ingress_ifindex; /* rxq->dev->ifindex */
    __u32 rx_queue_index;  /* rxq->queue_index  */
    __u32 egress_ifindex;  /* txq->dev->ifindex */
};
```

- data: 一个指向数据包开头的指针
- data_end: 一个指向数据包结尾的指针 (exclusive, 即不包含此位置指向的字节）
- data_meta:一个指向元数据的指针(可选)
- ingress_ifindex: 入站接口的索引号
- rx_queue_index: 接收数据包的队列索引号

在XDP程序中，结构体 `xdp_md` 被传递到用户定义的函数中，可以查看和修改其中的字段，从而实现流量控制等功能.

从 `struct xdp_md` 的定义可以看到，所有的字段都是整型数值，其中前三个表示数据指针信息（包括开始位置、结束位置、元数据位置），而后三个表示关联网卡的信息（包括入口网卡、入口网卡队列以及出口网卡的编号）。

由于 `struct xdp_md` 中并不包含 skb 数据结构，**在 XDP 程序中，只能通过 `data` 和 `data_end` 这两个指针去访问网络报文数据**。而要想利用原始网络数据指针来访问网络数据，就需要了解 TCP/IP 网络报文的基本格式。

![图片](https://static001.geekbang.org/resource/image/yy/91/yy7887570f06c1d075eb31701924e791.jpg?wh=1920x577)

所以，要访问 TCP/IP 协议某一层的头结构，就可以使用开始指针 `data` 再加上它之前所有头结构大小的偏移。

比如，对于以太网头，它的位置跟开始位置 `data` 是相同的，因而就可以使用下面的方式，把它转换为指针格式进行访问：

```c++
void *data = (void *)(long)ctx->data;
void *data_end = (void *)(long)ctx->data_end;

struct ethhdr *eth = data;
/*
为了帮助 eBPF 校验器验证数据访问的合法性，在访问以太网头数据结构 struct ethhdr 之前，
需要检查数据指针是否越界。如果检查失败，就要返回一个错误（这儿返回的 XDP_ABORTED 表示丢弃数据包并记录错误行为以便排错）
*/
if (data + sizeof(struct ethhdr) > data_end)
{
      return XDP_ABORTED;
}

// IP 头的访问也是类似的。在开始指针 data 之后加上太网头数据结构的长度偏移，就是 IP 头所指向的位置
struct iphdr *iph = data + sizeof(struct ethhdr);
if (data + sizeof(struct ethhdr) + sizeof(struct iphdr) > data_end)
{
      return XDP_ABORTED;
}
// 拿到 IP 头之后，还可以对网络数据进行初步的校验，比如忽略 IPv6、UDP 等我们不关心的数据，只处理 TCP 数据包
if (eth->h_proto != bpf_htons(ETH_P_IP))
{
    // 返回的 XDP_PASS 表示把网络包传递给内核协议栈，内核协议栈接收到网络包后，按正常流程继续处理。
	return XDP_PASS;
}

if (iph->protocol != IPPROTO_TCP)
{
      return XDP_PASS;
}
```

参考内核中[以太网头](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/if_ether.h#L165)和 [IP 头](https://elixir.bootlin.com/linux/v5.13/source/include/uapi/linux/ip.h#L86)的定义格式， IP 地址都是以 `__be16` 类型的**大端格式**存储，而 MAC 地址则是以字符数组 `unsigned char h_dest[6]` 的形式存储。因而，MAC 地址可以直接通过数据下标进行访问

```c
struct ethhdr {
	unsigned char	h_dest[ETH_ALEN];	/* destination eth addr	*/
	unsigned char	h_source[ETH_ALEN];	/* source ether addr	*/
	__be16		h_proto;		/* packet type ID field	*/
} __attribute__((packed));

struct iphdr {
#if defined(__LITTLE_ENDIAN_BITFIELD)
	__u8	ihl:4,
		version:4;
#elif defined (__BIG_ENDIAN_BITFIELD)
	__u8	version:4,
  		ihl:4;
#else
#error	"Please fix <asm/byteorder.h>"
#endif
	__u8	tos;
	__be16	tot_len;
	__be16	id;
	__be16	frag_off;
	__u8	ttl;
	__u8	protocol;
	__sum16	check;
	__be32	saddr;
	__be32	daddr;
	/*The options start here. */
};
```

 IP 地址的转换方法:

```c
#include <stdio.h>
#include <arpa/inet.h>

int main() {
  unsigned int a1 = inet_addr("172.17.0.2");
  unsigned int a2 = inet_addr("172.17.0.3");
  unsigned int a3 = inet_addr("172.17.0.4");
  unsigned int a4 = inet_addr("172.17.0.5");
  printf("0x%x 0x%x 0x%x 0x%x\n", a1, a2, a3, a4);
}
```

由于修改了 IP 头的数据，IP 头的校验和（checksum）就需要重新计算，否则网络包会被内核直接丢弃。可以从成熟的开源项目中查找到相关的计算方法。比如，参考 Facebook 开源的 [Katran](https://github.com/facebookincubator/katran/blob/main/katran/lib/bpf/csum_helpers.h#L30)，定义如下的 `ipv4_csum()` 函数来计算校验和：

```c++
static __always_inline __u16 csum_fold_helper(__u64 csum)
{
  int i;
#pragma unroll
  for (i = 0; i < 4; i++)
  {
  if (csum >> 16)
    csum = (csum & 0xffff) + (csum >> 16);
  }
  return ~csum;
}

static __always_inline __u16 ipv4_csum(struct iphdr *iph)
{
  iph->check = 0;
  unsigned long long csum = bpf_csum_diff(0, 0, (unsigned int *)iph, sizeof(struct iphdr), 0);
  return csum_fold_helper(csum);
}
```

关于校验和的具体算法，可以参考 TCP/IP 协议相关的原理书籍（如《TCP/IP详解》）来理解.

有了校验和的计算方法之后，最后更新 IP 头的 checksum，再返回 `XDP_TX` 把数据包从原网卡发送出去，交给内核去转发就可以了。下面展示的就是更新校验和的实现方法：

```c++
SEC("xdp")
int xdp_proxy(struct xdp_md *ctx)
{
  ...
  /* 重新计算校验和 */
  iph->check = ipv4_csum(iph);

  /* 把数据包从原网卡发送出去 */
  return XDP_TX;
}
```

把上述代码保存到一个文件 `xdp-proxy.bpf.c` 中，就完成了 XDP eBPF 程序的开发

>   可以在 [GitHub](https://github.com/feiskyer/ebpf-apps/blob/main/loadbalancer/xdp/xdp-proxy.bpf.c) 中找到完整的代码


静态地匹配目标MAC地址并进行转发：

当源MAC地址与 "36:b9:7d:c9:ab:af" 相同时，使用 XDP_TX 将数据包转发出去，否则使用默认行为 XDP_DROP 
```c
#include <linux/bpf.h>
#include <linux/if_ether.h>

int xdp_prog(struct xdp_md *ctx) {
    void* data = (void*)(long)ctx->data;
    void* data_end = (void*)(long)ctx->data_end;

    struct ethhdr* eth = data;
    if ((void*)eth + sizeof(struct ethhdr) <= data_end) {
        if (eth->h_dest[0] == 0x36 && eth->h_dest[1] == 0xb9 &&
            eth->h_dest[2] == 0x7d && eth->h_dest[3] == 0xc9 &&
            eth->h_dest[4] == 0xab && eth->h_dest[5] == 0xaf) {

            return XDP_TX; 
        }
    }

    return XDP_DROP;  // 默认行为是丢弃包
}
```


### 2. 编译并生成脚手架头文件

有了 XDP 程序之后，接下来的第二步就比较简单了。只需要执行下面的 `clang` 命令，把 XDP 程序编译成字节码，再执行 `bpftool gen skeleton` 命令生成脚手架头文件即可：

```bash
$ clang -g -O2 -target bpf -D__TARGET_ARCH_x86 -I/usr/include/x86_64-linux-gnu -I. -c xdp-proxy.bpf.c -o xdp-proxy.bpf.o

$ bpftool gen skeleton xdp-proxy.bpf.o > xdp-proxy.skel.h
```




### 3. 开发用户态程序


