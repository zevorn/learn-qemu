---
layout: center
---

# QEMU TCG 基本原理分析

[飞翔的猫咪: QEMU tcg 源码分析与 unicorn 原理](https://bbs.kanxue.com/thread-277163.html)

---

# Accel

QEMU 支持多种 accel，但大体可以分为两种：指令模拟技术（TCG）、虚拟化技术（KVM、HVF）等。

### Interpreter

解释器，每次解析并执行一条 Guest 指令，循环往复。

<br>

### SBT（static binary translation）

静态二进制转换，在程序运行前，进行提前翻译。运行时没有翻译开销，但优化幅度有限。

<br>

### DBT（dynamic binary translation）

动态二进制转换，在程序运行时，将 Guest 指令翻译成 Host 指令。一般翻译程序执行过的路径，不会全量翻译；可以对热点代码进行深度优化；


---

# Introduction

<br>
TCG (Tiny Code Generator) began as a generic backend for a C compiler.
It was simplified to be used in QEMU.  It also has its roots in the
QOP code generator written by Paul Brook.

(ps: docs/devel/tcg-ops.rst)

<br>

### Target

The TCG *target* is the architecture for which we generate the code.
It is of course not the same as the "target" of QEMU which is the
emulated architecture.

We use *guest* to specify what architecture we are
emulating; *target* always means the TCG target, the machine on which
we are running QEMU.

---

# TCG IR

类似 LLVM ，QEMU 也定义有自己的 IR，流程如下：

```
+---------------+      +----------------+      +---------------+
|               |      |                |      |               |
| Source binary | ---> |    QEMU IR     | ---> | Target binary |
|     code      |      |                |      |     code      |
|               |      |                |      |               |
+---------------+      +----------------+      +---------------+
      Guest                                          Host       
```

优势：

- 拓展性好：支持新的前端（Guest），只需要实现 source -> IR；
- 易于流程化：类似 LLVM ，可以引入各种 pass ，对不同环节进行优化。

劣势：

- 性能普遍不高（相对而言）


---

# TranslationBlock

TCG 的二进制转换是以代码块（Basic Block）为基本单元，翻译的产物为 Translation Block。

Basic Block 的划分规则：分支指令；特权指令/异常；代码段跨页。

```
                        +---------------------+                                     
         1)             |                     |                                     
       +----------------+   QEMU TCG engine   +---------------+                     
       |          +---->|                     |<---+          |                     
       |          |     +----------+---^------+    |          |                     
       |          |                |   |   4)      |          | 5)                  
       |          |            3)  |   +------+    |          |                     
       v          |2)              v          |    | 6)       v                     
+---------------+ |        +---------------+  |    |  +---------------+             
|   prologue    | |        |   prologue    |  |    |  |   prologue    |             
+---------------+ |        +---------------+  |    |  +---------------+             
|               | |        |               |  |    |  |               |             
|  Translation  | |        |  Translation  |  |    |  |  Translation  |             
|     Block1    | |        |     Block2    |  |    |  |     Block3    |             
|               | |        |               |  |    |  |               |             
+---------------+ |        +---------------+- |    |  +---------------+             
|   epilogue    | |        |   epilogue    |  |    |  |   epilogue    |             
+------+--------+ |        +-------+-------+  |    |  +------+--------+             
       +----------+                +----------+    +---------+                      
```

---

# Direct block chaining
<br>
拿 x86_64 平台举例，每次执行上下文切换需要执行大约 20 条指令(指令还会进行内存的读写)，

因此 DBT 的优化措施之一就是减少上下文切换，实现 TB 之间的直接链接:

```
            1)          +---------------------+                                 
       +----------------+   QEMU TCG engine   +---------------------------+     
       |                +---------------------+                           |     
       v                                                                  |     
+---------------+          +---------------+          +---------------+   |     
|   prologue    |          |   prologue    |   3)     |   prologue    |   |     
+---------------+ +------> +---------------+  +-----> +---------------+   |     
|               | |        |               |  |       |               |   | 5)  
|  Translation  | |        |  Translation  |  |       |  Translation  |   |     
|     Block1    | |        |     Block2    |  |       |     Block3    |   |     
|               | |2)      |               |  |       |               |   |     
+---------------+ |        +---------------+- |       +---------------+   |     
|   epilogue    | |        |   epilogue    |  |       |   epilogue    |   |     
+------+--------+ |        +-------+-------+  |       +------+--------+   |     
       +----------+                +----------+              +------------+     
```

PS: 两个 chained tb 对应的 Guest 指令需要在同一个 Guest page。

---

# TCG 翻译流程

当一个 Basic Block 被 DBT 转换为 TB 以后，

下次再执行到相同的 Basic Block 直接从缓存中获取TB执行即可，无需再经过转换：

```
                              +---------------+                    
       +----------------------| Do something  |-------------------+
       |                      +---------------+                   |
       v                                                          |
+--------------+       +----------------+ Y       +---------+     |
|   Guest PC   +------>| Check TB Cache +-------->| Exec TB +-----+
+--------------+       +------+---------+         +---------+      
                              | N                      ^           
                              v                        |           
                        +-------------+                |           
                        | translation |                |           
                        +-----+-------+                |           
                              v                        |           
                       +-----------------+             |           
                       |Save TB to Cache +-------------+           
                       +-----------------+                         
```

- Direct block chaining 在执行下一个 TB 时自动完成

---

# Code Buffer

在 qemu 启动的早期会执行一个函数叫 tcg_init_machine, 完成 code_buffer 的申请和初始化。

```
code_buffer = mmap()                                               
|                                             TCGContext.code_ptr  
v                                              v                   
+-----------+----------+-------------+---------+------------------+
|           |          |             |         |                  |
|  prologue | epilogue |  TB.struct  | TB.code |     ...          | size = Host / dynamic_code_size
|           |          |             |         |                  |
+-----------+----------+-------------+---------+------------------+
^           ^                        ^                             
|           |                        |                             
|           tcg_code_gen_epilogue    |                             
|                                    tb.tc.ptr                     
tcg_qemu_tb_exec                                                   
```

- 后续所有代码翻译和执行的工作，都围绕 code_buffer 展开

- TCGContext 的后端管理工作，也是围绕 code_buffer 进行
