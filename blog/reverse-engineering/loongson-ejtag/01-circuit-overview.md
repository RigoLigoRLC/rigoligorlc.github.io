
# EJTAG 电路简介

## MIPS EJTAG

EJTAG 实际是 MIPS 的标准，而 EJTAG 出现在龙芯处理器上也是在龙芯 MIPS 时代。

这里随便给出一个标准文件，后续所有内容以及页面，在语境意及 MIPS EJTAG 标准文档时均称“**标准**”，并引用其 PDF 页码：http://www.t-es-t.hu/download/mips/md00047f.pdf ，文档编号：MD00047，修订版本：6.10 。

EJTAG 实际上是在 IEEE JTAG 的基础上进行了一些增强和修改而成。不同之处简单来说：

- 两者主要的的电气接口、信号传输方式、时序是相同的
- 两者的 TAP 状态机逻辑是相同的
- EJTAG 定义了很多为了调试 MIPS 处理器新增的 TAP 寄存器（龙芯也就是使用这些东西进行调试）

## TAP 电气接口

JTAG 的 TAP（Test Access Port，这是在被调芯片上的电路）有五条信号：

- **\#TRST** 低电平有效，将 TAP 复位
- **TDI** 数据进入 TAP 的端口。TDI 那头接的是什么取决于 TAP 状态，由 TMS 控制
- **TDO** 数据输出 TAP 的端口。一般 TDI 和 TDO 在被 TAP 接到移位寄存器的时候才有用。
- **TCK** 数据时钟，上升沿从 TDI 移入一位、从 TMS 采样一位，下降沿 TDO 移出一位
- **TMS** TAP 状态控制信号，每次 TCK 上升沿时 TAP 采样 TMS，然后决定 TAP 的下一个状态。控制 TDI/TDO 连接什么、控制读写 TAP 寄存器都要用到 TMS。

## TAP 使用方式

具体 TAP 状态机的状态转移见标准 90 页。复位 TAP 后位于`Test-Logic-Reset`状态，而后 TAP 通过每时钟上升沿时采样 TMS，使 TAP 状态转移，比如序列 `010` 会使 DR 内容被放进移位寄存器，此时不断在 TMS 上发送 0 使 DR 原内容被从 TDO 移出、新内容被从 TDI 读入。TDO 和 TDI 都是最低位优先进出。然后在 TMS 上发送 `110` 就会使 新数据被写入 DR、然后 TAP 回到 `Run-Test/Idle` 状态。JTAG TAP 的其他性质不再在此赘述，请自行检索。

## MIPS EJTAG 插座

见标准 197 页，为标准 2.54mm 排针。龙芯官方的 EJTAG 调试探头沿用了这一形制，但开发板基本都会为了节省空间而使用其他引脚定义。

||||||||
|-|-|-|-|-|-|-|
|\#TRST|TDI|TDO|TMS|TCK|#BRST|DINT|
|1|3|5|7|9|11|13|
|2|4|6|8|10|12|14|
|GND|GND|GND|GND|GND|空脚|VIO|

官方 EJTAG 调试探头的第 12 脚为空脚，PCB上使用了缺 pin 的排针焊接，但其排线上并没有堵上这个 pin 的空位。其排线可以使用 IDC 插座，但官方硬件没有使用。我个人是推荐使用 IDC 插座提供防呆设计的。

官方 EJTAG 调试探头使用 3.3V 电平，而被调试的芯片可能需要电平转换 IC（如 2K0300 久久派需使用中科云售卖的转换器，将标准 EJTAG 接口转换为其小尺寸接口、将 3.3V 转换为 1.8V 逻辑电平）。

对官方 EJTAG 硬件的其他描述见对应章节。