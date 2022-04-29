<img src="C:\Users\53047\AppData\Roaming\Typora\typora-user-images\image-20220418232047901.png" alt="image-20220418232047901" style="zoom:80%;" />

#### 关于bpf架构原理图



# 编写属于你的第一个ebpf程序！

#### 使用bcc进行程序开发，我们来看一个例子：

```python
from bcc import BPF
BPF(text='int kprobe__sys_clone(void *ctx) { bpf_trace_printk("Hello, World!\\n"); return 0; }').trace_print()
```

其中：

1. `text='...'`: 定义了一个使用C编写的内联的BPF程序。

2. `kprobe__sys_clone()`: 这是通过kprobes对内核进行动态跟踪的一种捷径。如果C函数以`kprobe__`开头，剩下的部分将作为内核函数名来处理，本例为`sys_clone()`，

   （注：此时可用`kprobe_`来使用内核自己的函数，`sys_clone`内核自己的函数。或不用此开头使用自己编写的函数）

3. `void *ctx`: ctx包含参数，但鉴于此处没有使用，因此将其强转为`void *`

4. `bpf_trace_printk()`: 将printf()输出到通用的trace_pipe (/sys/kernel/debug/tracing/trace_pipe)的一个简单内核工具。该函数对于一些场景是合适的，但也有限制：最多3个参数，仅有一个%s，且trace_pipe是全局共享的，因此并行的程序会导致输出混乱。

5. `return 0`; kprobe__sys_clone函数的出参类型为int，必须返回一个值。

6. `.trace_print()`: 一个bcc例程，从trace_pipe中读取，并打印输出。

   #### 其结果是：

               ./examples/hello_world.py
               bash-13364 [002] d... 24573433.052937: : Hello, World!
               bash-13364 [003] d... 24573436.642808: : Hello, World!

#### 同时也可使用格式:

```python
prog = """
int hello(void *ctx) {
    bpf_trace_printk("Hello, World!\\n");
    return 0;
}
"""
# load BPF program
b = BPF(text=prog)
b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")
```

1. `prog =`: 将C程序定义为一个变量，后面会引用它，便于使用命令行参数时进行字符串替换。
2. `hello()`: 此处定义了一个C函数，而没有直接使用`kprobe__`开头的系统调用，后续会引用它。定义在BPF程序中的C函数将会运行在一个probe上，这些函数都使用`pt_reg* ctx`作为第一个参数。如果需要定义一些不在probe上执行的辅助函数，则需要将这些函数定义为`static inline`，以便编译器进行内联，有时还需要向其添加`_always_inline function`属性。
3. `b.attach_kprobe(event=b.get_syscall_fnname("clone"), fn_name="hello")`:为内核clone系统调用函数创建一个kprobe，该kprobe会执行自定义的hello()函数。可以通过多次执行attach_kprobe() ，将自定义的C函数附加到多个内核函数上。
4. `b.trace_fields()`:从trace_pipe返回固定的字段集，与trace_print()类似，对黑客来说很方便，但实际作为工具时，推荐使用BPF_PERF_OUTPUT()。

##### 	BPF_PERF_OUTPUT():

​	语法格式：BPF_PERF_OUTPUT(name)

​	创建BPF table，用于通过PE RF环缓冲区将自定义事件数据推送到用户空间。这是将每个	事件数据推送到用户空间的首选方法。

​	一个例子：

​	

```c
struct data_t {
    u32 pid;
    u64 ts;
    char comm[TASK_COMM_LEN];
};
BPF_PERF_OUTPUT(events);

int hello(struct pt_regs *ctx) {
    struct data_t data = {};
	data.pid = bpf_get_current_pid_tgid();
	data.ts = bpf_ktime_get_ns();
	bpf_get_current_comm(&data.comm, sizeof(data.comm));

	events.perf_submit(ctx, &data, sizeof(data));

	return 0;
}
```
​	返回的table是events，数据是通过events.perf_submit()方法传入进去的。下面一个就进行	解释。

#### 	perf_submit():

​	语法格式：`int perf_submit((void *)ctx, (void *)data, u32 data_size)`

​	返回值：成功返回0

​	这是一个BPF_PERF_OUTPUT的方法，用于提交特定数据到用户空间。

​	ctx参数是由kprobes和kretprobes提供的，对于`SCHED_CLS` 或者 `SOCKET_FILTER` 程	  	序, 必须使用 `struct __sk_buff *skb` 替代

### 总结

- BCC的代码分为python和C两部分，C代码通常运行在内核态，用于打点采集系统调用的信息，然后将结果返回给python代码；也可以用于对特定lib的函数进行打点探测。
- BCC的核心为C代码，需要了解系统调用接口，作为信息采集点；以及BCC本身提供的C代码，提供了获取当前时间，进程PID，map数据结构，输出等功能。
- 推荐`TRACEPOINT_PROBE`进行打点检测，`perf list`可以列出系统支持的tracepoint，该tracepoint基本包含了系统的大部分系统调用，如`clone`，`sync`等(要求内核版本>=4.7)，但网络相关的还是需要使用kprobes对内核函数进行打点，如`icmp_rcv`。













