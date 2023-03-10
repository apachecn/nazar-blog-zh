# JavaScript 中的 GameBoy 仿真:CPU

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/GameBoy-Emulation-in-JavaScript>

这是 JavaScript 仿真开发系列文章的第 1 部分；目前有 10 个部分可用，其他部分预计将随后推出。

*   [Part 1: CPU](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-The-CPU)
*   [Part 2: Memory](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Memory)
*   [Part 3: GPU Timing](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-GPU-Timings)
*   [Part 4: Graphics card](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Graphics)
*   [Part 5: Integration](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Integration)
*   [Part 6: Enter](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Input)
[第 7 部分:精灵](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Sprites)

*   [the first](http://web.archive.org/web/20220810161353/http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-Interrupts)

人们常说 JavaScript 是一种特殊用途的语言，它是为网站设计的，用于支持动态交互。然而，JavaScript 是一种完全面向对象的编程语言，除了 Web 之外，它还用于其他领域:Windows 和苹果 Mac OS 的最新版本中可用的小部件都是用 JavaScript 实现的，Mozilla 应用程序套件的 GUI 也是如此。

随着最近 HTML 中引入了`<canvas>`标签，JavaScript 程序是否能够模拟系统的问题就出现了，就像桌面应用程序可以模拟 Commodore 64、GameBoy Advance 和其他游戏控制台一样。检查这是否可行的最简单的方法当然是用 JavaScript 编写这样一个模拟器。

本文通过为模拟物理机器的每个部分打下基础，开始实现 GameBoy 模拟的基础。起点是 CPU。

### 模型

传统的计算机模型是一个处理单元，由指令程序告诉它做什么；这个程序可以用它自己的特殊内存访问，也可以和普通内存放在同一个区域，这取决于计算机。每条指令都需要很短的时间来运行，而且它们都是一个接一个地运行的。从 CPU 的角度来看，计算机一开机就启动一个循环，从内存中取出一条指令，计算出它所说的内容，并执行它。

为了跟踪 CPU 在程序中的位置，CPU 持有一个称为程序计数器(PC)的数字。从内存中取出指令后，无论指令由多少字节组成，PC 都会前进。

![Fetch-decode-execute loop](img/791f193b5a6489617c6ae294a4a0a8b8.png) *Figure 1: The fetch-decode-execute loop*

原版 GameBoy 中的 CPU 是改装的 Zilog Z80，所以以下几点比较中肯:

*   Z80 是一个 8 位芯片，所以所有的内部工作一次处理一个字节；
*   存储器接口最多可寻址 65，536 字节(16 位地址总线)；
*   程序通过与普通存储器相同的地址总线被访问；
*   指令可以是 1 到 3 个字节之间的任何值。

除了 PC 之外，CPU 内部还保存了其他可用于计算的数字，它们被称为寄存器:A、B、C、D、E、H 和 l。它们中的每一个都是一个字节，因此每一个都可以保存 0 到 255 之间的值。Z80 中的大多数指令用于处理这些寄存器中的值:将值从内存加载到寄存器中，加上或减去值，等等。

如果在一条指令的第一个字节中有 256 个可能的值，那么在基本表中就有 256 条可能的指令。该表在本网站发布的 [Gameboy Z80 操作码图](/web/20220810161353/https://imrannazar.com/Gameboy-Z80-Opcode-Map)中有详细说明。这些中的每一个都可以通过 JavaScript 函数来模拟，该函数在寄存器的内部模型上操作，并对存储器接口的内部模型产生影响。

Z80 中还有其他处理保持状态的寄存器:flags 寄存器(F)，其操作将在下面讨论；堆栈指针(SP)与 PUSH 和 POP 指令一起用于基本的 LIFO 值处理。因此，Z80 仿真的基本模型需要以下组件:

*   内部状态:
    *   用于保持寄存器的当前状态的结构；
    *   执行最后一条指令所用的时间；
    *   CPU 已经运行的总时间；
*   模拟每个指令的函数；
*   将所述函数映射到操作码映射上的表；
*   一个已知的接口与模拟内存对话。

内部状态可以保持如下:

#### Z80.js:内部状态值

```
Z80 = {
    // Time clock: The Z80 holds two types of clock (m and t)
    _clock: {m:0, t:0},

    // Register set
    _r: {
        a:0, b:0, c:0, d:0, e:0, h:0, l:0, f:0,    // 8-bit registers
        pc:0, sp:0,                                // 16-bit registers
        m:0, t:0                                   // Clock for last instr
    }
};
```

标志寄存器(F)对处理器的功能很重要:它根据上一次操作的结果自动计算某些位或标志。Gameboy Z80 中有四面旗帜:

*   Zero (0x80):如果最后一次操作的结果为 0，则设置该位；
*   Operation (0x40):如果最后一个操作是减法，则置位；
*   半进位(0x20):如果在最后一次操作的结果中，字节的下半部分溢出超过 15，则设置该位；
*   进位(0x10):如果最后一次操作产生的结果大于 255(对于加法)或小于 0(对于减法)，则设置该位。

由于基本计算寄存器是 8 位寄存器，如果计算结果溢出寄存器，进位标志允许软件计算出值的变化情况。考虑到这些标志处理问题，下面给出了几个指令模拟的例子。这些例子都是简化的，不计算半进位标志。

#### Z80.js:指令模拟

```
Z80 = {
    // Internal state
    _clock: {m:0, t:0},
    _r: {a:0, b:0, c:0, d:0, e:0, h:0, l:0, f:0, pc:0, sp:0, m:0, t:0},

    // Add E to A, leaving result in A (ADD A, E)
    ADDr_e: function() {
        Z80._r.a += Z80._r.e;                      // Perform addition
        Z80._r.f = 0;                              // Clear flags
        if(!(Z80._r.a & 255)) Z80._r.f |= 0x80;    // Check for zero
        if(Z80._r.a > 255) Z80._r.f |= 0x10;       // Check for carry
        Z80._r.a &= 255;                           // Mask to 8-bits
        Z80._r.m = 1; Z80._r.t = 4;                // 1 M-time taken
    }

    // Compare B to A, setting flags (CP A, B)
    CPr_b: function() {
        var i = Z80._r.a;                          // Temp copy of A
        i -= Z80._r.b;                             // Subtract B
        Z80._r.f |= 0x40;                          // Set subtraction flag
        if(!(i & 255)) Z80._r.f |= 0x80;           // Check for zero
        if(i < 0) Z80._r.f |= 0x10;                // Check for underflow
        Z80._r.m = 1; Z80._r.t = 4;                // 1 M-time taken
    }

    // No-operation (NOP)
    NOP: function() {
        Z80._r.m = 1; Z80._r.t = 4;                // 1 M-time taken
    }
};
```

### 存储器接口

一个能够在自身内部操作寄存器的处理器当然很好，但是它必须能够将结果存入内存才是有用的。同样，上述 CPU 仿真需要一个接口来仿真内存；这可以由存储器管理单元(MMU)来提供。由于 Gameboy 本身不包含复杂的 MMU，所以模拟单元可以非常简单。

此时，CPU 只需要知道接口存在；Gameboy 如何将内存和硬件映射到地址总线上的细节对于处理器的操作来说是无关紧要的。CPU 需要四种操作:

#### 内存接口

```
MMU = {
    rb: function(addr) { /* Read 8-bit byte from a given address */ },
    rw: function(addr) { /* Read 16-bit word from a given address */ },

    wb: function(addr, val) { /* Write 8-bit byte to a given address */ },
    ww: function(addr, val) { /* Write 16-bit word to a given address */ }
};
```

有了这些，剩下的 CPU 指令就可以模拟了。下面是另外几个例子:

#### Z80.js:内存处理指令

```
    // Push registers B and C to the stack (PUSH BC)
    PUSHBC: function() {
        Z80._r.sp--;                               // Drop through the stack
	MMU.wb(Z80._r.sp, Z80._r.b);               // Write B
	Z80._r.sp--;                               // Drop through the stack
	MMU.wb(Z80._r.sp, Z80._r.c);               // Write C
	Z80._r.m = 3; Z80._r.t = 12;               // 3 M-times taken
    },

    // Pop registers H and L off the stack (POP HL)
    POPHL: function() {
        Z80._r.l = MMU.rb(Z80._r.sp);              // Read L
	Z80._r.sp++;                               // Move back up the stack
	Z80._r.h = MMU.rb(Z80._r.sp);              // Read H
	Z80._r.sp++;                               // Move back up the stack
	Z80._r.m = 3; Z80._r.t = 12;               // 3 M-times taken
    }

    // Read a byte from absolute location into A (LD A, addr)
    LDAmm: function() {
        var addr = MMU.rw(Z80._r.pc);              // Get address from instr
	Z80._r.pc += 2;                            // Advance PC
	Z80._r.a = MMU.rb(addr);                   // Read from address
	Z80._r.m = 4; Z80._r.t=16;                 // 4 M-times taken
    }
```

### 调度和重置

指令就位后，CPU 剩下的难题是在 CPU 启动时重置它，并向仿真例程提供指令。有一个复位例程允许 CPU 停止并“倒回”到执行的开始；下面是一个例子。

#### Z80.js:重置

```
    reset: function() {
	Z80._r.a = 0; Z80._r.b = 0; Z80._r.c = 0; Z80._r.d = 0;
	Z80._r.e = 0; Z80._r.h = 0; Z80._r.l = 0; Z80._r.f = 0;
	Z80._r.sp = 0;
	Z80._r.pc = 0;      // Start execution at 0

	Z80._clock.m = 0; Z80._clock.t = 0;
    }
```

为了运行模拟，它必须模拟前面详述的提取-解码-执行序列。“执行”由指令仿真函数负责，但提取和解码需要一段专门的代码，称为“调度循环”。这个循环获取每条指令，在指令必须被发送执行的地方进行解码，并将其分派给相关的函数。

#### Z80.js:调度员

```
while(true)
{
    var op = MMU.rb(Z80._r.pc++);              // Fetch instruction
    Z80._map[op]();                            // Dispatch
    Z80._r.pc &= 65535;                        // Mask PC to 16 bits
    Z80._clock.m += Z80._r.m;                  // Add time to CPU clock
    Z80._clock.t += Z80._r.t;
}

Z80._map = [
    Z80._ops.NOP,
    Z80._ops.LDBCnn,
    Z80._ops.LDBCmA,
    Z80._ops.INCBC,
    Z80._ops.INCr_b,
    ...
];
```

### 在系统仿真中的使用

如果没有仿真器来运行，实现 Z80 仿真内核是没有用的。在本系列的下一部分，模拟 Gameboy 的工作开始了:我将查看 Gameboy 的内存映射，以及如何通过 Web 将游戏图像加载到模拟器中。

完整的 Z80 核心可在:[http://imrannazar.com/content/files/jsgb.z80.js](http://web.archive.org/web/20220810161353/http://imrannazar.com/content/files/jsgb.z80.js)；如果您在实现中遇到任何错误，请随时告诉我。

2010 年 7 月，伊姆兰·纳扎尔<>。

*文章日期:2010 年 7 月 22 日*