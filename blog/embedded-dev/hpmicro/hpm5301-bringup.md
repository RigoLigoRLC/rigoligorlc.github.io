
# HPM5301 Bringup

为了给我的毕设扩充芯片库（国产 RISC-V 叠加 JTAG 调试口），买了一点 HPM5301 和 HPM5321 回来玩。
早就有所耳闻 HPM 也是用的 CMake 的 SDK，不过他们这个 SDK 其实并没有做得像 RP2040 一样是很纯正的 CMake 项目，
和 RP2040 提供的也不是同样用法的 GUI 工具，RP2040 的工程生成器是真的为你生成一个工程项目出来，而 HPM 的
只是方便国内 Keil 老登不想配 CMake 而生的一款，只能用来 Configure 和 Build 的工具。至于创建工程嘛！
留给开发者自己探索了，而且它每个示例都是带 BSP 的，离了他那个 Board 活不了了，远洋丁真鉴定为智利障碍，
最小 example 里都封装了大量的东西。

这边先放出一个我自己的最小 Example：

```c
#include <hpm_sysctl_drv.h>
#include <hpm_gpio_drv.h>

int main(void)
{
    // Enable GPIO block
    sysctl_enable_group_resource(HPM_SYSCTL, 0, sysctl_resource_gpio, true);

    gpio_set_pin_output(HPM_GPIO0, GPIO_DI_GPIOA, 10); // LED
    gpio_set_pin_input(HPM_GPIO0, GPIO_DI_GPIOA, 03); // Button

    while (1) {
        int keyPressed = gpio_read_pin(HPM_GPIO0, GPIO_DI_GPIOA, 03);
        gpio_write_pin(HPM_GPIO0, GPIO_DI_GPIOA, 10, !keyPressed); // Button is active low
    }
}
```

初始化 GPIO，然后读取按钮状态并传输给 LED。时钟啥的都没配置。

另外我工程是自建的，手动建立工程也很难绷，基本上都让你去复制官方工程，这个东西很垃圾，
于是我自己写了一点导入 HPM SDK 的 CMake 脚本，摆脱那个 Configure 工具的苦海，当然了
CMake 脚本还是得你自己写。但是这样搞已经比较好了，起码比起官方那个逼用法更像个正规军的 CMake 工程。
不过和 RP2040 比起来依然不行，RP2040 可以用 `target_link_libraries` 启用 SDK 提供的外设、
还有 tinyusb 之类的第三方库，而且创建目标是自己用 `add_executable` 做的，HPM 给你全封装起来了，
四不像，极大差评。

这玩意是随手写的，我只自己在 HPM5301EVKLite 上用过，玩玩。
<https://github.com/RigoLigoRLC/hpm-sdk-importer>

当然自己在 VSCode 里编译的话 SES 不认的，要去 SES 里编译。不过可以对接开源工具链。
但话又说回来开源 OpenOCD 对 5301 官方支持根本没做的，当年没买 JLink 的时候拿 OpenOCD 调试
到处断连，一堆毛病，玩个蛋。还是学 RP2040 用 UF2 最方便。
