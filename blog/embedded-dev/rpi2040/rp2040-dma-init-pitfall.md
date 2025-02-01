
# RP2040 DMA 初始化踩坑

如果你要初始化 RP2040 的 DMA，是需要用

```c
    cfg = dma_channel_get_default_config(ejtag_ctx.tdi_chan);
```

这样的代码获取一个默认的配置结构体，然后用

```c
    channel_config_set_read_increment(&cfg, true);
    channel_config_set_write_increment(&cfg, false);
```

这样的代码修改设置，最后再

```c
    dma_channel_configure(ejtag_ctx.tdi_chan, &cfg, ejtag_ctx.tdi_write_addr, NULL, 0, false);
```

应用到 DMA 通道的。

然后呢，由于自己偷懒不想给所有的通道都写一遍，于是给配置 3 个通道的时候都复用了通道 0 的配置结构体。但是这样就踩上了 RP2040 DMA 里面这个坑：

RP2040 的 DMA 是支持 scatter-gather 的，也就是从分散的多段内存中读取然后把它们变成一个连续的流，或者把一个连续的流分段写入到多段内存中。支持 scatter-gather 的方式是 RP2040 的每个 DMA 通道都有一个 `chain_to` 字段，可以通过上述结构体配置，具体效果是如果一个通道的传输完成了，那么就立即启动 `chain_to` 指定的下一通道，这其实就是把对分散的多段内存进行 DMA 的操作自动化了，使之不需要 CPU 进行干预。而禁用一个通道的 `chain_to` 能力的方式就是向该通道的该字段中写入该通道本身的通道号。那么，

```c
static inline dma_channel_config dma_channel_get_default_config(uint channel) {
    dma_channel_config c = {0};
    channel_config_set_read_increment(&c, true);
    channel_config_set_write_increment(&c, false);
    channel_config_set_dreq(&c, DREQ_FORCE);
    channel_config_set_chain_to(&c, channel);
    channel_config_set_transfer_data_size(&c, DMA_SIZE_32);
    channel_config_set_ring(&c, false, 0);
    channel_config_set_bswap(&c, false);
    channel_config_set_irq_quiet(&c, false);
    channel_config_set_enable(&c, true);
    channel_config_set_sniff_enable(&c, false);
    channel_config_set_high_priority( &c, false);
    return c;
}
```

`dma_channel_get_default_config` 函数其实是给出了默认关闭某通道的 `chain_to` 功能的，但是如果要复用这个配置结构体，那就没办法用了，要对每个通道都设置一遍 `chain_to` 才行，如果忘了的话，DMA 就会被其他通道意外激活，导致各种各样奇怪的问题。

其实最终解法还是建议老老实实每个通道都获取一下 default config，以免以后又加了新的和通道号强相关的字段。

本案例来自于 [Pico Loongson EJTAG](https://github.com/RigoLigoRLC/pico-loongson-ejtag) 的 [此版本](https://github.com/RigoLigoRLC/pico-loongson-ejtag/blob/f1bfa0112c230ae67a3fabf90668e6e2478dedd1/src/main.c#L203-L237)。
