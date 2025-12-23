# 龙芯3A4000平台DDR4内存控制器初始化详细分析（由OpenCode驱动DeepSeek编写）

此文件由 [loongson-community/pmon](https://github.com/loongson-community/pmon) 中的 Targets\Bonito3a4000_7a\Bonito\ddr4_dir 分析而来。使用了 OpenCode 驱动 `deepseek-reasoner` 模型执行。请在使用时仔细核对事实，防止模型幻觉对工作产生误导。

## 概述

本文档基于对 `Targets/Bonito3a4000_7a/Bonito/ddr4_dir/` 目录下所有汇编文件的深入分析，详细梳理了龙芯3A4000平台DDR4内存控制器的初始化流程、各模块功能和交互逻辑。

## 文件结构分析

### 1. 头文件定义
- **`ddr_config_define.h`**：内存配置宏定义，包含s1寄存器格式定义、时序参数转换宏等
- **`ddr_config_define_v1.h`**：3A4000专用配置定义，支持SPD读取和自动配置
- **`ls3a4000_reg_def.h`**：DDR寄存器偏移量定义，包含PHY/CTL/MON/TST四大地址空间

### 2. 核心初始化文件

#### 2.1 配置探测阶段
| 文件 | 功能 | 关键输出 |
|------|------|----------|
| **`detect_node_dimm_all.S`** | 节点DIMM全检测 | 读取所有插槽SPD信息，整合为s1配置字 |
| **`detect_channel_dimm.S`** | 通道DIMM检测 | 探测指定通道的DIMM，处理双插槽兼容性 |
| **`config_ddr_capacity_tmp.S`** | 容量计算 | 基于SPD计算每个DIMM的物理容量 |

#### 2.2 寄存器配置阶段
| 文件 | 功能 | 关键操作 |
|------|------|----------|
| **`ddr4_register_param.S`** | 静态参数表 | 提供PHY/控制寄存器基础配置值 |
| **`mc_config.S`** | 主配置引擎 | 时序计算、寄存器批量写入、硬件初始化 |
| **`mc_vref_set.S`** | VREF基础设置 | 为MC0/MC1设置初始VREF值和比较器参数 |

#### 2.3 训练校准阶段
| 文件 | 功能 | 训练类型 |
|------|------|----------|
| **`ddr4_leveling.S`** | 电平校准 | 写电平校准 + 门电平校准 |
| **`mc_vref_training.S`** | VREF训练 | 扫描VREF/DLL_VREF，寻找最优参考电压 |
| **`ddr_vref_training.S`** | VREF训练(备用) | 基于MR6的VREF训练实现 |
| **`wr_bit_training.S`** | 写位训练 | 校准DQS与DQ写入时序 |
| **`rd_bit_training.S`** | 读位训练 | 优化读数据采样窗口 |

#### 2.4 测试验证阶段
| 文件 | 功能 | 测试方法 |
|------|------|----------|
| **`test_engin.S`** | 测试引擎 | 硬件加速内存测试，验证初始化结果 |

## 初始化详细流程

### 阶段一：硬件探测与参数收集 (Pre-Initialization)

#### 1.1 DIMM自动检测流程
```
detect_node_dimm_all.S (主入口)
    ↓
PROBE_NODE_DIMM()
    ├── GET_MC_ENABLE()           # 检查MC0/MC1使能状态
    ├── 探测MC0 Slot 0 → PROBE_DIMM()
    ├── 探测MC0 Slot 1 → PROBE_DIMM()
    ├── 双插槽兼容性检查
    │   ├── 类型匹配验证
    │   ├── CS映射合并
    │   └── 容量累加计算
    └── 输出s1配置字
```

**s1配置字格式（探测后）**：
```
[63:56] MC1_CS_MAP        # MC1片选映射
[55:44] MC1_MEMSIZE       # MC1内存容量(单位1GB)
[43:40] MC1_I2C_ADDR      # MC1 I2C地址
[39:32] 保留
[31:24] MC0_CS_MAP        # MC0片选映射
[23:12] MC0_MEMSIZE       # MC0内存容量(单位1GB)
[11:08] MC0_I2C_ADDR      # MC0 I2C地址
[07:04] 保留
[03:00] NODE_ID           # 节点ID
```

#### 1.2 SPD信息读取关键宏
```assembly
GET_SPD_SDRAM_TYPE_V1     # 读取内存类型 (DDR3=0x0B, DDR4=0x0C)
GET_SPD_MODULE_TYPE_V1    # 读取模块类型 (UDIMM/RDIMM/SODIMM)
GET_SPD_BG_NUM_V1         # Bank Group数量
GET_SPD_BA_NUM_V1         # Banks per Bank Group
GET_SPD_ROW_SIZE_V1       # 行地址宽度 (18-value)
GET_SPD_COL_SIZE_V1       # 列地址宽度 (12-value)
GET_SPD_CS_MAP_V1         # 物理Rank/CS映射
GET_SPD_DIMM_MEMSIZE_V1   # 计算DIMM容量
```

#### 1.3 容量计算算法
```c
// 基于JEDEC SPD规范的容量计算公式
容量 = (SDRAM密度 / 8) × (DIMM宽度 / SDRAM宽度) × Rank数 × 堆叠层数
```

#### 1.4 寄存器操作细节与算法实现

**文件**：`config_ddr_capacity_tmp.S`

**主要函数**：
- `get_dimmsize_a1()`：主入口函数，协调双控制器双插槽容量探测
- `ddr4_get_capasity()`：叶子函数，执行SPD读取和容量计算

**寄存器使用规范**：
| 寄存器 | 用途 | 调用约定 |
|--------|------|----------|
| **a0** | I2C设备地址 / 临时存储 | 参数输入，可能被修改 |
| **a1** | SPD寄存器地址 / 容量结果 | 参数输入，返回值输出 |
| **t5, t6** | 计算中间值 | 临时存储，调用者保存 |
| **s5** | 返回地址保存 | 被调用者保存 |
| **v0** | I2C读取返回值 | 函数返回值 |
| **ra** | 返回地址 | 调用者管理 |
| **t1** | s1配置字存储 | 通过宏访问插槽ID |

**内存地址映射**：
- **I2C SPD地址**：`0xa1`, `0xa3`等（通过a0寄存器传递）
- **配置寄存器**：通过t1寄存器传递内存控制器和插槽配置信息

**关键宏定义**（来自`detect_node_dimm_all.S`）：
```assembly
#define GET_MC0_SLOT0_ID dsrl a1, t1, 16; and a1, a1, 0xf
#define GET_MC0_SLOT1_ID dsrl a1, t1, 20; and a1, a1, 0xf  
#define GET_MC1_SLOT0_ID dsrl a1, t1, 24; and a1, a1, 0xf
#define GET_MC1_SLOT1_ID dsrl a1, t1, 28; and a1, a1, 0xf
```

**双控制器探测算法（C伪代码）**：
```c
// get_dimmsize_a1函数逻辑
uint64_t get_dimmsize_a1(uint64_t existing_capacity) {
    uint64_t t6 = existing_capacity;  // 高32位: MC0容量，低32位: 待累加
    
    // 处理MC0 Slot0
    slot_id = (t1 >> 16) & 0xF;  // GET_MC0_SLOT0_ID
    if (slot_id < 8) {  // 有效设备ID
        i2c_addr = (slot_id << 1) | 0xA1;
        capacity = ddr4_get_capasity(i2c_addr);
        t6_high = (t6 >> 32) + capacity;  // 累加到高32位
        t6 = (t6_high << 32) | (t6 & 0xFFFFFFFF);
    }
    
    // 处理MC0 Slot1  
    slot_id = (t1 >> 20) & 0xF;  // GET_MC0_SLOT1_ID
    if (slot_id < 8) {  // 有效设备ID
        i2c_addr = (slot_id << 1) | 0xA1;
        capacity = ddr4_get_capasity(i2c_addr);
        t6_low = (t6 & 0xFFFFFFFF) + capacity;  // 累加到低32位
        t6 = (t6 & 0xFFFFFFFF00000000) | t6_low;
    }
    
    return t6;  // 高32位: MC0 Slot0容量，低32位: MC0 Slot1容量
}
```

**容量计算算法实现（C伪代码）**：
```c
// ddr4_get_capasity函数逻辑
uint32_t ddr4_get_capasity(uint8_t i2c_addr) {
    // 1. 读取SDRAM密度（SPD寄存器4）
    density = i2cread(i2c_addr, 4) & 0xF;
    if (density == 0x1000) t5 = 0x30;      // 3Gb
    else if (density == 0x1001) t5 = 0x60; // 6Gb
    else t5 = 1 << density;                // 2^density Mb
    
    t5 = t5 >> 1;  // 转换为512Mb单位
    t5 = t5 << 3;  // 转换为64bit宽度（×8）
    
    // 2. 读取DIMM宽度（SPD寄存器12）
    dimm_width = (i2cread(i2c_addr, 12) >> 1) & 0x1;  // 0=64bit, 1=32bit
    t5 = t5 >> 3;  // 回退到SDRAM密度
    t5 = t5 >> dimm_width;  // 调整DIMM宽度
    
    // 3. 读取SDRAM宽度（SPD寄存器13）
    sdram_width = i2cread(i2c_addr, 13) & 0x1;  // 0=x8, 1=x16
    t5 = t5 << 5;  // 预调整
    t5 = t5 << sdram_width;  // 调整SDRAM宽度
    
    // 4. 读取Rank数（SPD寄存器12的bit[5:3]）
    rank_num = ((i2cread(i2c_addr, 12) >> 3) & 0x7) + 1;
    t5 = t5 * rank_num;
    
    // 5. 检查3DS堆叠（SPD寄存器6）
    if ((i2cread(i2c_addr, 6) & 0x3) == 2) {  // 3DS设备
        layers = ((i2cread(i2c_addr, 6) >> 4) & 0x7) + 1;
        t5 = t5 * layers;
    }
    
    return t5 & 0xFFFF;  // 返回256MB为单位的容量
}
```

**外部函数调用**：
- `i2cread(a0, a1)`：I2C读取函数，地址在a0，寄存器在a1，返回v0
- 调用约定：`bal i2cread`，需保存ra，结果在v0中

**插槽ID编码规则**：
- `4'b0xxx`：有效DIMM设备，xxx为I2C地址低3位
- `4'b1xxx`：无DIMM插入，跳过容量计算
- I2C完整地址 = `0xA0 | (xxx << 1) | 读位(1)` = `0xA1 | (slot_id << 1)`

**在DDR4初始化流程中的作用**：
1. **容量自动检测**：替代手动配置，支持不同容量内存条混插
2. **配置信息收集**：为`mc_config.S`等后续模块提供准确容量参数
3. **多控制器支持**：适配MC0/MC1双内存控制器架构
4. **兼容性处理**：支持DDR4各种密度、宽度、Rank和3DS堆叠配置
5. **错误容错**：检查无效设备ID（≥8），跳过无DIMM插槽

### 阶段二：寄存器基础配置 (Hardware Register Programming)

#### 2.1 静态参数加载 (`ddr4_register_param.S`)
- **DDR4参数表**：`ddr4_reg_data`, `ddr4_reg800_data`, `ddr4_ctlreg_data`
- **DDR3参数表**：`ddr3_reg_data`, `ddr3_reg800_data`, `ddr3_ctlreg_data`
- **双控制器支持**：每个参数表都有`_mc1`版本用于MC1

**关键寄存器组**：
- **PHY寄存器** (0x0000-0x0FFF)：PLL配置、DLL控制、驱动强度
- **控制寄存器** (0x1000-0x13FF)：时序参数、MR寄存器、CS映射
- **高级PHY寄存器** (0x0800-0x08FF)：阻抗校准、VREF训练参数

**寄存器参数配置细节**：

**PHY寄存器配置 (0x0000-0x0FFF)**：
```assembly
// PLL配置寄存器 (0x0030-0x003C)
.word 0x00040510  // 0x0030: PLL控制寄存器1
.word 0x00580100  // 0x0034: PLL控制寄存器2
.word 0x00000140  // 0x0038: DLL控制寄存器
.word 0x00000000  // 0x003C: 保留

// 时钟选择配置 (0x0040-0x005C)
.word 0x00000038  // 0x0040: 通道0时钟选择
.word 0x00000820  // 0x0044: 通道1时钟选择  
.word 0x00000001  // 0x0048: 通道2时钟选择
.word 0x00281140  // 0x004C: 通道3时钟选择
```

**控制寄存器配置 (0x1000-0x13FF)**：
```assembly
// 基础时序参数 (0x1000-0x101C)
.word 0x01464fc4  // 0x1000: tRP, tMOD, tXPR, tCKE, tRESET
.word 0x00221918  // 0x1004: tODTL
.word 0x0000000a  // 0x1008: tREF_IDLE, tRFC_dlr
.word 0x00000000  // 0x100C: 保留
.word 0x030000a0  // 0x1010: tREFretention, tRFC, tREF

// 读写延迟配置 (0x1060-0x1064)
.word 0x000b090c  // 0x1060: tPHY_RDLAT (DDR4) / 0x000a090c (DDR3)
.word 0x00000b0c  // 0x1064: DDR4_TPHY_WRLAT_OFFSET
```

**高级PHY寄存器配置 (0x0800-0x08FF)**：
```assembly
// 阻抗校准控制 (0x0800-0x0804)
.word 0x0026d23f  // 0x0800: PAD驱动控制基础值
#define OCD_DQS 0x6    // DQS驱动强度 (max 7)
#define OCD_DQ  0x4     // DQ驱动强度 (max 7)  
#define COMP    0x1     // 比较器使能
#define DCC     0x0     // 占空比校正器 (0=1.2GHz, 0x4=600MHz)
#define ENZI    0x0     // Z阻抗使能
.word 0x00000066  // 0x0804: OCD_DQS_V组合值

// VREF训练参数 (0x0810-0x081C)  
.word 0x00000441  // 0x0810: VREF_CTRL_DS0_OFFSET
.word 0x00000000  // 0x0814: 保留
.word 0x00000000  // 0x0818: 保留
.word 0x00000000  // 0x081C: 保留
```

**模式寄存器配置 (MR0-MR6)**：
```assembly
// DDR4模式寄存器CS0 (0x1180-0x118C)
.word 0x02010d14  // 0x1180: MR1-MR0
.word 0x02000218  // 0x1184: MR3-MR2 (800M/1G/1.2G配置)
.word 0x04000800  // 0x1188: MR5-MR4, rdpreambl2 << 11
.word 0x00000400  // 0x118C: MR6
```

**配置宏定义与算法**：

**OCD（片内驱动）配置宏**：
```c
// ddr4_register_param.S 中的宏定义
#define OCD_DQS_V (ENZI<<11)|(DCC<<7)|(COMP<<6)|(OCD_DQ<<3)|(OCD_DQS)
// 位域解析：
// [11]    ENZI: Z阻抗使能
// [10:7]  DCC: 占空比校正器控制
// [6]     COMP: 比较器使能  
// [5:3]   OCD_DQ: DQ驱动强度 (0-7)
// [2:0]   OCD_DQS: DQS驱动强度 (0-7)
```

**比较器配置宏**：
```c
#define COMP_PD 0x0       // 比较器功耗控制
#define COMP_VREFSEL 0x1  // VREF选择
#define COMP_MODE 0x1     // 比较器模式
#define COMP_PCODE 0x0    // P型代码
#define COMP_NCODE 0x1f   // N型代码 (31)
#define COMP_PD_V (COMP_NCODE<<8)|(COMP_PCODE<<3)|(COMP_MODE<<2)|(COMP_VREFSEL<<1)|(COMP_PD)
```

**时序参数转换算法**（在关联文件`loongson_mc_param.S`中）：
```c
// 时间到时钟周期转换
#define T2C(x)          (DDR_FREQ*(x)/500)  // DDR_FREQ为内存频率(MHz)
#define tRP             (T2C(15) & 0xff)    // 行预充电时间(15ns→周期数)
#define tRCD            (T2C(14) & 0x1f)    // 行到列延迟(14ns→周期数)
#define tRFC            (T2C(350) & 0xfff)  // 刷新周期(350ns→周期数)

// 示例：DDR4-2400 (1200MHz)下的tRFC计算
// tRFC = 1200 * 350 / 500 = 840 时钟周期
```

**双控制器差异化配置**：
```assembly
// MC0和MC1的参数大部分相同，但关键时序有细微差异
#define RDGATE_CTRL     0x88    // MC0读门控制
#define MC1_RDGATE_CTRL 0x00    // MC1读门控制
#define RDDQS_DLY       0x19    // MC0读DQS延迟
#define MC1_RDDQS_DLY   0x19    // MC1读DQS延迟

// 条件编译支持不同测试模式
#ifdef LPBK
.word 0x10002010  // 0x0100: 回环测试模式DLL配置
#else
.word 0x20202020  // 0x0100: 正常工作模式DLL配置
#endif
```

**参数表结构与使用**：
```c
// mc_config.S 中的使用方式
void mc_config(uint64_t *param_table_base, int channel) {
    // 批量写入744个寄存器 (0x0000-0x13FF范围)
    for(int i = 0; i < 744; i++) {
        uint64_t reg_value = param_table_base[i];
        uint64_t reg_addr = MC_BASE + (i * 8);  // 每个寄存器8字节间隔
        write_reg(reg_addr, reg_value);
    }
    
    // 根据通道选择不同参数表
    if(channel == 0) {
        // 使用ddr4_reg_data (MC0)
    } else {
        // 使用ddr4_reg_data_mc1 (MC1)
    }
}
```

**在初始化流程中的作用**：
1. **启动阶段基础配置**：由`mc_config.S`读取并批量写入内存控制器寄存器
2. **训练前准备**：提供DLL校准、阻抗设置的基础值，为`ddr4_leveling.S`动态训练奠定基础
3. **兼容性保障**：同时包含DDR4和DDR3参数，系统根据SPD检测结果自动选择
4. **硬件抽象层**：将复杂的寄存器位域配置封装为可维护的数据表格

#### 2.2 动态配置引擎 (`mc_config.S`)

**主要流程**：
```assembly
mc_config:
    ↓
批量写入参数 (744次循环)
    ↓
节点/通道识别 (GET_NODE_ID_a1)
    ↓
时序参数计算:
    ├── 时钟周期转换: 时间(ps) → 时钟周期数
    ├── SPD数据融合: 基值(125ps) + 细调值(1ps)
    ├── 整数舍入: int_rounding()函数
    └── 速度分级: 比较tCKAVG与tCKAVGmin
    ↓
DLL/PLL初始化:
    ├── 配置DLL_CTRL寄存器
    ├── 启动PHY初始化 (phy_init_start=1)
    ├── 轮询DLL锁定状态
    └── DCC(占空比校正器)控制
    ↓
DRAM初始化触发:
    ├── 设置init_start位
    └── 轮询DRAM_INIT状态
```

**关键算法**：
```assembly
// 时钟周期计算
dli t0, DDR_FREQ          # 频率值
dsll t0, t0, 1            # ×2 (DDR双倍数据率)
dli t1, 1000000
dmulou t0, t0, t1         # 转换为Hz
dli t2, 0xe8d4a51000      # 10^12 (皮秒)
ddivu t0, t2, t0          # t0 = 一个时钟周期的时间(ps)

// 时序参数转换
tRFC_clocks = int_rounding(tRFC_ps, tCK_ps)
```

**寄存器操作细节**

**MIPS寄存器使用约定**
| 寄存器 | 用途 | 说明 |
|--------|------|------|
| `t8` | 内存控制器基地址 | 所有MC/PHY寄存器访问的基址指针 |
| `t0`-`t7` | 临时计算寄存器 | 时序计算、地址偏移、临时数据 |
| `a0`-`a3` | 函数参数传递 | 调用`int_rounding`等函数时使用 |
| `v0`-`v1` | 函数返回值 | `GET_SPD`、`int_rounding`等返回值 |
| `ra` | 返回地址保存 | 由`mc_config`入口保存到`t7` |
| `k0` | 通道选择标识 | `0`=MC0，`1`=MC1 |
| `k1` | 参数表基址备份 | 保存传入的`a2`参数表地址 |

**内存控制器寄存器映射（相对于`t8`基址）**
| 偏移范围 | 功能区域 | 关键寄存器示例 |
|----------|----------|----------------|
| `0x0000`-`0x0FFF` | PHY寄存器 | `DLL_CTRL`(0x00)、`DLL_OUT`(0x0A) |
| `0x1000`-`0x13FF` | 控制寄存器 | `tRESET`(0x1000)、`tRFC`(0x1012)、`tRL`(0x1060) |
| `0x1180`-`0x11FC` | MR模式寄存器 | `MR0_CS0`(0x1180)、`MR1_CS0`(0x1182)等 |
| `0x1500`-`0x1510` | 节点窗口配置 | `0x1500`、`0x1508`、`0x1510`节点掩码 |
| `PHY_ADDRESS`(0x900) | 物理层控制 | `phy_init_start`(0x010)、DLL状态(0x030) |

**关键操作流程的C伪代码**

```c
// 1. 参数批量写入（744个64位参数）
void write_parameters(uint64_t* mc_base, uint64_t* param_table) {
    for(int i = 0; i < 744; i++) {
        mc_base[i] = param_table[i];  // 8字节对齐访问
    }
}

// 2. 时序参数计算（以tRFC为例）
uint16_t calculate_tRFC(uint64_t tCK_ps, SPD_Data* spd) {
    // 从SPD读取tRFC值（基值*125ps + 细调值*1ps）
    uint32_t tRFC_ps = spd->tRFC_base * 125;
    if (spd->tRFC_fine & 0x80) {  // 负值处理
        tRFC_ps |= 0xFFFFFF00;     // 符号扩展
    }
    tRFC_ps += spd->tRFC_fine;
    
    // 转换为时钟周期数（向上取整）
    return (uint16_t)((tRFC_ps + tCK_ps - 1) / tCK_ps);
}

// 3. DLL初始化与锁定检查
void dll_initialization(uint64_t* phy_base) {
    // 配置DLL控制寄存器
    phy_base[DLL_CTRL] = DLL_ENABLE | PREDELAY_BYPASS;
    
    // 启动PHY初始化
    phy_base[PHY_INIT_CTRL] |= PHY_INIT_START;
    
    // 轮询DLL锁定状态
    while (!(phy_base[DLL_STATUS] & DLL_LOCK_BIT)) {
        // 超时处理
    }
    
    // 检查预延迟溢出
    if (phy_base[DLL_OUT] & PREDELAY_OVERFLOW) {
        // 调整预延迟值并重试
    }
}

// 4. DRAM初始化触发与完成检查
void dram_initialization(uint64_t* ctl_base, uint64_t* phy_base, uint8_t cs_mask) {
    // 设置init_start位
    phy_base[PHY_INIT_CTRL] |= DRAM_INIT_START;
    
    // 轮询初始化完成状态
    while ((phy_base[DRAM_INIT_STATUS] & 0xFF00) >> 8 != cs_mask) {
        // 等待所有使能的CS完成初始化
    }
}
```

**配置空间的节点/通道识别**
```assembly
# 多节点系统支持：地址 = 基地址 | (节点ID << 44)
GET_NODE_ID_a1:           # 宏：将节点ID加载到a1
    andi    a1, s1, 0x3   # 从s1获取节点ID（低2位）
    dsll    a1, a1, 44    # 左移44位用于地址对齐

# 双控制器选择（k0标志位）
# k0=0: MC0（通道0），k0=1: MC1（通道1）
# 影响DBL配置、时钟相位等参数
```

#### 2.3 VREF基础设置 (`mc_vref_set.S`)
```assembly
VREF_SET(0x0801):
    ↓
打开MC0配置空间 (位4翻转)
    ↓
设置MC0使能 (mc_en=01)
    ↓
写入VREF值到三个位置
    ↓
设置比较器控制 (COMP_CTRL=0x0b08)
    ↓
关闭MC0配置空间 (位4置位)
    ↓
重复上述步骤配置MC1 (mc_en=10)
    ↓
恢复MC0为默认控制器
```

#### 2.3.1 寄存器操作细节与C伪代码分析

**寄存器使用**：
| 寄存器 | 用途 | 生存期 |
|--------|------|--------|
| `a0` | 节点ID偏移量（由`GET_NODE_ID_a0`宏计算） | 整个宏 |
| `t0` | 地址指针寄存器 | 临时计算 |
| `t1` | 数据临时寄存器 | 临时计算 |
| `t2` | 掩码/控制字寄存器 | 临时计算 |
| `t3` | VREF模式值寄存器（64位打包值） | 临时计算 |
| `s1` | 系统配置寄存器（提供节点ID，宏外部传入） | 宏外部 |

**关键内存地址**：
| 地址 | 功能 | 操作 |
|------|------|------|
| `0x900000003ff00180` | 配置空间控制寄存器 | 位4(0x10): MC0配置空间开关，位9(0x200): MC1配置空间开关 |
| `0x900000003ff00400` | 内存控制器使能寄存器 | 位30(0x40000000): MC0使能，位31(0x80000000): MC1使能 |
| `0x900000000ff00810` | VREF数据寄存器 | 存储VREF值的三个64位位置(偏移0,8,16)和比较器控制(偏移0x20) |

**地址计算**：所有地址都与节点ID(`a0`)进行`OR`操作，支持多节点系统：
- `GET_NODE_ID_a0`从`s1`提取低2位节点ID，左移44位
- 最终地址 = 基地址 | (节点ID << 44)

##### 配置空间控制机制
- **打开配置空间**：使用`ORI`+`XORI`序列**翻转特定位**（位4对应MC0，位9对应MC1）
- **关闭配置空间**：直接使用`ORI`**置位**相应位
- 这种设计确保在修改关键寄存器前正确访问配置空间

##### 内存控制器选择逻辑
- **MC0选择**：`mc_en` = `01` (位30=1, 位31=0)
  ```asm
  lui t2, 0x4000; or t1, t2;    # 置位30
  lui t2, 0x8000; or t1, t2;    # 置位31
  xor t1, t2;                    # 清除位31 → 01
  ```
- **MC1选择**：`mc_en` = `10` (位30=0, 位31=1)
  ```asm
  lui t2, 0x8000; or t1, t2;    # 置位31
  lui t2, 0x4000; or t1, t2;    # 置位30
  xor t1, t2;                    # 清除位30 → 10
  ```

##### VREF值格式
- **16位VREF值**：宏参数`VREF`（如`0x0801`）
- **64位打包格式**：`(VREF<<48) | (VREF<<32) | (VREF<<16) | VREF`
- **存储位置**：写入三个连续64位寄存器（可能对应不同通道或rank）

##### 比较器控制常量`COMP_CTRL`
```c
#define COMPVREF_IN  1  // 1:输入模式
#define COMP_MANUAL 0   // 0:自动模式
#define COMP_PD     0   // 不休眠
#define COMP_PCODE  0x8 // P码（较小值，电阻较小）
#define COMP_NCODE  0xb // N码（较小值，电阻较大）
#define COMP_CTRL   ((COMP_NCODE<<8)|(COMP_PCODE<<3)|(COMP_MANUAL<<2)|(COMPVREF_IN<<1)|(COMP_PD))
```
- **0x0b08**：最终控制字（NCODE=0xb, PCODE=0x8）
- 控制VREF训练使用的比较器参数

##### C伪代码解释完整逻辑
```c
// mc_vref_set宏的C伪代码实现
void vref_set(uint16_t vref_value) {
    // 1. 节点识别
    uint64_t node_id = (s1 & 0x3) << 44;  // GET_NODE_ID_a0
    
    // 2. MC0配置
    // 打开MC0配置空间
    uint32_t* conf_space_ctrl = (uint32_t*)(0x900000003ff00180 | node_id);
    *conf_space_ctrl ^= 0x10;  // 翻转位4
    
    // 选择MC0控制器 (mc_en=01)
    uint32_t* mc_enable = (uint32_t*)(0x900000003ff00400 | node_id);
    *mc_enable = (*mc_enable | 0x40000000) & ~0x80000000;  // 位30=1, 位31=0
    
    // 写入VREF值到三个位置
    uint64_t vref_packed = ((uint64_t)vref_value << 48) | 
                          ((uint64_t)vref_value << 32) | 
                          ((uint64_t)vref_value << 16) | 
                          vref_value;
    uint64_t* vref_reg = (uint64_t*)(0x900000000ff00810 | node_id);
    vref_reg[0] = vref_packed;  // 偏移0
    vref_reg[1] = vref_packed;  // 偏移8
    vref_reg[2] = vref_packed;  // 偏移16
    
    // 设置比较器控制
    vref_reg[4] = COMP_CTRL;  // 偏移0x20
    
    // 关闭MC0配置空间
    *conf_space_ctrl |= 0x10;  // 置位位4
    
    // 3. MC1配置
    // 打开MC1配置空间
    *conf_space_ctrl ^= 0x200;  // 翻转位9
    
    // 选择MC1控制器 (mc_en=10)
    *mc_enable = (*mc_enable | 0x80000000) & ~0x40000000;  // 位31=1, 位30=0
    
    // 写入相同的VREF值和比较器控制
    vref_reg[0] = vref_packed;
    vref_reg[1] = vref_packed;
    vref_reg[2] = vref_packed;
    vref_reg[4] = COMP_CTRL;
    
    // 关闭MC1配置空间
    *conf_space_ctrl |= 0x200;  // 置位位9
    
    // 4. 恢复默认控制器为MC0
    *mc_enable = (*mc_enable | 0x40000000) & ~0x80000000;  // mc_en=01
}
```

##### 算法步骤总结
1. **节点识别**：计算当前节点的地址偏移
2. **MC0配置**：
   - 打开MC0配置空间（翻转位4）
   - 选择MC0控制器（`mc_en=01`）
   - 写入VREF值到三个位置
   - 设置比较器控制寄存器（`COMP_CTRL`）
   - 关闭MC0配置空间（置位位4）
3. **MC1配置**：
   - 打开MC1配置空间（翻转位9）
   - 选择MC1控制器（`mc_en=10`）
   - 写入相同的VREF值
   - 设置相同的比较器控制
   - 关闭MC1配置空间（置位位9）
4. **恢复默认**：重新选择MC0为默认控制器

##### 设计特点
- **原子性操作**：每个配置步骤都包含完整的读-修改-写序列
- **错误恢复**：通过位翻转确保配置空间状态正确
- **对称设计**：MC0和MC1使用相同的配置流程
- **默认状态恢复**：最终恢复MC0为活动控制器

##### 在DDR4初始化流程中的作用
该宏通常在以下位置被调用（见`Targets/Bonito3a4000_7a/Bonito/start.S`）：
```asm
#define VREF 0x0801  // DDR4默认值
VREF_SET(VREF)
```
- **前置条件**：内存控制器基本时钟和电源已初始化
- **后置操作**：通常跟随VREF训练(`mc_vref_training.S`)
- **目的**：提供初始VREF值，为后续的VREF训练建立基础

### 阶段三：时序训练校准 (Timing Training)

#### 3.1 写电平校准 (`ddr4_leveling.S` - 写部分)

**校准目标**：调整DQS与DQ写入时序关系

**流程**：
```
write_leveling_new:
    ↓
配置阶段:
    ├── 关闭非校准CS的DQ输出 (MR1[12]=1)
    ├── 使能写电平模式 (MR1[7]=1, RDIMM需要)
    └── 设置LVL_MODE=1
    ↓
校准循环(每个数据片):
    ├── 发起电平请求 (LVL_REQ[8]=1)
    ├── 等待电平完成 (轮询LVL_DONE[8])
    ├── 检查响应位 (LVL_RESP每个数据片1位)
    ├── 阶段A(找0): 递增DLL_WRDQ直到所有响应为0
    ├── 阶段B(找1): 继续递增直到所有响应为1
    └── 记录跳变中间点
    ↓
结果应用:
    ├── 计算最佳DLL_WRDQ = (找到1时的值 - CHECKCOUNT/2)
    ├── 设置DLL_1XDLY为相同值
    └── 计算DLY_2X补偿值
```

#### 3.2 门电平校准 (`ddr4_leveling.S` - 读部分)

**校准目标**：优化读数据采样窗口

**流程**：
```
gate_leveling_new:
    ↓
初始化:
    ├── 使能MPR模式 (MR3[2]=1)
    ├── 使能读前导训练 (MR4[10]=1)
    └── 设置LVL_MODE=2
    ↓
扫描寻找稳定窗口:
    ├── 固定RDDATA值，扫描DLL_GATE (0x00~0x7F)
    ├── 寻找连续CONTINUE_VALUE(0x20)个响应为1的区域
    └── 记录(起始DLL_GATE, RDDATA)对
    ↓
计算最终值:
    ├── 取所有数据片中最小的RDDATA作为基准
    ├── 调整DLL_GATE: final_gate = 找到的gate - 0x40
    └── 计算DLY_2X补偿: 根据(数据片RDDATA - 基准RDDATA)
```

#### 3.3 VREF训练 (`mc_vref_training.S`)

**二维参数空间扫描**：
```
mc_vref_training:
    ↓
VREF值扫描 (0x00-0x7f, 128个点)
    ↓
DLL VREF值扫描 (0x00-0x7f, 128个点)
    ↓
五样本采集策略:
    ├── 采集5个独立样本(SAMPLE0-4)
    ├── 或操作统计: ONE_SAMPLE = SAMPLE0 | SAMPLE1 | ...
    ├── 与操作统计: ZERO_SAMPLE = SAMPLE0 & SAMPLE1 & ...
    └── 连续窗口检测: 连续40个1和连续40个0
    ↓
通过/失败判定:
    ├── 通过: 同时找到连续40个1和连续40个0
    ├── 失败: 任一连续性检查失败
    └── 结果存储: 位图记录每个VREF值的通过状态
    ↓
VREF延迟优化:
    ├── 初始VREF_DLY = 0x5
    ├── 根据训练结果质量动态调整
    └── 最大VREF_DLY限制为0x20
    ↓
读延迟(RL)同步优化:
    ├── 测试5个连续RL值 (RL, RL+1, RL+2, RL+3, RL+4)
     └── 选择具有最佳训练结果的RL值
```

#### 寄存器操作细节

##### 关键寄存器偏移（VREF训练专用）
| 偏移量 | 绝对地址 | 功能描述 |
|--------|----------|----------|
| `ONE_SAMPLE_L64` | 0x3300 | 所有样本的或操作结果（低64位） |
| `ONE_SAMPLE_H64` | 0x3308 | 所有样本的或操作结果（高64位） |
| `ZERO_SAMPLE_L64` | 0x3310 | 所有样本的与操作结果（低64位） |
| `ZERO_SAMPLE_H64` | 0x3318 | 所有样本的与操作结果（高64位） |
| `SAMPLE0_L64` | 0x3320 | 样本0数据（低64位） |
| `SAMPLE0_H64` | 0x3328 | 样本0数据（高64位） |
| `SAMPLE1_L64` | 0x3330 | 样本1数据（低64位） |
| `SAMPLE1_H64` | 0x3338 | 样本1数据（高64位） |
| `SAMPLE2_L64` | 0x3340 | 样本2数据（低64位） |
| `SAMPLE2_H64` | 0x3348 | 样本2数据（高64位） |
| `SAMPLE3_L64` | 0x3350 | 样本3数据（低64位） |
| `SAMPLE3_H64` | 0x3358 | 样本3数据（高64位） |
| `SAMPLE4_L64` | 0x3360 | 样本4数据（低64位） |
| `SAMPLE4_H64` | 0x3368 | 样本4数据（高64位） |
| `REG_COUNT` | 0x3370 | DLL循环计数器（0-4，对应5个样本） |
| `REG_SHIFT` | 0x3371 | 位偏移计数器（0-7，对应8位测试） |
| `REG_VREF_COUNT` | 0x3372 | VREF循环计数器（0-1，共2次循环） |
| `SLICE_NUMBER` | 0x3373 | 数据切片总数（2/4/8/9） |
| `RDDATA_DEFAULT` | 0x3374 | 默认读数据延迟值 |
| `VREF_FLAG` | 0x3375 | VREF训练状态标志（0=初始，1=已优化） |
| `RL_CURRENT` | 0x3376 | 当前读延迟（RL）值 |
| `RL_FLAG` | 0x3377 | RL训练状态标志（0=未完成，1=已完成） |
| `RL_1`-`RL_5` | 0x3378-0x337c | RL训练结果的5个连续值存储 |
| `FIND_RL_FLAG` | 0x337d | RL查找标志（计数器1-5） |
| `VREF_RESULT_MAX` | 0x337e | ODT训练中最大的通过位数 |
| `VREF_VS_ODT_MAX` | 0x337f | ODT训练最佳ODT值（位[2:0]）+完成标志（位3） |

##### 内存控制器寄存器偏移（来自`ls3a4000_reg_def.h`）
| 偏移量 | 符号名 | 功能描述 |
|--------|--------|----------|
| 0x0700 | `LVL_MODE_OFFSET` | 电平模式控制寄存器 |
| 0x0701 | `LVL_REQ_OFFSET` | 电平请求寄存器 |
| 0x0708 | `LVL_RDY_OFFSET` | 电平就绪状态寄存器 |
| 0x0709 | `LVL_DONE_OFFSET` | 电平完成状态寄存器 |
| 0x1062 | `TPHY_RDDATA_OFFSET` | PHY读数据延迟寄存器 |
| 0x106? | `VREF_DLY_OFFSET` | VREF延迟控制寄存器（内部定义） |
| 0x108? | `DLL_VREF_OFFSET` | DLL VREF控制寄存器（内部定义） |
| 0x108? | `VREF_SAMPLE_OFFSET` | VREF采样数据寄存器（内部定义） |
| 0x0800-0x080F | `DS0_ODT_OFFSET`等 | 数据片0-8的ODT控制寄存器 |

##### 训练结果存储区
| 地址偏移 | 符号名 | 功能描述 |
|----------|--------|----------|
| 0x1140 | `VREF_RESULT_L64` | VREF训练结果低64位（每比特代表一个VREF值通过与否） |
| 0x1148 | `VREF_RESULT_H64` | VREF训练结果高64位 |
| 0x1150 | `VREF_COMMON_L64` | 所有切片公共VREF结果低64位（位与结果） |
| 0x1158 | `VREF_COMMON_H64` | 所有切片公共VREF结果高64位 |
| 0x1160 | `RA_STORE` | 返回地址保存寄存器 |

##### 寄存器配置操作细节

**1. 节点基址计算**：
```assembly
GET_NODE_ID_a0                 # 获取节点ID到a0寄存器
dli    conf_base, 0x900000000ff00000  # 内存控制器基址
or     conf_base, a0           # conf_base = 基址 | (节点ID << 44)
```

**2. 电平模式配置（门电平校准）**：
```assembly
li     t0, 0x2                 # 模式2：门电平校准
sb     t0, LVL_MODE_OFFSET(conf_base)  # 设置LVL_MODE
WAIT_FOR(20000)                # 等待20,000个时钟周期

# 等待电平就绪
1:
lb     t0, LVL_RDY_OFFSET(conf_base)  # 读取就绪状态
beqz   t0, 1b                  # 未就绪则循环等待
nop
```

**3. 切片数量确定算法**：
```c
// C伪代码解释切片数量计算逻辑
uint8_t determine_slice_number(uint8_t dimm_width, bool ecc_enabled) {
    uint8_t slices = 0;
    switch (dimm_width) {
        case 1:  // x4 DIMM
            slices = 2;
            break;
        case 2:  // x8 DIMM  
            slices = 4;
            break;
        case 3:  // x16 DIMM
            slices = 8;
            break;
    }
    
    if (ecc_enabled) {
        slices += 1;  // 增加ECC切片
    }
    
    return slices;
}
```

**4. MPR模式配置（针对RDIMM）**：
```assembly
# 启用MPR模式
GET_LVL_CS_NUM                 # 获取电平校准的CS数量
move   mrs_cs, v0              # CS编号
dsll   t1, v0, 4               # 计算MR寄存器偏移（每个CS 16字节）
daddu  t1, conf_base           # t1 = 基址 + 偏移
lh     mrs_cmd_a, DDR4_MR3_CS0_REG(t1)  # 读取MR3当前值
or     mrs_cmd_a, (1<<2)       # 设置位2（MPR使能）
and    mrs_cmd_a, ~(0x3 | 0x3<<11)  # 清零位[1:0]和位[12:11]
li     mrs_num, 3              # MR3编号
MRS_SEND(mrs_cmd_a,mrs_cs,mrs_num)  # 发送MRS命令
```

**5. ODT值初始化与扫描**：
```assembly
# ODT值初始化
dsll   t0, num_slice, 1        # t0 = 切片编号 * 2（每个切片2字节）
daddu  t0, conf_base           # t0 = 目标寄存器地址
lhu    t1, DS0_ODT_OFFSET(t0)  # 读取当前ODT值
dli    t2, 0x7                 # ODT值掩码（3位）
dsll   t2, 9                   # 移动到ODT位字段位置（位9-11）
not    t2                      # 取反得到清除掩码
and    t1, t2                  # 清除当前ODT值
dli    t2, 0x1                 # 初始ODT值 = 1
dsll   t2, 9                   # 移动到ODT位字段
or     t1, t2                  # 设置新的ODT值
sh     t1, DS0_ODT_OFFSET(t0)  # 写回寄存器

# ODT扫描循环（0x0-0x7，共8个值）
odt_loop:
dli    t0, 0
sb     t0, VREF_RESULT_MAX(conf_base)  # 初始化最大通过位数
sb     t0, VREF_VS_ODT_MAX(conf_base)  # 初始化最佳ODT值

# 每个ODT值执行VREF训练
bal    vref_train_kernal       # 调用VREF训练核心函数
nop
```

**6. VREF值设置寄存器操作**：
```assembly
# 设置当前测试的VREF值
dsll   t4, num_slice, 1        # t4 = 切片编号 * 2
daddu  t4, conf_base           # t4 = 目标寄存器地址
lhu    t1, 0x810(t4)           # 读取VREF控制寄存器
li     t0, (0x7f<<5)           # VREF值掩码（7位，位5-11）
not    t0                      # 取反得到清除掩码
and    t1, t0                  # 清除当前VREF值
ori    t1, 0x1                 # 设置使能位（位0）
dsll   t0, vref_value, 5       # 将VREF值移动到正确位置（位5-11）
or     t1, t0                  # 设置新的VREF值
sh     t1, 0x810(t4)           # 写回寄存器
```

**7. DLL VREF值设置与同步**：
```assembly
dli    t0, 0x7f                # DLL VREF最大值（127）
and    dll_vref, t0            # 确保值在0-127范围内
sb     dll_vref, DLL_VREF_OFFSET(conf_base)  # 写入DLL VREF寄存器

# 同步等待（当DLL VREF=0时需要额外等待）
bnez   dll_vref, 2f
nop
dli    t0, WAIT_NUMBER         # WAIT_NUMBER = 0x200
1:
dsubu  t0, 1
bnez   t0, 1b                  # 延迟等待
nop
2:

# 硬件同步
lbu    t0, DLL_VREF_OFFSET(conf_base)  # 读取两次确保硬件同步
lbu    t0, DLL_VREF_OFFSET(conf_base)
```

**8. 电平请求与完成检测**：
```assembly
# 发起电平请求
li     t0, 0x1                 # 请求位
sb     t0, LVL_REQ_OFFSET(conf_base)  # 设置LVL_REQ
sb     $0, LVL_REQ_OFFSET(conf_base)  # 清除LVL_REQ（脉冲式请求）

# 等待电平完成
1:
lbu    t0, LVL_DONE_OFFSET(conf_base)  # 读取完成状态
beqz   t0, 1b                  # 未完成则循环等待
nop
```

**9. 采样数据读取与存储**：
```assembly
# 读取VREF采样数据
lbu    t0, VREF_SAMPLE_OFFSET(conf_base)  # 读取采样数据

# 根据DLL_VREF值决定存储位置（低64位或高64位）
li     t1, 0x40                # 分界值（64）
bge    dll_vref, t1, set_h64   # 如果≥64，存储到高64位
nop

# 存储到低64位（SAMPLEx_L64）
move   t3, dll_vref            # t3 = DLL_VREF值（0-63）
lbu    t4, REG_SHIFT(conf_base) # 获取当前位偏移
li     t1, 0x1
dsrl   t0, t4                  # 右移到位偏移位置
and    t1, t0                  # 提取当前位的值
dsll   t1, t3                  # 左移到DLL_VREF对应位置

# 根据REG_COUNT（样本索引）存储到对应样本寄存器
lbu    t4, REG_COUNT(conf_base)
beqz   t4, store_sample0
nop
# ... 类似逻辑处理SAMPLE1-4

store_sample0:
ld     t2, SAMPLE0_L64(conf_base)  # 读取现有结果
or     t2, t1                      # 设置对应位
sd     t2, SAMPLE0_L64(conf_base)  # 写回结果
```

**10. 统计结果计算（OR和AND操作）**：
```assembly
# 计算所有样本的OR结果
ld     t0, ONE_SAMPLE_L64(conf_base)  # 读取现有OR结果
ld     t1, ZERO_SAMPLE_L64(conf_base) # 读取现有AND结果
lbu    t4, REG_COUNT(conf_base)       # 获取当前样本索引

# 根据样本索引加载对应样本
beqz   t4, load_sample0
nop
# ... 类似逻辑处理SAMPLE1-4

load_sample0:
ld     t2, SAMPLE0_L64(conf_base)     # 加载样本0数据
or     t0, t0, t2                     # OR操作：ONE_SAMPLE |= SAMPLEx
and    t1, t1, t2                     # AND操作：ZERO_SAMPLE &= SAMPLEx
sd     t0, ONE_SAMPLE_L64(conf_base)  # 保存OR结果
sd     t1, ZERO_SAMPLE_L64(conf_base) # 保存AND结果
```

**11. 连续性检查算法寄存器操作**：
```c
// C伪代码解释连续性检查算法
int check_continuity(uint64_t one_sample, uint64_t zero_sample, int threshold) {
    // 查找连续threshold个1
    int found_cont1 = 0;
    for (int i = 0; i < 128 - threshold; i++) {
        uint64_t mask = ((1ULL << threshold) - 1) << i;
        if ((one_sample & mask) == mask) {
            found_cont1 = 1;
            break;
        }
    }
    
    if (!found_cont1) return 0;
    
    // 查找连续threshold个0
    int found_cont0 = 0;
    for (int i = 0; i < 128 - threshold; i++) {
        uint64_t mask = ((1ULL << threshold) - 1) << i;
        if ((zero_sample & mask) == 0) {
            found_cont0 = 1;
            break;
        }
    }
    
    return found_cont0;
}
```

**12. VREF训练结果存储**：
```assembly
# 将当前VREF值的通过状态存储到位图中
li     t0, 0x40                # 分界值（64）
blt    vref_value, t0, store_low
nop

# VREF值>=64，存储到高64位结果
subu   t0, vref_value, t0      # t0 = vref_value - 64
li     t1, 0x1
dsll   t1, t0                  # t1 = 1 << (vref_value-64)
or     s6, t1                  # s6高64位对应位置1
b      2f
nop

store_low:
# VREF值<64，存储到低64位结果
li     t0, 0x1
dsll   t0, vref_value          # t0 = 1 << vref_value
or     s5, t0                  # s5低64位对应位置1

2:
# 合并到总结果中（位与操作，只有所有位都为1才表示通过）
ld     t0, 0x1140(conf_base)   # 读取现有总结果（低64位）
and    t0, s5                  # 位与操作
sd     t0, 0x1140(conf_base)   # 保存更新后的结果
```

**13. VREF延迟优化逻辑**：
```assembly
# 统计当前结果中1的数量（通过位数）
dli    t4, 0                   # 计数器
dli    t1, 0                   # 位索引
dli    t2, 0x1                 # 位掩码
1:
ld     t0, 0x1140(conf_base)   # 读取低64位结果
and    t3, t0, t2              # 提取当前位
dsrl   t3, t1                  # 右移到最低位
daddu  t4, t3                  # 计数器累加
ld     t0, 0x1148(conf_base)   # 读取高64位结果
and    t3, t0, t2              # 提取当前位
dsrl   t3, t1                  # 右移到最低位
daddu  t4, t3                  # 计数器累加
daddu  t1, 1                   # 位索引加1
dsll   t2, 1                   # 位掩码左移1位
bleu   t1, 63, 1b              # 循环64次
nop

# 根据通过位数调整VREF_DLY
lb     t0, VREF_FLAG(conf_base)
beq    t0, 0x1, rl_training    # 如果已优化过，进入RL训练
nop

bgtu   t4, 0xa, increase_vref_dly  # 如果通过位数>10，增加VREF_DLY
nop

# VREF_DLY递增
lb     t0, VREF_DLY_OFFSET(conf_base)
daddu  t0, 1                   # VREF_DLY加1
bgeu   t0, 0x20, vref_dly_error  # 如果达到最大值0x20，报错
nop
sb     t0, VREF_DLY_OFFSET(conf_base)
b      continue_training
nop

increase_vref_dly:
# 找到合适的VREF_DLY，增加2个NCK延迟
li     t0, 0x1
sb     t0, VREF_FLAG(conf_base)  # 设置VREF_FLAG=1
lb     t0, VREF_DLY_OFFSET(conf_base)
daddu  t0, 0x2                  # VREF_DLY加2
sb     t0, VREF_DLY_OFFSET(conf_base)
```

**14. 读延迟（RL）训练寄存器操作**：
```assembly
# RL训练：测试5个连续的RL值
lb     t0, FIND_RL_FLAG(conf_base)
daddu  t1, t0, 0x1              # 计数器加1
sb     t1, FIND_RL_FLAG(conf_base)

# 存储当前RL的训练结果（通过位数）
beq    t1, 0x1, store_rl1
nop
beq    t1, 0x2, store_rl2
nop
beq    t1, 0x3, store_rl3
nop
beq    t1, 0x4, store_rl4
nop
beq    t1, 0x5, store_rl5
nop

store_rl1:
sb     t4, RL_1(conf_base)      # 存储RL的训练结果（通过位数）
b      continue_rl_training

# 当收集完5个RL值后，选择最优的RL
lb     t0, RL_5(conf_base)
lb     t1, RL_1(conf_base)
bgtu   t0, t1, rl_found         # 如果RL+4的结果比RL好，说明趋势正确
nop

# 找到最佳RL值（中间值）
lb     t0, TPHY_RDDATA_OFFSET(conf_base)
subu   t0, t0, 0x3              # 回退3个RDDATA值
sb     t0, TPHY_RDDATA_OFFSET(conf_base)
lb     t0, RL_CURRENT(conf_base)
subu   t0, t0, 0x3              # 回退3个RL值
sb     t0, RL_CURRENT(conf_base)

rl_found:
li     t1, 0x1
sb     t1, RL_FLAG(conf_base)   # 设置RL_FLAG=1，表示RL训练完成
```

**15. 最终VREF值计算算法**：
```c
// C伪代码解释最终VREF值计算
uint8_t calculate_final_vref(uint64_t vref_result_low, uint64_t vref_result_high) {
    // 1. 寻找最大的连续通过区域
    uint64_t combined = (vref_result_high << 64) | vref_result_low;
    int start = 0, end = 0, max_length = 0;
    int current_start = 0, current_length = 0;
    
    for (int i = 0; i < 128; i++) {
        if ((combined >> i) & 0x1) {
            if (current_length == 0) {
                current_start = i;
            }
            current_length++;
        } else {
            if (current_length > max_length) {
                max_length = current_length;
                start = current_start;
                end = i - 1;
            }
            current_length = 0;
        }
    }
    
    // 2. 计算中间位置作为最终VREF值
    int middle = start + (max_length / 2);
    
    // 3. 边界处理：如果没有有效窗口，使用默认值
    if (max_length == 0) {
        return VREF_DEFAULT >> 5;  // 默认VREF值（右移5位）
    }
    
    return middle;
}
```

**16. 最佳VREF值写入所有切片**：
```assembly
# 单个切片VREF设置
dsll   t4, num_slice, 1        # t4 = 切片编号 * 2
daddu  t4, conf_base           # t4 = 目标寄存器地址
lhu    t1, 0x810(t4)           # 读取当前VREF控制寄存器
li     t0, (0x7f<<5)           # VREF值掩码
not    t0                      # 取反得到清除掩码
and    t1, t0                  # 清除当前VREF值
ori    t1, 0x1                 # 设置使能位（位0）
dsll   t0, vref_value, 5       # 将最终VREF值移动到正确位置
or     t1, t0                  # 设置新的VREF值
sh     t1, 0x810(t4)           # 写回寄存器

# 所有切片统一设置（使用广播模式）
dli    t4, 0x900004000ff00000  # 第一个数据片基址
lhu    t1, 0x810(t4)           # 读取寄存器
li     t0, (0x7f<<5)
not    t0
and    t1, t0
ori    t1, 0x1
dsll   t0, vref_value, 5
or     t1, t0
sh     t1, 0x810(t4)           # 数据片0
sh     t1, 0x812(t4)           # 数据片1
sh     t1, 0x814(t4)           # 数据片2
sh     t1, 0x816(t4)           # 数据片3
sh     t1, 0x818(t4)           # 数据片4
sh     t1, 0x81a(t4)           # 数据片5
sh     t1, 0x81c(t4)           # 数据片6
sh     t1, 0x81e(t4)           # 数据片7
sh     t1, 0x820(t4)           # 数据片8（ECC）
```

##### 核心算法C伪代码解释

**1. 五样本统计策略**：
```c
// 采集5个独立样本并统计分析
void five_sample_statistics(uint64_t samples[5], uint64_t* one_sample, uint64_t* zero_sample) {
    *one_sample = samples[0] | samples[1] | samples[2] | samples[3] | samples[4];
    *zero_sample = samples[0] & samples[1] & samples[2] & samples[3] & samples[4];
}

// 连续性检查：寻找连续CONT_NUM个相同比特
int check_continuity(uint64_t bitmap, int cont_num, int expected_value) {
    uint64_t mask = (1ULL << cont_num) - 1;
    
    for (int i = 0; i <= 128 - cont_num; i++) {
        uint64_t window = (bitmap >> i) & mask;
        if (expected_value == 1) {
            if (window == mask) return 1;  // 找到连续cont_num个1
        } else {
            if (window == 0) return 1;     // 找到连续cont_num个0
        }
    }
    
    return 0;  // 未找到
}
```

**2. VREF训练主算法**：
```c
#define VREF_RANGE 128     // VREF值范围：0-127
#define DLL_VREF_RANGE 128 // DLL VREF值范围：0-127
#define SAMPLE_COUNT 5     // 样本数量
#define CONT1_NUM 40       // 连续1阈值
#define CONT0_NUM 40       // 连续0阈值

uint64_t vref_training(uint8_t dataslice) {
    uint64_t result_low = 0;   // VREF 0-63的结果
    uint64_t result_high = 0;  // VREF 64-127的结果
    
    for (int vref = 0; vref < VREF_RANGE; vref++) {
        int pass = 0;
        
        // 设置当前VREF值
        set_vref_register(dataslice, vref);
        
        for (int dll_vref = 0; dll_vref < DLL_VREF_RANGE; dll_vref++) {
            // 设置DLL VREF值
            set_dll_vref_register(dll_vref);
            
            // 采集5个样本
            uint64_t samples[SAMPLE_COUNT];
            for (int sample = 0; sample < SAMPLE_COUNT; sample++) {
                samples[sample] = capture_sample(dataslice, dll_vref);
            }
            
            // 统计分析
            uint64_t one_sample, zero_sample;
            five_sample_statistics(samples, &one_sample, &zero_sample);
            
            // 连续性检查
            int has_cont1 = check_continuity(one_sample, CONT1_NUM, 1);
            int has_cont0 = check_continuity(zero_sample, CONT0_NUM, 0);
            
            // 如果同时存在连续CONT1_NUM个1和CONT0_NUM个0，则通过
            if (has_cont1 && has_cont0) {
                pass = 1;
                break;  // 只要有一个DLL VREF值通过，该VREF值就通过
            }
        }
        
        // 记录该VREF值的通过状态
        if (pass) {
            if (vref < 64) {
                result_low |= (1ULL << vref);
            } else {
                result_high |= (1ULL << (vref - 64));
            }
        }
    }
    
    return (result_high << 64) | result_low;
}
```

**3. ODT训练算法**：
```c
uint8_t odt_training(uint8_t dataslice) {
    int max_pass_bits = 0;
    uint8_t best_odt = 0;
    
    // 测试8个ODT值（0-7）
    for (int odt = 0; odt < 8; odt++) {
        // 设置ODT值
        set_odt_register(dataslice, odt);
        
        // 执行VREF训练
        uint64_t vref_result = vref_training(dataslice);
        
        // 统计通过位数
        int pass_bits = count_bits(vref_result);
        
        // 更新最佳ODT值
        if (pass_bits > max_pass_bits) {
            max_pass_bits = pass_bits;
            best_odt = odt;
        }
    }
    
    // 设置最佳ODT值
    set_odt_register(dataslice, best_odt);
    
    return best_odt;
}
```

**4. 最终VREF选择算法**：
```c
uint8_t select_final_vref(uint64_t* slice_results, int num_slices) {
    // 计算所有切片的公共VREF（位与操作）
    uint64_t common_low = 0xFFFFFFFFFFFFFFFF;
    uint64_t common_high = 0xFFFFFFFFFFFFFFFF;
    
    for (int i = 0; i < num_slices; i++) {
        common_low &= (slice_results[i] & 0xFFFFFFFFFFFFFFFF);
        common_high &= (slice_results[i] >> 64);
    }
    
    // 寻找最大的连续通过窗口
    uint64_t combined = (common_high << 64) | common_low;
    int start = 0, max_length = 0;
    int current_start = 0, current_length = 0;
    
    for (int i = 0; i < 128; i++) {
        if ((combined >> i) & 0x1) {
            if (current_length == 0) current_start = i;
            current_length++;
        } else {
            if (current_length > max_length) {
                max_length = current_length;
                start = current_start;
            }
            current_length = 0;
        }
    }
    
    // 处理最后一个窗口
    if (current_length > max_length) {
        max_length = current_length;
        start = current_start;
    }
    
    // 如果没有有效窗口，使用默认值
    if (max_length == 0) {
        return DEFAULT_VREF >> 5;
    }
    
    // 返回窗口中间位置作为最终VREF
    return start + (max_length / 2);
}
```

##### 配置参数
| 参数名 | 值 | 描述 |
|--------|----|------|
| `CONT1_NUM` | 0x28 (40) | 连续1的阈值，模拟"数据眼图"的开启状态 |
| `CONT0_NUM` | 0x28 (40) | 连续0的阈值，模拟"数据眼图"的关闭状态 |
| `DLL_LOOP_COUNT` | 0x5 | DLL循环次数（5个样本） |
| `BIT_LOOP_CONTROL` | 0x8 | 位循环控制（8位测试） |
| `VREF_LOOP_COUNT` | 0x2 | VREF循环次数（2次，用于平均） |
| `ODT_LOOP_CTRL` | 0x8 | ODT循环控制（8个ODT值） |
| `WAIT_NUMBER` | 0x200 | 硬件同步等待循环计数 |
| `VREF_TRAINING_DEBUG` | 定义 | 控制VREF训练调试信息输出 |
| `PRINT_DLL_SAMPLE` | 定义 | 控制DLL样本调试信息输出 |
| `DISABLE_DQ_ODT_TRAINING` | 定义 | 禁用DQ ODT训练 |

##### 寄存器使用约定
| 寄存器 | 用途 | 生存期 |
|--------|------|--------|
| `t8` (`conf_base`) | 内存控制器寄存器基址指针 | 整个函数 |
| `t7` (`vref_value`) | 当前VREF值（0-127） | VREF循环 |
| `t6` (`dll_vref`) | 当前DLL VREF值（0-127） | DLL VREF循环 |
| `t5` (`num_slice`) | 当前数据切片编号 | 切片循环 |
| `s5` | VREF训练结果低64位存储 | VREF训练核心 |
| `s6` | VREF训练结果高64位存储 | VREF训练核心 |
| `t9` | 保存返回地址（主函数） | 主函数 |
| `ra` | 返回地址（子函数） | 子函数调用 |
| `v0`, `v1` | 函数返回值 | 临时 |
| `a0`, `a1` | 参数传递和调试输出 | 临时 |

**基址计算**：
```assembly
# 节点基址计算
GET_NODE_ID_a0                 # 从s1[1:0]提取节点ID
dli    t8, 0x900000000ff00000  # 内存控制器基址
or     t8, a0                  # 加上节点偏移：t8 = 基址 | (节点ID << 44)
```

##### 错误处理寄存器操作

**1. VREF训练失败处理**：
```assembly
vref_dly_error:
#ifdef VREF_TRAINING_DEBUG
    PRINTSTR("\r\nVref_dly training ERROR!!!!!!!!!!!!!!!!!!***********************************\n")
#endif
    b       out_vref_dly_loop3
    nop
```

**2. RL训练失败处理**：
```assembly
rl_error:
    PRINTSTR("Training RL ERROR!!!!!***********************************************************************\r\n")
    b       out_vref_dly_loop2
    nop
```

**3. ODT值越界处理**：
```assembly
bdly_error:
    PRINTSTR("ERROR: ODT value exceed max value\r\n")
    # 使用默认ODT值继续
```

##### 调试寄存器操作

**1. 训练进度输出**：
```assembly
#ifdef VREF_TRAINING_DEBUG
    PRINTSTR("\r\nslice:")
    move    a0, num_slice
    bal     hexserial
    nop
#endif
```

**2. VREF值调试输出**：
```assembly
#ifdef VREF_TRAINING_DEBUG
    PRINTSTR("Vref_value is :")
    dsrl    a0, vref_value, 32
    bal     hexserial
    nop
    move    a0, vref_value
    bal     hexserial
    nop
    PRINTSTR("\r\n")
#endif
```

**3. 训练结果输出**：
```assembly
#ifdef VREF_TRAINING_DEBUG
    PRINTSTR("\r\nresult:          ")
    dsrl    a0, s6, 32
    bal     hexserial
    nop
    dsrl    a0, s6, 0
    bal     hexserial
    nop
    dsrl    a0, s5, 32
    bal     hexserial
    nop
    dsrl    a0, s5, 0
    bal     hexserial
    nop
    PRINTSTR("\r\n")
#endif
```

##### 电平校准寄存器操作细节 (来自 `ddr4_leveling.S`)

**寄存器使用约定（电平校准专用）**：

| 寄存器 | 用途 | 生存期 | 备注 |
|--------|------|--------|------|
| `t8` | 基地址寄存器，固定为 `0x900000000ff00000`（内存控制器寄存器空间基址） | 整个电平校准过程 | 代码开头注释明确指定 |
| `t0-t7`, `t9` | 临时寄存器（elastic reg） | 临时使用 | 根据算法需要动态分配 |
| `s5` | 电平响应状态指示器（`lvl_resp`） | 写电平和门电平校准 | 0 - 响应为1时找到0；响应为0时找到1 |
| `s7` | DLL_WRDQ调整完成标志 | 写电平校准 | 0 - 完成，1 - 进行中 |
| `s6` | 保留（用于额外的电平请求） | 未使用 | 注释标记为"not used now" |
| `a0-a3`, `v0-v1` | 函数参数/返回值 | 临时 | 用于宏调用和函数返回 |
| `ra` | 返回地址 | 函数调用 | 主函数中由`t9`保存 |

**关键内存映射偏移（PHY地址空间）**：

| 偏移量宏定义 | 物理地址（相对t8） | 功能描述 |
|--------------|-------------------|----------|
| `PHY_ADDRESS + 0x700` | 0x1000700 | 电平请求控制寄存器（LVL_REQ） |
| `PHY_ADDRESS + 0x708` | 0x1000708 | 电平状态寄存器（检查校准是否完成） |
| `PHY_ADDRESS + 0x710` | 0x1000710 | 电平响应数据寄存器（每个数据片的响应位） |
| `PHY_ADDRESS + 0x100` | 0x1000100 | DLL_WRDQ控制寄存器（每个数据片偏移0x80） |
| `PHY_ADDRESS + 0x108` | 0x1000108 | DLL_GATE控制寄存器 |
| `CTL_ADDRESS + 0x280` | 0x2000280 | 控制器配置寄存器（检查ECC使能） |
| `LVL_MODE_OFFSET` | 未在文件中定义（可能来自头文件） | 电平模式选择寄存器 |
| `DDR4_MR1_CS0_REG` | 可变（CS相关） | DDR4模式寄存器1（MR1） |
| `DDR4_MR3_CS0_REG` | 可变 | DDR4模式寄存器3（MR3） |
| `DDR4_MR4_CS0_REG` | 可变 | DDR4模式寄存器4（MR4） |

**临时存储区域（SRAM/Scratchpad）**：

代码使用固定地址作为临时变量存储，这些地址位于内存控制器地址空间内：

| 地址偏移 | 符号名 | 功能描述 |
|----------|--------|----------|
| `RDDATA_STORE` (0x3319) | 读数据存储区 | 存储每个数据片的RDDATA值（9字节，每数据片1字节） |
| `RDDATA_FIND_FLAG` (0x3330) | 标志位存储区 | 指示哪些数据片已找到有效读数据（2字节位图） |
| `DLL_GATE_STORE` (0x3300) | DLL_GATE备份区 | 存储DLL_GATE的备份值（用于调试，9字节） |
| `DLY_2X_STORE` (0x3310) | DLY_2X备份区 | 存储DLY_2X的备份值（用于调试，9字节） |
| `GATE_POSITION_STORE` (0x3300) | DDR3门校准存储区 | DDR3门校准时存储（gate, rddata）对（18字节，每数据片2字节） |

**写电平校准关键寄存器操作**：

```assembly
// 1. 设置写电平模式
sb      zero, LVL_MODE_OFFSET(t8)     // 先清零
WAIT_FOR(20000)                      // 等待20,000个时钟周期
li      t2, 1
sb      t2, LVL_MODE_OFFSET(t8)      // 设置模式为1（写电平）

// 2. 发起电平请求
ld      t2, (PHY_ADDRESS + 0x700)(t8)
dli     t3, 0xffffffffffff00ff
and     t2, t2, t3
ori     t2, t2, (0x1 << 8)           // 设置位8（写电平请求）
sd      t2, (PHY_ADDRESS + 0x700)(t8)

// 3. 等待电平完成
lvl_done:
ld      t2, (PHY_ADDRESS + 0x708)(t8)
andi    t2, t2, (0x1 << 8)           // 检查位8（写电平完成）
beqz    t2, lvl_done                 // 未完成则循环等待
nop

// 4. 读取响应并调整DLL_WRDQ
dli     t1, (PHY_ADDRESS + 0x100)    // DLL_WRDQ寄存器基址
or      t1, t1, t8
dsll    t5, t4, 0x7                  // t5 = 数据片索引 * 0x80
dadd    t1, t1, t5                   // t1 = 当前数据片的DLL_WRDQ寄存器
ld      t2, (0x0)(t1)                // 读取当前DLL_WRDQ值
andi    t6, t2, 0x7f                 // 提取低7位（DLL_WRDQ值）
dli     t7, 0x1
dadd    t6, t6, t7                   // DLL_WRDQ加1
dli     t5, 0x7f
and     t6, t6, t5                   // 保持7位范围（0-127）
sb      t6, 0x0(t1)                  // 写回DLL_WRDQ
sb      t6, 0x3(t1)                  // 同时设置DLL_1XDLY（相同值）
```

**门电平校准关键寄存器操作**：

```assembly
// 1. 设置门电平模式
sb      zero, LVL_MODE_OFFSET(t8)     // 先清零
WAIT_FOR(20000)
li      t2, 2
sb      t2, LVL_MODE_OFFSET(t8)      // 设置模式为2（门电平）

// 2. 初始化DLL_GATE和DLY_2X
move    t1, t8
dli     t2, 0
dli     t4, 9                        // 最多9个数据片
1:
sb      t2, DLL_GATE_OFFSET(t1)      // DLL_GATE清零
lb      t3, DLY_2X_OFFSET(t1)
dli     t5, 0x3c                     // 清除DLY_2X[5:2]
not     t5
and     t3, t5
sb      t3, DLY_2X_OFFSET(t1)        // DLY_2X[5:2]清零
daddu   t1, 0x80                     // 下一个数据片（偏移0x80）
dsubu   t4, 1
bnez    t4, 1b
nop

// 3. 扫描寻找稳定窗口
dli     t1, (PHY_ADDRESS + 0x108)    // DLL_GATE寄存器基址
or      t1, t1, t8
dsll    t5, t4, 0x7                  // t5 = 数据片索引 * 0x80
dadd    t1, t1, t5                   // t1 = 当前数据片的DLL_GATE寄存器
ld      t2, (0x0)(t1)
dli     t7, 0x7f0000
and     t6, t2, t7                   // 提取DLL_GATE值（位16-22）
dli     t7, (0x1 << 16)              // 增量步长
dadd    t6, t6, t7                   // DLL_GATE加1
dli     t5, 0x00000000007f0000
and     t6, t6, t5                   // 保持7位范围
dli     t5, 0xffffffffff00ffff
and     t2, t2, t5                   // 清除旧的DLL_GATE值
or      t2, t2, t6                   // 设置新的DLL_GATE值
sd      t2, (0x0)(t1)                // 写回寄存器
```

**C伪代码解释核心逻辑**：

**写电平校准算法**：
```c
#define CHECKCOUNT 0x20      // 32次检查阈值
#define IDLE_TIMES 0x1000    // 空闲等待时间

void write_leveling_calibration(int dataslice_count) {
    uint8_t dll_wrdq[dataslice_count];
    uint8_t found_zero[dataslice_count] = {0};
    uint8_t found_one[dataslice_count] = {0};
    
    // 阶段1：确保所有数据片响应为0
    for (int i = 0; i < dataslice_count; i++) {
        dll_wrdq[i] = 0;
        int consecutive_zeros = 0;
        
        while (consecutive_zeros < CHECKCOUNT) {
            // 设置DLL_WRDQ值
            set_dll_wrdq(i, dll_wrdq[i]);
            
            // 发起校准请求并获取响应
            uint8_t response = get_level_response(i);
            
            if (response == 0) {
                consecutive_zeros++;
            } else {
                consecutive_zeros = 0;
                dll_wrdq[i] = (dll_wrdq[i] + 1) & 0x7F;  // 7位范围
                
                // 处理溢出（当值从127回到0时需要额外等待）
                if (dll_wrdq[i] == 0) {
                    wait_idle(IDLE_TIMES);
                }
            }
        }
        found_zero[i] = dll_wrdq[i];
    }
    
    // 阶段2：从找到0的位置继续，直到找到1
    for (int i = 0; i < dataslice_count; i++) {
        int consecutive_ones = 0;
        
        while (consecutive_ones < CHECKCOUNT) {
            set_dll_wrdq(i, dll_wrdq[i]);
            uint8_t response = get_level_response(i);
            
            if (response == 1) {
                consecutive_ones++;
            } else {
                consecutive_ones = 0;
                dll_wrdq[i] = (dll_wrdq[i] + 1) & 0x7F;
                
                if (dll_wrdq[i] == 0) {
                    wait_idle(IDLE_TIMES);
                }
            }
        }
        found_one[i] = dll_wrdq[i];
    }
    
    // 计算最佳值：取从0到1跳变的中间点
    for (int i = 0; i < dataslice_count; i++) {
        int optimal_value = found_one[i] - (CHECKCOUNT / 2);
        if (optimal_value < 0) optimal_value += 128;
        optimal_value &= 0x7F;
        
        // 应用最终值
        set_dll_wrdq(i, optimal_value);
        set_dll_1xdly(i, optimal_value);  // DLL_1XDLY设置为相同值
        
        // 计算DLY_2X补偿值
        calculate_dly_2x_compensation(i, optimal_value);
    }
}
```

**门电平校准算法**：
```c
#define CONTINUE_VALUE 0x20      // 32个连续响应阈值
#define FLUCTUATION_MARGIN_VALUE 0x10  // 波动裕量

void gate_leveling_calibration(int dataslice_count) {
    uint8_t rddata[dataslice_count];
    uint8_t dll_gate[dataslice_count];
    uint8_t found_window[dataslice_count] = {0};
    
    // 初始化
    for (int i = 0; i < dataslice_count; i++) {
        rddata[i] = GATE_RDDATA_START;  // 起始值
        dll_gate[i] = 0;
    }
    
    // 扫描寻找稳定窗口
    for (int slice = 0; slice < dataslice_count; slice++) {
        if (found_window[slice]) continue;
        
        for (int r = 0; r < MAX_RDDATA; r++) {
            set_rddata(slice, rddata[slice]);
            
            for (int g = 0; g < 128; g++) {  // DLL_GATE范围0-127
                set_dll_gate(slice, dll_gate[slice]);
                
                // 发起校准请求
                trigger_level_request();
                wait_for_level_done();
                
                // 读取响应
                uint8_t response = get_gate_response(slice);
                
                // 检查连续响应
                if (response == 1) {
                    consecutive_ones_count[slice]++;
                    if (consecutive_ones_count[slice] >= CONTINUE_VALUE) {
                        // 找到稳定窗口
                        found_window[slice] = 1;
                        store_window_position(slice, dll_gate[slice], rddata[slice]);
                        break;
                    }
                } else {
                    consecutive_ones_count[slice] = 0;
                }
                
                dll_gate[slice] = (dll_gate[slice] + 1) & 0x7F;
                
                // 处理DLL_GATE溢出
                if (dll_gate[slice] == 0) {
                    wait_idle(IDLE_TIMES);
                    // 同步DLL：发送128次校准请求
                    for (int sync = 0; sync < 128; sync++) {
                        trigger_level_request();
                        wait_for_level_done();
                    }
                }
            }
            
            if (found_window[slice]) break;
            
            // 增加RDDATA值，继续扫描
            rddata[slice]++;
            dll_gate[slice] = 0;
        }
    }
    
    // 计算最终值
    // 1. 找到所有数据片中最小的RDDATA值作为基准
    uint8_t min_rddata = 0xFF;
    for (int i = 0; i < dataslice_count; i++) {
        if (window_rddata[i] < min_rddata) {
            min_rddata = window_rddata[i];
        }
    }
    
    // 2. 调整每个数据片的DLL_GATE和DLY_2X
    for (int i = 0; i < dataslice_count; i++) {
        // 调整DLL_GATE：找到的值减去0x40（居中）
        int adjusted_gate = window_gate[i] - 0x40;
        if (adjusted_gate < 0) adjusted_gate += 128;
        adjusted_gate &= 0x7F;
        set_dll_gate(i, adjusted_gate);
        
        // 调整RDDATA
        if (adjusted_gate >= 0x40) {  // 如果调整后的gate >= 0x40
            window_rddata[i]--;       // RDDATA减1
        }
        
        // 计算DLY_2X补偿：根据(数据片RDDATA - 基准RDDATA)
        int rddata_diff = window_rddata[i] - min_rddata;
        set_dly_2x_compensation(i, rddata_diff);
    }
    
    // 3. 设置最终的TPHY_RDDATA（取所有数据片中最小的RDDATA）
    set_tphy_rddata(min_rddata);
}
```

**配置参数总结**：

| 参数名 | 值 | 描述 |
|--------|----|------|
| `CHECKCOUNT` | 0x20 (32) | 连续响应检查阈值，确保稳定性 |
| `CHECKCOUNT_G` | 0x20 (32) | 门电平校准的连续检查阈值 |
| `IDLE_TIMES` | 0x1000 (4096) | DLL值溢出（从127回到0）时的空闲等待时间 |
| `WAIT_ITEM` | 60 | 通用等待项（未在代码中使用） |
| `CONTINUE_VALUE` | 0x20 (32) | 连续相同响应的阈值（用于窗口检测） |
| `FLUCTUATION_MARGIN_VALUE` | 0x10 (16) | 波动裕量值（DDR3门校准用） |
| `GATE_MARGIN_VALUE` | 0x20 (32) | 门限裕量值（DDR3门校准用） |
| `GATE_RDDATA_START` | 0x6 | 门电平校准起始RDDATA值 |

#### 3.4 写位训练 (`wr_bit_training.S`)

**位独立延迟校准**：
```
wr_bit_training:
    ↓
数据片循环 (s6=0~n)
    ↓
位循环 (s5=0~7)
    ↓
重复测试循环 (REPEAT_TIMES=4次)
    ↓
参数扫描:
    ├── WRDQS DLL扫描: 0~DLL_VALUE_OFFSET (最大63)
    ├── WRDQS BDLY扫描: 0~(DLL_VALUE_OFFSET-63)
    └── 调用test_engine验证
    ↓
结果处理:
    ├── 重复性过滤: 通过次数 < (REPEAT_TIMES-1)则清零
    ├── 连续0填充: 修复结果中的"空洞"
    └── 寻找最大连续通过区域
    ↓
最优参数计算:
    ├── 计算连续1区域的中间点
    ├── 对所有位取中间点最大值作为基准
    ├── 每个位的延迟 = 基准 - 该位中间点
    └── 限制延迟值不超过0x10
```

**寄存器操作细节**：

##### 关键寄存器偏移（写位训练专用）
| 偏移量 | 绝对地址 | 功能描述 |
|--------|----------|----------|
| `DLL_WRDQS_OFFSET` | 0x0101 | WRDQS DLL 控制寄存器，[7]=旁路模式(0x80) |
| `WRDQS0_BDLY_OFFSET` | 0x012b | WRDQS BDLY 控制寄存器，数字延迟线配置 |
| `WRDQ_BDLY00_OFFSET` | 0x0120 | WRDQ BDLY 控制寄存器（位0），每个数据位独立延迟 |
| `DLL_VALUE_OFFSET` | 0x0036 | DLL 最大值配置，决定扫描范围 |
| `BASE_ADDR_OFFSET` | 0x1140 | 测试引擎基地址配置 |
| `VALID_BITS_OFFSET` | 0x1148 | 有效数据位掩码，用于测试引擎 |
| `VALID_BITS_ECC_OFFSET` | 0x115b | ECC有效位掩码 |
| `PAGE_SIZE_OFFSET` | 0x1150 | 测试页面大小配置 |
| `PAGE_NUM_OFFSET` | 0x1154 | 测试页面数量配置 |

##### 训练结果存储区（文件内定义）
| 符号名 | 偏移量 | 功能描述 |
|--------|--------|----------|
| `WRDQS_DLL_RESULT_BIT0` | 0x3300 | DLL测试结果存储区起始地址（每循环16字节） |
| `WRDQS_BDLY_RESULT_BIT0` | 0x3308 | BDLY测试结果存储区起始地址 |
| `WRDQS_DLL_RESULT_OR` | 0x1160 | 多次循环DLL结果的按位或累积 |
| `WRDQS_BDLY_RESULT_OR` | 0x1168 | 多次循环BDLY结果的按位或累积 |
| `FIND_CONTINUE_MAX` | 0x1158 | 最大连续通过数存储 |
| `FIND_CONTINUE_LSB` | 0x1159 | 连续通过区域的起始位置存储 |
| `CYCLE_COUNT` | 0x115a | 当前循环计数存储 |
| `MID_STORE_BIT0` | 0x1170 | 每个数据位的中间点计算结果存储 |

##### 寄存器配置操作细节
**1. DLL寄存器配置（旁路模式）**：
```assembly
or      t1, t5, 0x80    # t5 = DLL值, 0x80 = 旁路模式
dsll    t0, s6, 7       # t0 = s6 * 0x80 (数据片偏移)
daddu   t0, t0, t8      # t0 = 基址 + 数据片偏移
sb      t1, DLL_WRDQS_OFFSET(t0)  # 写入DLL寄存器
```

**2. BDLY寄存器配置**：
```assembly
dsll    t0, s6, 7       # t0 = s6 * 0x80 (数据片偏移)
daddu   t0, t0, t8      # t0 = 基址 + 数据片偏移
sb      t5, WRDQS0_BDLY_OFFSET(t0)  # 写入BDLY寄存器，t5 = BDLY值
```

**3. 测试引擎参数配置**：
```assembly
sd      t0, BASE_ADDR_OFFSET(t8)     # 设置测试基地址
sd      t0, VALID_BITS_OFFSET(t8)    # 设置有效位掩码
sb      t0, PAGE_SIZE_OFFSET(t8)     # 设置页面大小
sw      t0, PAGE_NUM_OFFSET(t8)      # 设置页面数量
```

**4. 错误模式读取（测试失败时）**：
```assembly
# 对于非ECC数据片（s6 != 8）
ld      t1, 0x3180(t8)   # 读取错误寄存器0
ld      t2, 0x3188(t8)   # 读取错误寄存器1
or      t1, t2           # 合并错误位
ld      t2, 0x31a0(t8)   # 读取错误寄存器2
or      t1, t2           # 合并错误位
ld      t2, 0x31a8(t8)   # 读取错误寄存器3
or      t1, t2           # 合并错误位
not     t1               # 反转：1表示测试通过

# 对于ECC数据片（s6 == 8）
lb      t1, 0x316e(t8)   # 读取ECC错误寄存器0
lb      t2, 0x316f(t8)   # 读取ECC错误寄存器1
or      t1, t2           # 合并错误位
lb      t2, 0x31b2(t8)   # 读取ECC错误寄存器2
or      t1, t2           # 合并错误位
lb      t2, 0x31b3(t8)   # 读取ECC错误寄存器3
or      t1, t2           # 合并错误位
not     t1               # 反转：1表示测试通过
```

**5. 测试结果存储**：
```assembly
# DLL结果存储
lb      t2, CYCLE_COUNT(t8)     # 获取当前循环计数
dsll    t6, t2, 4               # t6 = t2 * 16（每个循环16字节）
daddu   t0, t6, t8              # t0 = 结果存储地址
dsrl    t6, t1, s5              # 提取当前位的结果
and     t6, 0x1                 # 取最低位
dsll    t6, t6, t5              # 左移到DLL值对应位置
ld      t2, WRDQS_DLL_RESULT_BIT0(t0)  # 读取现有结果
or      t2, t6                  # 设置对应位
sd      t2, WRDQS_DLL_RESULT_BIT0(t0)  # 写回结果

# BDLY结果存储（类似逻辑）
ld      t3, WRDQS_BDLY_RESULT_BIT0(t0)  # 读取现有结果
or      t3, t6                  # 设置对应位
sd      t3, WRDQS_BDLY_RESULT_BIT0(t0)  # 写回结果
```

**6. 最终参数计算与设置**：
```assembly
# DLL值计算
lb      t0, DLL_VALUE_OFFSET(t8)    # 获取DLL最大值
dsll    t1, t7, 7                  # t1 = 中间点 * 128
ddivu   t0, t1, t0                 # t0 = (中间点 * 128) / DLL最大值
dsll    t1, s6, 7                  # t1 = 数据片偏移
daddu   t1, t1, t8                 # t1 = 目标寄存器地址
sb      t0, DLL_WRDQS_OFFSET(t1)   # 写入最终DLL值

# BDLY值清零
sb      t2, WRDQS0_BDLY_OFFSET(t1) # 清零BDLY值

# WRDQ BDLY值设置（每个位独立）
dsll    t3, s6, 7                  # t3 = 数据片偏移
daddu   t3, t3, t8                 # t3 = 寄存器基址
daddu   t1, t0, t8                 # t1 = MID_STORE_BIT0偏移 + 基址
lb      t2, MID_STORE_BIT0(t1)     # 读取该位的中间点
dsubu   t2, t7, t2                 # t2 = 基准中间点 - 该位中间点
daddu   t1, t3, t0                 # t1 = 目标寄存器地址
sb      t2, (WRDQ_BDLY00_OFFSET)(t1)  # 写入WRDQ BDLY值
```

##### 关键算法寄存器操作
**1. 重复性过滤算法**：
```assembly
# 统计某个位置在多次循环中的通过次数
dli     t2, 0       # 循环计数器
dli     t6, 0       # 通过次数计数器
wrdqs_dll_correct_loop:
    dsll    t3, t2, 4           # t3 = 循环索引 * 16
    daddu   t3, t8              # t3 = 结果存储地址
    ld      t5, WRDQS_DLL_RESULT_BIT0(t3)  # 读取该循环的结果
    dsrl    t5, t0              # 移位到目标位置
    and     t5, 0x1             # 取最低位
    beqz    t5, wrdqs_dll_correct_loop_ctrl  # 如果为0，跳过计数
    nop
    daddu   t6, 1               # 通过次数加1
```

**2. 连续0填充算法**：
```assembly
dli     t6, CORRECT_PARAM       # 校正参数（0x9）
dli     t2, 0x1
dsll    t2, t6                  # t2 = 1 << CORRECT_PARAM
dsubu   t2, 1                   # t2 = (1 << CORRECT_PARAM) - 1（掩码）
dsrl    t1, t6, 1               # t1 = CORRECT_PARAM / 2
dli     t3, 0x1
dsll    t3, t1                  # t3 = 1 << (CORRECT_PARAM/2)
not     t3                      # t3 = ~(1 << (CORRECT_PARAM/2))
and     t1, t2, t3              # t1 = 目标模式（中间有0的模式）

# 扫描查找中间有0的连续区域
ld      t3, WRDQS_DLL_RESULT_OR(t8)  # 读取或结果
dsrl    t5, t3, t0              # 移位到当前位置
and     t5, t2                  # 应用掩码
bne     t5, t1, skip_fill       # 如果不是目标模式，跳过
nop
move    t5, t2
dsll    t5, t0                  # t5 = 掩码 << 位置
or      t3, t5                  # 设置这些位为1
sd      t3, WRDQS_DLL_RESULT_OR(t8)  # 写回结果
```

**3. 最大连续区域查找**：
```assembly
dli     t6, 0                   # 当前连续1计数
sb      t6, FIND_CONTINUE_MAX(t8)  # 初始化最大计数
sb      t6, FIND_CONTINUE_LSB(t8)  # 初始化起始位置

1:
    ld      t3, WRDQS_DLL_RESULT_OR(t8)  # 读取DLL或结果
    and     t2, t3, 0x1          # 取最低位
    dsrl    t3, 1                # 右移一位
    ld      t7, WRDQS_BDLY_RESULT_OR(t8) # 读取BDLY或结果
    and     t5, t7, 0x1          # 取最低位
    dsrl    t7, 1                # 右移一位
    dsll    t5, 63               # 将BDLY位移到最高位
    or      t3, t5               # 合并到DLL结果的高位
    sd      t3, WRDQS_DLL_RESULT_OR(t8)  # 更新DLL结果
    dsll    t2, 15               # 将DLL位移到BDLY结果的高位
    or      t7, t2               # 合并到BDLY结果
    sd      t7, WRDQS_BDLY_RESULT_OR(t8)  # 更新BDLY结果
    
    beqz    t2, count_reset      # 如果当前位为0，重置计数
    nop
    daddu   t6, 1                # 连续1计数加1
    b       continue_check
    nop
    
count_reset:
    lb      t3, FIND_CONTINUE_MAX(t8)  # 读取当前最大值
    bgeu    t3, t6, do_reset     # 如果当前计数不大于最大值，跳过更新
    nop
    move    t3, t6               # 更新最大值
    sb      t3, FIND_CONTINUE_MAX(t8)  # 保存最大值
    dsubu   t2, t0, t3           # 计算起始位置 = 当前位置 - 计数
    sb      t2, FIND_CONTINUE_LSB(t8)  # 保存起始位置
    
do_reset:
    dli     t6, 0                # 重置连续计数
    
continue_check:
    daddu   t0, 1                # 位置加1
    bleu    t0, 79, 1b           # 总共检查80位（DLL:64 + BDLY:16）
    nop
```

##### 调试寄存器操作
**1. 调试信息输出**：
```assembly
#ifdef WR_BIT_TRAINING_DEBUG
    PRINTSTR("\r\nSlice No.")
    move    a0, s6               # 数据片编号
    bal     hexserial            # 十六进制输出
    nop
    PRINTSTR(" start")
#endif
```

**2. 详细调试输出**：
```assembly
#ifdef WR_BIT_TRAINING_DETAIL_DEBUG
    PRINTSTR("\r\nwrdqs_dll is ")
    move    a0, t5               # DLL值
    bal     hexserial            # 十六进制输出
    nop
#endif
```

##### 配置参数
| 参数名 | 值 | 描述 |
|--------|----|------|
| `REPEAT_TIMES` | 0x4 | 每个参数重复测试次数 |
| `PAGE_SIZE` | 0x0 | 测试页面大小配置 |
| `PAGE_NUMBER` | 0x1 | 测试页面数量配置 |
| `CORRECT_PARAM` | 0x9 | 结果校正参数，用于连续0填充 |
| `WATI_INIT_TIME` | 0x200 | 初始化等待时间循环计数 |
| `WR_BIT_TRAINING_DEBUG` | 定义 | 控制调试信息输出 |
| `WR_BIT_TRAINING_DETAIL_DEBUG` | 定义 | 控制详细调试信息输出 |

##### 寄存器使用约定
| 寄存器 | 用途 | 生存期 |
|--------|------|--------|
| `t8` | 内存控制器寄存器基址指针 | 整个函数 |
| `s6` | 当前数据片编号（slice number） | 数据片循环 |
| `s5` | 当前位编号（bit number，0~7） | 位循环 |
| `t9` | 保存返回地址 | 整个函数 |
| `t0-t7` | 临时计算和循环控制 | 局部块 |
| `v0` | 函数返回值（test_engine测试结果） | 测试调用后 |
| `a0-a1` | 参数传递和调试输出 | 临时 |
| `s7` | 在test_engine中保存返回地址 | test_engine函数 |

**基址计算**：
```assembly
# 节点基址计算（test_engine.S中使用）
GET_NODE_ID_a0                 # 获取节点ID到a0
dli     t8, 0x900000000ff00000  # 内存控制器基址
or      t8, a0                 # 加上节点偏移
```

#### 3.5 读位训练 (`rd_bit_training.S`)

**DLL与比特延迟联合优化**：
```
rd_bit_training:
    ↓
数据片迭代 (每个数据片独立训练)
    ↓
阶段一: DLL延迟扫描(粗调)
    ├── 扫描64个DLL值 (0-63)
    ├── 每个值进行10次稳定性测试
    ├── 压缩结果: 相邻两个DLL值都通过才算稳定
    └── 记录到BITx_TMP_REG_LO
    ↓
阶段二: 比特延迟扫描(细调)
    ├── 扫描16个BDLY值 (0-15)
    ├── 记录到BITx_TMP_REG_HI
    └── 压缩结果: 相邻两个BDLY值都通过才算稳定
    ↓
最优值计算:
    ├── 使用find_continues_1找到连续通过窗口
    ├── 返回格式: [7:0]最大长度, [15:8]起始位置, [23:16]中间位置
    ├── DLL值计算: (中间位置 * 128) / DLL基准值
    └── 比特延迟计算: 训练结果 - 中间位置
```

**寄存器操作细节**：

##### 关键寄存器偏移（读位训练专用）
| 偏移量 | 绝对地址 | 功能描述 |
|--------|----------|----------|
| `BIT0_TMP_REG_LO` | 0x3300 | 比特0低32位DLL测试结果存储 |
| `BIT1_TMP_REG_LO` | 0x3308 | 比特1低32位DLL测试结果存储 |
| `BIT2_TMP_REG_LO` | 0x3310 | 比特2低32位DLL测试结果存储 |
| `BIT3_TMP_REG_LO` | 0x3318 | 比特3低32位DLL测试结果存储 |
| `BIT4_TMP_REG_LO` | 0x3320 | 比特4低32位DLL测试结果存储 |
| `BIT5_TMP_REG_LO` | 0x3328 | 比特5低32位DLL测试结果存储 |
| `BIT6_TMP_REG_LO` | 0x3330 | 比特6低32位DLL测试结果存储 |
| `BIT7_TMP_REG_LO` | 0x3338 | 比特7低32位DLL测试结果存储 |
| `BIT8_TMP_REG_LO` | 0x3340 | 比特8（ECC）低32位DLL测试结果存储 |
| `BIT0_TMP_REG_HI` | 0x3304 | 比特0高32位BDLY测试结果存储 |
| `BIT1_TMP_REG_HI` | 0x330c | 比特1高32位BDLY测试结果存储 |
| `BIT2_TMP_REG_HI` | 0x3314 | 比特2高32位BDLY测试结果存储 |
| `BIT3_TMP_REG_HI` | 0x331c | 比特3高32位BDLY测试结果存储 |
| `BIT4_TMP_REG_HI` | 0x3324 | 比特4高32位BDLY测试结果存储 |
| `BIT5_TMP_REG_HI` | 0x332c | 比特5高32位BDLY测试结果存储 |
| `BIT6_TMP_REG_HI` | 0x3334 | 比特6高32位BDLY测试结果存储 |
| `BIT7_TMP_REG_HI` | 0x333c | 比特7高32位BDLY测试结果存储 |
| `BIT8_TMP_REG_HI` | 0x3344 | 比特8（ECC）高32位BDLY测试结果存储 |
| `COUNT_REG` | 0x3350 | 稳定性测试计数寄存器 |
| `TMP_REG0` | 0x3360 | 临时寄存器0（存储RDQSN BDLY计算结果） |
| `TMP_REG1` | 0x3368 | 临时寄存器1（存储RDQSP BDLY计算结果） |
| `TMP_REG2` | 0x3370 | 临时寄存器2（保存返回地址） |
| `TMP_REG3` | 0x3378 | 临时寄存器3（保留） |

##### 内存控制器寄存器偏移（来自`ls3a4000_reg_def.h`）
| 偏移量 | 符号名 | 功能描述 |
|--------|--------|----------|
| 0x0108 | `DLL_RDDQS0_OFFSET` | 读DQS DLL控制0寄存器 |
| 0x0109 | `DLL_RDDQS1_OFFSET` | 读DQS DLL控制1寄存器（ECC使用） |
| 0x0150 | `RDQSP_BDLY00_OFFSET` | 读DQS正相延迟比特0寄存器 |
| 0x0151 | `RDQSP_BDLY01_OFFSET` | 读DQS正相延迟比特1寄存器 |
| 0x0152 | `RDQSP_BDLY02_OFFSET` | 读DQS正相延迟比特2寄存器 |
| 0x0153 | `RDQSP_BDLY03_OFFSET` | 读DQS正相延迟比特3寄存器 |
| 0x0154 | `RDQSP_BDLY04_OFFSET` | 读DQS正相延迟比特4寄存器 |
| 0x0155 | `RDQSP_BDLY05_OFFSET` | 读DQS正相延迟比特5寄存器 |
| 0x0156 | `RDQSP_BDLY06_OFFSET` | 读DQS正相延迟比特6寄存器 |
| 0x0157 | `RDQSP_BDLY07_OFFSET` | 读DQS正相延迟比特7寄存器 |
| 0x0160 | `RDQSN_BDLY00_OFFSET` | 读DQS负相延迟比特0寄存器 |
| 0x0161 | `RDQSN_BDLY01_OFFSET` | 读DQS负相延迟比特1寄存器 |
| 0x0162 | `RDQSN_BDLY02_OFFSET` | 读DQS负相延迟比特2寄存器 |
| 0x0163 | `RDQSN_BDLY03_OFFSET` | 读DQS负相延迟比特3寄存器 |
| 0x0164 | `RDQSN_BDLY04_OFFSET` | 读DQS负相延迟比特4寄存器 |
| 0x0165 | `RDQSN_BDLY05_OFFSET` | 读DQS负相延迟比特5寄存器 |
| 0x0166 | `RDQSN_BDLY06_OFFSET` | 读DQS负相延迟比特6寄存器 |
| 0x0167 | `RDQSN_BDLY07_OFFSET` | 读DQS负相延迟比特7寄存器 |
| 0x0036 | `DLL_VALUE_OFFSET` | DLL最大值配置寄存器 |

##### 寄存器配置操作细节

**1. 数据片基址计算**：
```assembly
dsll   t5, t4, 7          # t5 = 数据片编号(t4) * 128（每个数据片128字节偏移）
dadd   t5, t5, t8         # t5 = 内存控制器基址(t8) + 数据片偏移
```

**2. DLL寄存器配置（旁路模式）**：
```assembly
dli    a0, 0x80           # 旁路模式值0x80
move   a1, t9             # t9 = DLL索引值(0-63)
or     a0, a1, a0         # 合并：a0 = DLL值 | 0x80
sb     a0, DLL_RDDQS0_OFFSET(t5)  # 写入DLL_RDDQS0寄存器
# 如果支持ECC且数据片编号>3，同时配置DLL_RDDQS1
lb     a1, 0xd(t8)        # 检查配置标志
bne    a1, 3, skip_ecc_dll
nop
sb     a0, DLL_RDDQS1_OFFSET(t5)  # 写入DLL_RDDQS1寄存器（ECC）
skip_ecc_dll:
```

**3. 比特延迟寄存器初始化**：
```assembly
# 清零所有RDQSP和RDQSN延迟寄存器
sb     $0, RDQSP_BDLY00_OFFSET(t5)
sb     $0, RDQSP_BDLY01_OFFSET(t5)
sb     $0, RDQSP_BDLY02_OFFSET(t5)
sb     $0, RDQSP_BDLY03_OFFSET(t5)
sb     $0, RDQSP_BDLY04_OFFSET(t5)
sb     $0, RDQSP_BDLY05_OFFSET(t5)
sb     $0, RDQSP_BDLY06_OFFSET(t5)
sb     $0, RDQSP_BDLY07_OFFSET(t5)
sb     $0, RDQSN_BDLY00_OFFSET(t5)
sb     $0, RDQSN_BDLY01_OFFSET(t5)
sb     $0, RDQSN_BDLY02_OFFSET(t5)
sb     $0, RDQSN_BDLY03_OFFSET(t5)
sb     $0, RDQSN_BDLY04_OFFSET(t5)
sb     $0, RDQSN_BDLY05_OFFSET(t5)
sb     $0, RDQSN_BDLY06_OFFSET(t5)
sb     $0, RDQSN_BDLY07_OFFSET(t5)
```

**4. 测试引擎参数配置**：
```c
// C伪代码解释测试引擎配置逻辑
void configure_test_engine(uint64_t dataslice_num, uint64_t base_addr) {
    if (dataslice_num == 8) {  // ECC数据片
        valid_bits_ecc = 0xFF;  // 所有ECC位有效
        valid_bits = 0x0;       // 数据位无效
    } else {
        valid_bits = 0xFF << (dataslice_num * 8);  // 对应数据片的8位有效
        valid_bits_ecc = 0x0;    // ECC位无效
    }
    
    // 写入配置寄存器
    write_reg(BASE_ADDR_OFFSET, base_addr);
    write_reg(VALID_BITS_OFFSET, valid_bits);
    write_reg(VALID_BITS_ECC_OFFSET, valid_bits_ecc);
    write_reg(PAGE_SIZE_OFFSET, PAGE_SIZE);
    write_reg(PAGE_NUM_OFFSET, PAGE_NUMBER);
}
```

**5. 测试结果收集与处理**：
```assembly
# 非ECC数据片错误模式读取
dsll    t6, t4, 3          # t6 = 数据片编号 * 8（位偏移）
ld      t0, 0x3180(t8)     # 读取错误寄存器0
dsrl    t0, t0, t6         # 右移到当前数据片位置
andi    t0, t0, 0xff       # 提取低8位
or      v1, v1, t0         # 累积到v1[7:0]

ld      t0, 0x3188(t8)     # 读取错误寄存器1
dsrl    t0, t0, t6
andi    t0, t0, 0xff
or      v1, v1, t0         # 累积到v1[15:8]

ld      t0, 0x31a0(t8)     # 读取错误寄存器2
dsrl    t0, t0, t6
andi    t0, t0, 0xff
or      v1, v1, t0         # 累积到v1[23:16]

ld      t0, 0x31a8(t8)     # 读取错误寄存器3
dsrl    t0, t0, t6
andi    t0, t0, 0xff
or      v1, v1, t0         # 累积到v1[31:24]

not     v1, v1             # 反转：1表示测试通过，0表示失败
```

**6. DLL测试结果存储**：
```assembly
# 将测试结果按位存储到BITx_TMP_REG_LO寄存器
move   v0, v1              # v0 = 测试结果（0x1表示该比特通过）
andi   v0, v0, 0x1         # 提取当前比特的结果
dli    t6, 63
dsubu  t6, t6, t9          # t6 = 63 - DLL索引（反向存储）
dsll   v0, v0, t6          # 左移到对应DLL索引位置
ld     t6, BIT0_TMP_REG_LO(t8)  # 读取现有结果
or     t6, t6, v0          # 设置对应位
sd     t6, BIT0_TMP_REG_LO(t8)  # 写回结果
```

**7. BDLY测试结果存储**：
```assembly
# 类似逻辑，但存储到BITx_TMP_REG_HI寄存器
dsrl   v0, v1, t2          # t2 = 偏移量（8或16）
andi   v0, v0, 0x1         # 提取当前比特的结果
dli    t6, 31
dsubu  t6, t6, t9          # t6 = 31 - BDLY索引（反向存储）
dsll   v0, v0, t6          # 左移到对应BDLY索引位置
lw     t6, BIT0_TMP_REG_HI(t8)  # 读取现有结果
or     t6, t6, v0          # 设置对应位
sw     t6, BIT0_TMP_REG_HI(t8)  # 写回结果
```

##### 结果压缩算法寄存器操作

**1. DLL结果压缩（64位→32位）**：
```c
// C伪代码解释压缩算法
uint64_t compress_dll_result(uint64_t raw_result) {
    uint32_t compressed = 0;
    for (int i = 0; i < 32; i++) {
        // 检查相邻两个DLL值是否都通过
        uint64_t bit_pair = (raw_result >> (i * 2)) & 0x3;
        if (bit_pair == 0x3) {  // 两个DLL值都通过
            compressed |= (1 << i);
        }
    }
    return compressed;
}
```

**2. BDLY结果压缩（16位→8位）**：
```c
// C伪代码解释压缩算法  
uint16_t compress_bdly_result(uint16_t raw_result) {
    uint8_t compressed = 0;
    for (int i = 0; i < 8; i++) {
        // 检查相邻两个BDLY值是否都通过
        uint16_t bit_pair = (raw_result >> (i * 2)) & 0x3;
        if (bit_pair == 0x3) {  // 两个BDLY值都通过
            compressed |= (1 << i);
        }
    }
    return compressed;
}
```

##### 核心算法寄存器操作：find_continues_1

**函数功能**：寻找位图中最长的连续"1"区域，返回最大长度、起始位置和中间位置。

**寄存器使用**：
- `v0`：输入位图（64位）
- `t7`：结果寄存器：[7:0]当前计数，[15:8]最大计数，[23:16]起始位置，[31:24]中间位置
- `t6`：当前比特位置索引
- `t2`：临时位图存储

**算法C伪代码**：
```c
typedef struct {
    uint8_t current_count;    // 当前连续1计数
    uint8_t max_count;        // 最大连续1计数
    uint8_t start_pos;        // 最大连续区域起始位置
    uint8_t mid_pos;          // 最大连续区域中间位置
} FindResult;

FindResult find_continues_1(uint64_t bitmap) {
    FindResult result = {0, 0, 0, 0};
    
    // 第一遍扫描：处理低32位
    for (int i = 0; i < 32; i++) {
        uint64_t bit = (bitmap >> (31 - i)) & 0x1;  // 从高位向低位扫描
        
        if (bit == 1) {
            result.current_count++;
        } else {
            if (result.current_count > result.max_count) {
                result.max_count = result.current_count;
                result.start_pos = i - result.current_count;
                result.mid_pos = result.start_pos + (result.current_count / 2);
            }
            result.current_count = 0;
        }
    }
    
    // 第二遍扫描：处理高32位（类似逻辑）
    // ...
    
    return result;
}
```

**汇编实现关键操作**：
```assembly
# 检查当前比特是否为1
dli    a0, 31
dsubu  a1, a0, t6          # a1 = 31 - 当前位置
dsrl   v1, t2, a1          # 提取当前比特
dli    a1, 0x1
and    v1, v1, a1          # v1 = 当前比特值
bne    v1, a1, bit_is_zero # 如果为0，跳转到处理逻辑
nop

# 当前比特为1：计数加1
daddu  t7, t7, 0x1         # 当前计数加1

bit_is_zero:
# 当前比特为0：检查是否需要更新最大值
andi   v1, t7, 0xff        # 提取当前计数
dsrl   a1, t7, 8           # 提取最大计数
and    a1, a1, 0xff
bleu   v1, a1, skip_update # 如果当前计数 <= 最大计数，跳过
nop

# 更新最大计数和起始位置
dli    a1, 0xffff00ff      # 掩码：清除最大计数字段
and    t7, a1, t7          # 清除原最大计数
dsll   v1, v1, 8           # 将当前计数移动到[15:8]
or     t7, t7, v1          # 设置新的最大计数

# 计算起始位置
move   a0, t6              # 当前位置
dsubu  v1, a0, v1          # 起始位置 = 当前位置 - 计数
dli    a0, 0xff00ffff      # 掩码：清除起始位置字段
and    t7, t7, a0          # 清除原起始位置
dsll   v1, v1, 16          # 移动到[23:16]
or     t7, t7, v1          # 设置新的起始位置

skip_update:
# 重置当前计数
dli    a0, 0xffffff00      # 掩码：清除当前计数字段
and    t7, t7, a0
```

##### 最终参数计算与配置

**1. DLL最优值计算**：
```assembly
lb     t6, DLL_VALUE_OFFSET(t8)  # 获取DLL基准值
dsll   a0, s5, 7                # a0 = 中间位置(s5) * 128
divu   t6, a0, t6               # t6 = (中间位置 * 128) / DLL基准值
sb     t6, DLL_RDDQS0_OFFSET(t5) # 写入最终DLL值
```

**2. RDQSP延迟配置**：
```assembly
bit_one_by_one1:
dadd   t0, t5, t9              # t0 = 数据片基址 + 比特偏移
dsll   t6, t9, 3               # t6 = 比特编号 * 8（位偏移）
ld     a0, TMP_REG1(t8)        # 读取RDQSP计算结果
dsrl   t6, a0, t6              # 提取当前比特的值
andi   t6, t6, 0xff            # 取低8位
dsubu  t6, t6, s5              # t6 = 训练结果 - 中间位置
dli    a0, 0x10
bgeu   t6, a0, bdly_error      # 如果延迟值>=16，报错
sb     t6, RDQSP_BDLY00_OFFSET(t0) # 写入RDQSP延迟寄存器
```

**3. RDQSN延迟配置**：
```assembly
bit_one_by_one2:
dadd   t0, t5, t9              # t0 = 数据片基址 + 比特偏移
dsll   t6, t9, 3               # t6 = 比特编号 * 8（位偏移）
ld     a0, TMP_REG0(t8)        # 读取RDQSN计算结果
dsrl   t6, a0, t6              # 提取当前比特的值
andi   t6, t6, 0xff            # 取低8位
dsubu  t6, t6, s5              # t6 = 训练结果 - 中间位置
dli    a0, 0x10
bgeu   t6, a0, bdly_error      # 如果延迟值>=16，报错
sb     t6, RDQSN_BDLY00_OFFSET(t0) # 写入RDQSN延迟寄存器
```

##### 配置参数
| 参数名 | 值 | 描述 |
|--------|----|------|
| `_RDDQS_DLL_NUM` | 64 | DLL扫描范围（0-63） |
| `WAITING_TIME` | 128 | 硬件稳定等待时间循环计数 |
| `PAGE_SIZE` | 0x0 | 测试页面大小配置 |
| `PAGE_NUMBER` | 0x1 | 测试页面数量配置 |
| 稳定性测试次数 | 10 | 每个延迟值测试10次确保稳定性 |

##### 寄存器使用约定
| 寄存器 | 用途 | 生存期 |
|--------|------|--------|
| `t8` | 内存控制器寄存器基址指针 | 整个函数 |
| `t4` | 当前数据片编号（0-8，8为ECC） | 数据片循环 |
| `t5` | 当前数据片寄存器基址（t8 + t4*128） | 数据片循环 |
| `t9` | DLL/BDLY扫描索引 | 扫描循环内 |
| `s5` | 训练结果存储（DLL中间位置） | 整个函数 |
| `s6` | 训练结果存储（ECC DLL中间位置） | 整个函数 |
| `v0`, `v1` | 测试结果和临时计算 | 测试和计算块 |
| `a0`, `a1` | 参数传递和临时存储 | 函数调用和计算 |
| `ra` | 返回地址（保存到TMP_REG2） | 函数入口保存 |

**基址计算**：
```assembly
# 节点基址计算（在调用前已设置）
# t8 = 0x900000000ff00000 | (节点ID << 44)
```

##### 错误处理寄存器操作
```assembly
bdly_error:
PRINTSTR("ERROR: bit dly value for rddqs exceed max value\r\n")
# 延迟值超过0x10（16）时报错
```

##### 调试寄存器操作
**1. 训练进度输出**：
```assembly
PRINTSTR("\r\nnow train dataslice:")
move a0, t4              # 数据片编号
bal hexserial            # 十六进制输出
nop
PRINTSTR("...")
```

**2. 结果向量输出**：
```assembly
PRINTSTR("\r\ntest rddqs_dll vector:\r\n")
lw     a0, BIT0_TMP_REG_HI(t8)  # 读取高32位结果
bal    hexserial
nop
lw     a0, BIT0_TMP_REG_LO(t8)  # 读取低32位结果
bal    hexserial
nop
PRINTSTR("\r\n")
```

**3. 压缩结果输出**：
```assembly
PRINTSTR("\r\ntest bly_dll vector:\r\n")
lw     a0, BIT0_TMP_REG_HI(t8)  # 读取压缩后的BDLY结果
bal    hexserial
nop
PRINTSTR("\r\n")
```

### 阶段四：测试验证 (Verification)

#### 4.1 测试引擎 (`test_engin.S`)

**硬件加速内存测试**：
```
test_engine:
    ↓
配置阶段:
    ├── 清零测试引擎寄存器
    ├── 配置测试参数寄存器组
    │   ├── 0x3100: 控制寄存器0 (页面大小、总线宽度)
    │   ├── 0x3120: 测试基地址
    │   ├── 0x3130: 用户定义测试模式 (0x5555555555555555)
    │   ├── 0x3140: 有效数据位掩码
    │   └── 0x3150: 控制寄存器1 (各种功能控制位)
    └── 设置控制寄存器
    ↓
启动测试: 设置START位(位0)
    ↓
轮询循环: 检查状态寄存器0x3160
    ├── 位1和位2: 测试完成标志 (两者都为1表示完成)
    ├── 位0: 错误标志 (1=错误, 0=正常)
    └── 未完成则继续轮询
    ↓
结果判断:
    ├── 成功: v0=1
    └── 失败: v0=0
```

#### 寄存器操作细节

##### 关键寄存器偏移（测试引擎专用）
| 偏移量 | 绝对地址 | 功能描述 |
|--------|----------|----------|
| 0x3100 | TST+0x100 | 控制寄存器0（页面大小、总线宽度、固定模式索引） |
| 0x3110 | TST+0x110 | 列差异配置（从0x1230加载） |
| 0x3120 | TST+0x120 | 测试基地址（64位） |
| 0x3130 | TST+0x130 | 用户定义测试模式（默认0x5555555555555555） |
| 0x3140 | TST+0x140 | 有效数据位掩码（64位） |
| 0x3148 | TST+0x148 | ECC有效位掩码（8位） |
| 0x3150 | TST+0x150 | 控制寄存器1（各种功能控制位） |
| 0x3154 | TST+0x154 | 页面数量（32位） |
| 0x3158 | TST+0x158 | 读操作间隔（32位） |
| 0x315c | TST+0x15c | 写操作间隔（32位） |
| 0x3160 | TST+0x160 | 状态寄存器（轮询检查） |

##### 输入参数存储区（来自配置阶段）
| 偏移量 | 功能描述 |
|--------|----------|
| BASE_ADDR_OFFSET (0x1140) | 测试基地址配置 |
| VALID_BITS_OFFSET (0x1148) | 有效数据位掩码配置 |
| VALID_BITS_ECC_OFFSET (0x115b) | ECC有效位掩码配置 |
| PAGE_SIZE_OFFSET (0x1150) | 测试页面大小配置 |
| PAGE_NUM_OFFSET (0x1154) | 测试页面数量配置 |

##### 寄存器配置操作细节

**1. 节点基址计算与寄存器清零**：
```assembly
GET_NODE_ID_a0                 # 获取节点ID到a0寄存器
dli     t8, 0x900000000ff00000  # 内存控制器基址
or      t8, a0                 # 加上节点偏移：t8 = 基址 | (节点ID << 44)

# 清零测试引擎寄存器组
sd      $0, 0x3100(t8)         # 控制寄存器0
sd      $0, 0x3120(t8)         # 测试基地址
sd      $0, 0x3130(t8)         # 用户测试模式
sd      $0, 0x3140(t8)         # 有效数据位掩码
sd      $0, 0x3148(t8)         # ECC有效位掩码
sd      $0, 0x3150(t8)         # 控制寄存器1
sd      $0, 0x3158(t8)         # 读操作间隔
```

**2. 控制寄存器0配置（0x3100）**：
```assembly
dli     t0, 0x0
dli     t1, 1
or      t0, t1                 # 位0：保留位（始终为1）
lb      t1, PAGE_SIZE_OFFSET(t8)  # 从配置区读取页面大小
sll     t1, t1, 8              # 移动到位8-10
or      t0, t0, t1
li      t1, BUS_WIDTH          # BUS_WIDTH = 0x3 (64位)
sll     t1, t1, 16             # 移动到位16-17
or      t0, t0, t1
li      t1, FIX_PATTEN_INDEX   # FIX_PATTEN_INDEX = 0x9
sll     t1, t1, 24             # 移动到位24-28
or      t0, t0, t1
sw      t0, 0x3100(t8)         # 写入控制寄存器0
```

**3. 测试参数配置**：
```assembly
# 测试基地址（从配置区加载）
ld      t0, BASE_ADDR_OFFSET(t8)
sd      t0, 0x3120(t8)

# 用户定义测试模式
dli     t0, USR_PATTERN        # USR_PATTERN = 0x5555555555555555
sd      t0, 0x3130(t8)

# 有效数据位掩码
ld      t0, VALID_BITS_OFFSET(t8)
sd      t0, 0x3140(t8)

# ECC有效位掩码
lb      t0, VALID_BITS_ECC_OFFSET(t8)
sb      t0, 0x3148(t8)

# 列差异配置（从控制区复制）
ld      t0, 0x1230(t8)          # 读取列差异值
sd      t0, 0x3110(t8)          # 存储到测试引擎

# 页面数量
lw      t0, PAGE_NUM_OFFSET(t8)
sw      t0, 0x3154(t8)

# 读/写操作间隔
li      t0, READ_INTERVAL      # READ_INTERVAL = 0x0f0f0f0f
sw      t0, 0x3158(t8)
li      t0, WRITE_INTERVAL     # WRITE_INTERVAL = 0x0f0f0f0f
sw      t0, 0x315c(t8)
```

**4. 控制寄存器1配置（0x3150）**：
```assembly
li      t0, 0x0
li      t1, SKIP_INIT          # SKIP_INIT = 0x0
sll     t1, t1, 1              # 位1
or      t0, t0, t1
li      t1, STOP_WR            # STOP_WR = 0x0
sll     t1, t1, 2              # 位2
or      t0, t0, t1
li      t1, STOP_RD            # STOP_RD = 0x0
sll     t1, t1, 3              # 位3
or      t0, t0, t1
li      t1, STOP_WHEN_FINISH   # STOP_WHEN_FINISH = 0x1
sll     t1, t1, 4              # 位4
or      t0, t0, t1
li      t1, CLOSE_COMPARE      # CLOSE_COMPARE = 0x0
sll     t1, t1, 5              # 位5
or      t0, t0, t1
li      t1, INV_USR_PATTERN    # INV_USR_PATTERN = 0x1
sll     t1, t1, 6              # 位6
or      t0, t0, t1
li      t1, FIX_PATTERN_EN     # FIX_PATTERN_EN = 0x1
sll     t1, t1, 7              # 位7
or      t0, t0, t1
li      t1, STOP_WHEN_ERR      # STOP_WHEN_ERR = 0x1
sll     t1, t1, 8              # 位8
or      t0, t0, t1
li      t1, AUTO_START         # AUTO_START = 0x0
sll     t1, t1, 9              # 位9
or      t0, t0, t1
li      t1, RD_WORKER_NUM      # RD_WORKER_NUM = 0x0
sll     t1, t1, 16             # 位16-19
or      t0, t0, t1
li      t1, WR_WORKER_NUM      # WR_WORKER_NUM = 0x0
sll     t1, t1, 20             # 位20-23
or      t0, t0, t1
li      t1, 1
not     t1                     # t1 = 0xfffffffffffffffe
and     t0, t1                 # 清除位0（START位）
sw      t0, 0x3150(t8)         # 写入控制寄存器1（不启动）
sync                           # 确保配置生效
```

**5. 测试启动与状态轮询**：
```assembly
# 设置START位（位0）
lw      t0, 0x3150(t8)         # 读取当前控制寄存器1
li      t1, START              # START = 0x1
sll     t1, t1, 0              # 移动到位0
or      t0, t0, t1             # 设置START位
sw      t0, 0x3150(t8)         # 启动测试

# 轮询状态寄存器（0x3160）
test_loop:
    ld      t0, 0x3160(t8)     # 读取状态寄存器
    li      t1, 0x6            # 检查位1和位2（测试完成标志）
    and     a1, t1, t0
    bne     a1, t1, test_loop  # 未完成（0x6）则继续轮询
    nop
    
    andi    t1, t0, 0x1        # 检查位0（错误标志）
    beqz    t1, test_pass      # 错误位为0表示成功
    nop
```

**6. 测试结果处理**：
```assembly
# 测试失败处理
test_fail:
#ifdef DEBUG_TEST_ENGINE
    PRINTSTR("\r\ntest engine fail\r\n")
#endif
    li      v0, 0              # 返回失败标志
    b       test_engine_end
    nop

# 测试成功处理
test_pass:
#ifdef DEBUG_TEST_ENGINE
    PRINTSTR("\r\ntest engine pass\r\n")
#endif
    li      v0, 1              # 返回成功标志

# 清理阶段
test_engine_end:
    dli     t1, 0x1000         # 延迟循环计数
1:
    dsubu   t1, 1
    bnez    t1, 1b             # 短暂延迟
    nop
    
    sd      $0, 0x3150(t8)     # 清零控制寄存器1
    sd      $0, 0x3100(t8)     # 清零控制寄存器0
    jr      s7                 # 返回调用者（s7保存了返回地址）
    nop
```

##### 关键算法寄存器操作

**1. 状态寄存器位解析算法**：
状态寄存器（0x3160）的位定义：
- **位0**：错误标志（1 = 测试中发生错误，0 = 无错误）
- **位1**：写操作完成标志（1 = 写操作完成）
- **位2**：读操作完成标志（1 = 读操作完成）
- **位1和位2同时为1**：整个测试完成

```assembly
# 测试完成判定逻辑
ld      t0, 0x3160(t8)        # 读取状态寄存器
li      t1, 0x6               # 二进制0110（位1和位2）
and     a1, t1, t0            # 提取完成标志
bne     a1, t1, not_complete  # 如果 != 0x6，测试未完成

# 错误状态判定逻辑
andi    t1, t0, 0x1           # 提取错误标志位
beqz    t1, test_success      # 错误位为0表示成功
```

**2. 控制寄存器1位字段构建算法**：
通过逐步移位和或操作构建控制字，每个控制位有明确的功能定义。

##### 配置参数
| 参数名 | 默认值 | 描述 |
|--------|--------|------|
| `BUS_WIDTH` | 0x3 | 总线宽度（0=8位，1=16位，2=32位，3=64位） |
| `FIX_PATTEN_INDEX` | 0x9 | 固定模式索引（位4：地址递增/递减，位3:0：模式选择） |
| `USR_PATTERN` | 0x5555555555555555 | 用户定义测试模式（64位交替01模式） |
| `START` | 0x1 | 启动测试位 |
| `SKIP_INIT` | 0x0 | 跳过初始化 |
| `STOP_WR` | 0x0 | 停止写操作 |
| `STOP_RD` | 0x0 | 停止读操作 |
| `STOP_WHEN_FINISH` | 0x1 | 完成后停止 |
| `CLOSE_COMPARE` | 0x0 | 关闭比较 |
| `INV_USR_PATTERN` | 0x1 | 反转用户模式 |
| `FIX_PATTERN_EN` | 0x1 | 固定模式使能（0=随机，1=固定） |
| `STOP_WHEN_ERR` | 0x1 | 错误时停止 |
| `AUTO_START` | 0x0 | 自动启动 |
| `RD_WORKER_NUM` | 0x0 | 读工作线程数（0=1线程，1=2线程，2=3线程，3=4线程） |
| `WR_WORKER_NUM` | 0x0 | 写工作线程数 |
| `READ_INTERVAL` | 0x0f0f0f0f | 读操作间隔 |
| `WRITE_INTERVAL` | 0x0f0f0f0f | 写操作间隔 |

##### 寄存器使用约定
| 寄存器 | 用途 | 生存期 |
|--------|------|--------|
| `s7` | 保存返回地址（`ra`） | 整个函数 |
| `t8` | 内存控制器寄存器基址指针 | 整个函数 |
| `t0`, `t1` | 临时计算和配置值 | 局部计算块 |
| `a0`, `a1` | 参数传递和临时存储 | 函数调用和状态检查 |
| `v0` | 返回值（0=失败，1=成功） | 函数结束前设置 |
| `ra` | 原始返回地址（保存到s7） | 函数入口保存 |

**基址计算**：
```assembly
# 节点基址计算
GET_NODE_ID_a0                 # 获取节点ID到a0
dli     t8, 0x900000000ff00000  # 内存控制器基址
or      t8, a0                 # 加上节点偏移：t8 = 基址 | (节点ID << 44)
```

##### 调试寄存器操作

**1. 调试信息输出（条件编译）**：
```assembly
#ifdef DEBUG_TEST_ENGINE
    PRINTSTR("\r\nddr test engin start\r\n")
#endif
```

**2. 寄存器内容调试输出（被注释）**：
文件中包含详细的调试代码段，可以输出0x1000-0x1068、0x1200-0x1250、0x3100-0x3160三个地址范围的寄存器值，用于调试测试引擎的配置状态。

## 关键算法总结

### 1. 连续窗口检测算法 (`find_continues_1`)
```c
// 寻找向量中连续的"1"比特
输入: 位图向量
输出: [7:0]最大连续长度, [15:8]起始位置, [23:16]中间位置

算法步骤:
1. 初始化最大长度=0, 当前长度=0, 起始位置=0
2. 遍历每个比特位:
   - 如果当前位=1: 当前长度++
   - 如果当前位=0: 
       如果当前长度>最大长度:
           更新最大长度=当前长度
           更新起始位置=当前位置-当前长度
       重置当前长度=0
3. 计算中间位置 = 起始位置 + (最大长度/2)
```

### 2. 时序参数计算算法
```c
// 时间到时钟周期转换
tCK_ps = 1e12 / (DDR_FREQ * 2 * 1e6)  // 频率单位MHz
时钟周期数 = ceil(时间_ps / tCK_ps)

// SPD数据融合
tRFC_total = tRFC_base * 125ps + tRFC_fine * 1ps
tRFC_clocks = int_rounding(tRFC_total, tCK_ps)
```

### 3. VREF训练统计方法
```c
// 五样本统计策略
for (i=0; i<5; i++) {
    采集样本SAMPLE[i]
}
ONE_SAMPLE = SAMPLE[0] | SAMPLE[1] | SAMPLE[2] | SAMPLE[3] | SAMPLE[4]
ZERO_SAMPLE = SAMPLE[0] & SAMPLE[1] & SAMPLE[2] & SAMPLE[3] & SAMPLE[4]

// 通过条件: 同时存在连续CONT1_NUM个1和连续CONT0_NUM个0
```

## 寄存器映射与地址空间

### 内存控制器地址空间布局
```
0x900000000ff00000 + 节点偏移 (节点ID << 44)
├── 0x0000-0x0FFF: PHY寄存器 (物理层控制)
├── 0x1000-0x1FFF: CTL寄存器 (控制器配置)
├── 0x2000-0x2FFF: MON寄存器 (监控状态)
├── 0x3000-0x3FFF: TST寄存器 (测试功能)
└── 0x8000-0x8FFF: 高级PHY寄存器 (阻抗校准等)
```

### 关键寄存器偏移（来自ls3a4000_reg_def.h）
```c
// PHY寄存器
#define PHY_ADDRESS 0x0000
#define DLL_WRDQ_OFFSET         0x0100
#define DLL_WRDQS_OFFSET        0x0101
#define DLL_1XGEN_OFFSET        0x0102
#define DLL_1XDLY_OFFSET        0x0103
#define DLL_RDDQS0_OFFSET       0x0108
#define DLL_RDDQS1_OFFSET       0x0109
#define DLL_GATE_OFFSET         0x010a
#define LVL_MODE_OFFSET         0x0700
#define LVL_REQ_OFFSET          0x0701
#define LVL_DONE_OFFSET         0x0708

// 控制寄存器
#define CTL_ADDRESS 0x1000
#define DDR4_MR0_CS0_REG        0x1180
#define DDR4_MR1_CS0_REG        0x1182
#define DDR4_MR2_CS0_REG        0x1184
#define DDR4_MR3_CS0_REG        0x1186
#define DDR4_MR4_CS0_REG        0x1188
#define DDR4_MR5_CS0_REG        0x118a
#define DDR4_MR6_CS0_REG        0x118c
#define BASE_ADDR_OFFSET        0x1140
#define VALID_BITS_OFFSET       0x1148
#define PAGE_SIZE_OFFSET        0x1150
#define PAGE_NUM_OFFSET         0x1154
```

## 多节点与多控制器支持

### 节点ID计算
```assembly
#define GET_NODE_ID_a0  dli a0, 0x00000003; and a0, s1, a0; dsll a0, 44;
```
- 从s1[1:0]提取节点ID
- 左移44位作为地址偏移量
- 支持最多4个节点 (0-3)

### 内存控制器选择
- **MC0使能**：配置寄存器位30=1, 位31=0
- **MC1使能**：配置寄存器位30=0, 位31=1
- **双控制器使能**：位30=1, 位31=1

### 配置空间管理
```assembly
// 打开MC0配置空间
lui t2, 0x4000  // 位4控制MC0配置空间
xor t1, t1, t2  // 翻转位4

// 打开MC1配置空间  
lui t2, 0x200   // 位9控制MC1配置空间
xor t1, t1, t2  // 翻转位9
```

## 调试与错误处理

### 调试输出机制
```assembly
#ifdef DEBUG_PROBE_NODE_DIMM
    PRINTSTR("\r\nDebug info: ...")
    hexserial(a0)  // 打印寄存器值
#endif
```

### 错误处理类型
1. **DIMM类型不匹配**：双插槽DIMM类型不同，仅使用插槽0
2. **不支持的类型**：打印错误信息，标记为无DIMM
3. **时序参数越界**：行/列宽度超过支持范围，报错终止
4. **训练失败**：VREF训练找不到有效窗口，使用默认值
5. **测试失败**：test_engine返回v0=0，表示内存测试失败

### 错误寄存器读取
```assembly
// 读取测试错误模式
ld t0, ERROR_REG_OFFSET(t8)
// 错误模式可能包含:
// - 错误地址
// - 错误数据模式
// - 错误类型编码
```

## 性能优化特点

### 1. 硬件辅助训练
- 使用PHY内置的电平检测电路
- 硬件测试引擎加速内存验证
- DLL数字延迟线提供精细时序调节

### 2. 统计优化算法
- 多样本统计提高训练可靠性
- 连续窗口检测确保时序裕量
- 重复性过滤排除偶发性错误

### 3. 自适应参数范围
- 根据DIMM宽度动态确定数据片数量
- 根据DLL_VALUE_OFFSET调整扫描范围
- 支持不同频率的速度分级

### 4. 并行处理能力
- 多数据片并行训练
- 支持双内存控制器独立配置
- 多节点系统统一地址映射

## 初始化流程图

```
系统启动
    ↓
PMON引导
    ↓
内存初始化开始
    ↓
├── 阶段1: 硬件探测
│   ├── detect_node_dimm_all.S: 读取SPD信息
│   ├── detect_channel_dimm.S: 通道DIMM检测
│   └── config_ddr_capacity_tmp.S: 容量计算
│
├── 阶段2: 寄存器配置
│   ├── ddr4_register_param.S: 加载静态参数
│   ├── mc_config.S: 动态时序计算
│   └── mc_vref_set.S: VREF基础设置
│
├── 阶段3: 时序训练
│   ├── ddr4_leveling.S: 写/门电平校准
│   ├── mc_vref_training.S: VREF训练
│   ├── wr_bit_training.S: 写位训练
│   └── rd_bit_training.S: 读位训练
│
└── 阶段4: 测试验证
    └── test_engin.S: 内存功能测试
        ↓
测试通过 → 内存可用
测试失败 → 错误处理/重试
```

## 总结

龙芯3A4000平台的DDR4内存控制器初始化是一个复杂但高度系统化的过程，具有以下特点：

1. **全自动配置**：通过SPD读取实现免手动配置，支持不同类型DIMM混插
2. **多层次训练**：从电平校准到位训练，逐步优化时序参数
3. **硬件软件协同**：充分利用硬件辅助功能，软件实现智能算法
4. **鲁棒性设计**：多重错误检测和恢复机制，确保系统稳定性
5. **可扩展架构**：支持多节点、多控制器、不同内存拓扑

该初始化流程体现了现代内存控制器设计的核心理念：通过系统性的训练和校准，使硬件适应具体的物理环境，在标准规范的基础上实现最佳的实际性能。