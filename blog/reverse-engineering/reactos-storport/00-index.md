
# Storport

由于最开始想在ReactOS贡献崩溃转储功能，误入了Windows储存栈的天坑。从给`MmLoadSystemImage`添加prefix功能的叶子节点出发探索一番后，DarkFire建议我先从Storport下手，解决ReactOS多年来因为UniATA不稳定而日常文件系统爆炸的问题。于是我又跳入了Storport研究的天坑。

此处是学习Storport逻辑的笔记，由Windows内核调试、公开的驱动程序源码（如StorAHCI）、ReactOS代码逻辑等研究内容组成。不保证更新速度。

## 章节

[01 Storport 简述](01-principles.md)
