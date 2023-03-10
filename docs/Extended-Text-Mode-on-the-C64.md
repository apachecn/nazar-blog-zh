# C64 上的扩展文本模式

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Extended-Text-Mode-on-the-C64>

我有时会想，如何才能让一台 80 年代的电脑成为当今世界可行的工作终端。经过一番思考，我想到了旧电脑需要的两样东西:

Internet access:

Getting data to an old computer without the Internet is a thankless task, involving microcassette recordings and funky formats of floppy disc which are unreadable in a PC. Getting the data back off the computer in question is even more of a problem; it's simply orders of magnitude easier to connect to a standard network and transfer through that.

Display compatibility:

Computers of such vintage as the Commodore 64 and the Spectrum have a variety of display modes, but none of them put a great deal of information on the screen. To viably communicate with a computer of a different type, such as a Linux server, it's a prerequisite to extend the text mode capabilities of the computer, and get something approaching a usable terminal.

本文是系列文章的第 1 部分，其中将设置 Commodore 64 作为标准的 Unix 兼容终端。雄心勃勃计划的第一步是在 Commodore 64 上提供合理的文本显示。

### C64 的视频模式

Commodore 64 的显示分辨率为 320×200 像素，这个分辨率对于任何处理过 VGA 显示模式的 PC 程序员来说都是熟悉的。C64 提供了两种基本的显示模式:位图和平铺。每一种都能够在一种颜色的背景上显示另一种颜色的像素。在这两种模式中，显示器在逻辑上被分解成 8×8 像素的“瓦片”；区别在于这两种模式下如何处理和绘制图形。

在平铺模式(也称为“文本模式”)下，屏幕的平铺分辨率为 40 宽 25 高，显示方式如下。

![Tiled mode](img/5725fbe411e104c25f14f16f4e153db8.png) *Figure 1a: Display in tiled mode*

图块地址缓冲区，通常称为“屏幕内存”，是一个 1000 字节的内存区域，其中每个字节表示屏幕上的一个 8×8 块。为了获得用于显示的位图数据，视频电路使用屏幕存储器中的值作为指向图块数据缓冲区(称为“字符存储器”)的指针。当计算机第一次启动时，这个内存包含字母和数字的形状，可以用来在屏幕上绘制文本；因此，平铺模式通常被称为“40x25 文本模式”。

平铺模式允许每个 8×8 像素块具有不同的前景色；图块中设置为“1”的任何位将以该图块的前景色绘制。就像屏幕内存一样，一个 1000 字节的区域是为瓷砖颜色缓冲区留出的，称为“颜色内存”，它为每个瓷砖提供前景色。整个屏幕上的背景是相同的，任何“0”位将以全局背景色绘制。

位图模式跳过显示过程中的平铺寻址步骤，而是选择位图数据的统一缓冲区。留出 8000 字节的区域用于 320x200 位的显示，8×8 的块仍然作为一个图块被寻址。

![Drawing in bitmapped mode](img/2389c9ac39dd6d268d0b207503be116b.png) *Figure 1b: Drawing in bitmapped mode*

就像平铺模式一样，每个 8x8 块可以有不同的前景色。然而，在位图模式的情况下，一个块也可以有它自己的背景颜色，如果在位图中遇到任何“0”位，它将代替全局背景使用。

### 选项

所以我们有两个选择来绘制到 C64 的屏幕上。这两种模式都可以用于 80x25 文本模式的渲染，但是有一些反对使用平铺模式的理由:

It's more complex:

Drawing to a bitmapped screen involves writing the appropriate bits to a piece of the bitmap buffer. In tiled mode, a tile has to be written into Character Memory, and Screen Memory has to be updated to reflect the new tile.

It's slower to work with:

Because of the above complexity of tiled mode, more memory has to be worked with to set a tile up correctly, which means it takes more time to write text out to screen.

It's not big enough:

By placing an 80x25 "extended text mode" into 40x25 tiled mode, each tile can hold two characters. In theory, there are over 65,000 combinations of two-character tiles, any of which could show up on screen; tiled mode can only deal with 256 of these combinations on screen at the same time, before some hacks have to be employed.

由于这些原因，使用位图模式来绘制字符更简单。现在需要的是渲染系统可以使用的可读字体。

### 字体

大多数终端系统使用单倍行距的字体，主要是因为这样可以更容易地计算文本的位置和大小。这种扩展的文本模式也不例外:为了使 80x25 的文本屏幕适合 320x200 的图形显示，每个字符必须是 4x8 像素:换句话说，每个 8x8 的拼贴必须从中间切开，每一半放一个字符。

这没有考虑到分隔字符的需要:如果字体是由 4x8 像素的字形组成的，那么一行文本中的每个字符都将与下一个字符相连，没有任何分隔。相反，需要的是字符之间的像素间隔:这意味着字体将由 4x8 框中的 3x7 像素字形组成。

在如此小的范围内，设计一个清晰的字体是很棘手的:即使在最好的情况下，区分零(0)和大写 O 也是很困难的，而一(1)、小 L 和竖线(|)之间的区别可能更是一个问题。我不是字体设计师，所以我选择使用 Novaterm 的字体符号，nova term 是 C64 的一个终端程序。

![The font](img/298959df20cba4f90f33e11c46679df6.png) *Figure 2: 3x7 pixel font, Novaterm's "ansi81"*

为了以编程方式使用这种字体，每个字符都必须被分解成它的组成部分，并重新组合成数据。因为字形是 4 像素宽，所以结果数据将是 4 位宽。

### 该过程

使用字体和位图模式，我们现在可以将文本绘制到位图中。不幸的是，这并不像在每个图块上写一个字符那么容易，因为整个屏幕上只有 40 个图块的空间。相反，两个字符必须放在一个瓷砖空间内。这包括移动“左”字符的位图值，并将它们与“右”字符组合。

![Drawing into one tile](img/c4a69378762e8f976fd8f81ee021adf6.png) *Figure 3: Drawing two characters in one tile*

在基本代码中，这可以表示如下，假设`FONT`二维数组表示 3x7 Novaterm 字体:

#### 在 BASIC 中呈现“他”

```
LET CH1 = 72: REM "H"
LET CH2 = 101: REM "e"
FOR A = 0 TO 7
  OUT(A) = (FONT(CH1)(A) * 16) + FONT(CH2)(A)
NEXT A
```

通过使用“光标”位置来跟踪屏幕上必须填充的区域，使用上面的技术一次呈现两个字符相对简单。当一个文本字符串包含奇数个字符时，问题就出现了:不仅渲染器必须填充半个图块而不是整个图块，而且下一个字符串将从所讨论的图块的中间开始。因此，渲染功能变得更加复杂:

*   检查我们是否从瓷砖的一半开始。如果是这样，用第一个字符位图填充现有屏幕区域的右半部分。如果没有，完全跳过这一步。
*   主渲染循环:对于字符串中的每一对字符(如果执行了步骤 1，则从第二个字符开始)，构建一个图块并将其绘制到屏幕上。这样做，直到剩下 1 个或 0 个字符需要渲染。
*   如果还有一个字符，填充空白方块的左半部分，并将其绘制到屏幕上。如果没有要画的字符，跳过这一步。

最上面和最下面的算法是主渲染循环的扩展，这里就不详细介绍了。相反，我将用伪 C++来解释主循环的内部。

#### 呈现两个字符的平铺

```
BYTE *bitmap;     // 8000-byte bitmap to render to
BYTE *font;       // 2048 bytes font data, 8 bytes per char
char t1, t2;      // Text to render (two characters long)

int X, Y;         // Current cursor position

// Calculate position of destination tile in the bitmap
// Each tile is 8 bytes long
BYTE *tile = bitmap + ((Y * 80 + X) * 8);

for(int i=0; i<8; i++)
{
    // Retrieve font data for this line of the bitmap
    BYTE ch1 = font[t1 * 8 + i];
    BYTE ch2 = font[t2 * 8 + i];

    // Calculate final tile contents
    tile[i] = (ch1 * 16) + ch2;
}
```

在算法的顶部和底部的情况下，`ch1`或`ch2`在最终的图块中不被使用；否则，这些部分的代码如上。

### 实施

为了应对平台的限制，在 Commodore 64 上使用该算法时，需要考虑一些事情。

#### 字体数据:

上面概述的算法从字体数据中提取两个字符，并在将“左”字符定位到“右”字符之前，将“左”字符移动 4 位。可以通过将字体数据的预转换副本作为原始字体的独立缓冲区来消除这一步骤，这意味着构建一个单元格只需在原始字体中找到一个字符，在转换后的字体中找到另一个字符，然后将这两个值相加。

#### 初始屏幕颜色:

如上所述，位图模式下的每个图块都可以保持自己的前景色和背景色。每个图块的颜色值存储在颜色存储器中，每个图块一个字节:背景颜色代码(0 到 15 之间)存储在字节的下半部分，前景代码存储在字节的上半部分。

出于本文的目的，我们不会处理不同颜色的文本或其他属性，所以所有需要的是初始化颜色内存:设置所有字节以反映“黑色上的灰色”允许简单的单色输出。

#### 乘法:

Commodore 64 使用的 6510 CPU 没有乘法指令，这意味着我们不能简单地“乘以 8”来获得字体数据位置。幸运的是，我们可以使用二进制幂的一个基本属性来进行计算:

#### 二进制乘方乘法

```
x * 8 = x * (2 ** 3)
x * 8 = x << 3
```

通过左移值，我们可以模拟乘法。然而，在这种情况下，这还不够。左移一个值会将最左边的位推出寄存器的末尾，丢弃结果的较高部分:我们需要该较高部分，因此需要更多的计算:

#### 乘以二进制幂得到双倍宽度的结果

```
x = 01101101b
LOBYTE(x * 8) = 01101101 << 3
              = [*011*]01101000
HIBYTE(x * 8) = [*011*]
              = 01101101 >> (8-3)

LOBYTE(x * 8) = x << 3
HIBYTE(x * 8) = x >> 5
```

上面的示例演示了一个更一般的规则:1 字节乘 1 字节的乘法将生成 2 字节的结果，这两部分都可以通过适当的移位来计算。该规则可由实现的 6510 代码使用。

### 结果呢

将这些算法写入代码后，将会生成类似下面的内容:

![Lorem ipsum](img/2be9845107047ffed12dfb07294c9131.png) *Figure 5: Lorem ipsum on the C64*

在上面的例子中，处理换行符的附加算法被添加到 80x25 显示系统中，允许文本包含换行符。这只是在遇到换行符时将光标向下移动到下一行的开始。

系统当前不处理文本缓冲区的滚动:如果文本被绘制在第 25 行之下，它将不会出现在显示器上。滚动和其他控制序列，包括字符颜色，将在本系列的第 2 部分讨论。

[80x25.s: 6510 汇编源码](http://web.archive.org/web/20220810161352/http://oopsilon.com/c64/80x25.s)
[ansi.font:编码字体数据](http://web.archive.org/web/20220810161352/http://oopsilon.com/c64/ansi.font)
[80x25.prg:汇编二进制，仿真就绪](http://web.archive.org/web/20220810161352/http://oopsilon.com/c64/80x25.prg)

[伊姆兰·纳扎尔·(tf@oopsilon.com)](http://web.archive.org/web/20220810161352/mailto:tf@oopsilon.com)

*文章日期:2008 年 8 月 3 日*