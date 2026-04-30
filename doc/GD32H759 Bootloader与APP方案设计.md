# GD32H759 Bootloader 与 APP 方案设计

## 与前版的核心变化

| 项目            | 旧方案                   | 新方案                              | 原因                              |
|---------------|-----------------------|----------------------------------|---------------------------------|
| BSP 共享        | Shared BSP 分区 + 函数向量表 | 去掉，Bootloader 自包含                | 向量表机制脆弱，BSP 升级失败无法恢复            |
| 暂存区位置         | 片内 Flash 1600 KB      | QSPI Flash 4 MB                  | 板载 8 MB QSPI 闲置，片内 Flash 全给 APP |
| APP 上限        | 1600 KB               | **3584 KB**                      | 暂存区外移后的收益                       |
| 参数区格式         | 一扇区一条记录（8 KB/条）       | 紧凑条目（64 B/条，128 条/扇区）            | 原方案空间利用率 < 1%                   |
| Bootloader 驱动 | 依赖 Shared BSP         | 自包含 UART0 + OSPI + Flash + CRC32 | 去掉依赖，降低变砖风险                     |
| 跳转细节          | 未描述                   | Cache / NVIC / MPU 完整清理          | 缺失会导致 APP 启动随机故障                |

---

## 一、存储资源

| 存储         | 容量      | 起始地址       | 用途                     |
|------------|---------|------------|------------------------|
| 片内 Flash   | 3840 KB | 0x08000000 | Bootloader + 参数区 + APP |
| QSPI Flash | 8 MB    | 0x90000000 | 固件暂存（4 MB）+ 资源文件（4 MB） |
| AXI SRAM   | 512 KB  | 0x24000000 | 主运行内存                  |
| 共享 SRAM    | 512 KB  | 0x20000000 | ITCM/DTCM/AXI 可配置      |
| SDRAM      | 32 MB   | 0xC0000000 | 大容量运行缓冲                |

---

## 二、Flash 分区

### 2.1 片内 Flash（3840 KB）

```
0x08000000 ┌──────────────────────────────────┐
           │  Bootloader          128 KB       │  扇区 0 ~ 15
           │  · 自包含：UART0 / OSPI / CRC32   │
           │  · 校验 / 搬移 / 跳转              │
0x08020000 ├──────────────────────────────────┤
           │  参数区              128 KB        │  扇区 16 ~ 31
           │  · 64 B/条，128 条/扇区            │
           │  · 日志循环写入，磨损均衡            │
0x08040000 ├──────────────────────────────────┤
           │  APP 运行区         3584 KB        │  扇区 32 ~ 479
           │  · 唯一运行地址 0x08040000         │
           │  · 向量表后 0x400 处嵌入固件信息    │
0x083C0000 └──────────────────────────────────┘
```

| 分区         | 起始地址       | 大小      | 扇区数 |
|------------|------------|---------|-----|
| Bootloader | 0x08000000 | 128 KB  | 16  |
| 参数区        | 0x08020000 | 128 KB  | 16  |
| APP 运行区    | 0x08040000 | 3584 KB | 448 |

### 2.2 QSPI Flash（8 MB）

```
0x90000000 ┌──────────────────────────────────┐
           │  固件暂存区           4 MB         │
           │  · 固件包格式：64 B 头 + bin 数据  │
           │  · OSPI 内存映射直接读取           │
0x90400000 ├──────────────────────────────────┤
           │  资源文件区           4 MB         │
           │  · 字体 / 图片 / 配置             │
0x90800000 └──────────────────────────────────┘
```

> APP 始终编译到 0x08040000。升级包写入 QSPI 暂存区后由 Bootloader 搬移到运行区，只需一套链接脚本，只需一个 .bin 文件。

---

## 三、参数区——紧凑日志条目

### 3.1 条目格式（64 字节/条）

```c++
#define PARAM_ENTRY_MAGIC    0xA55AU   /* 有效 */
#define PARAM_ENTRY_EMPTY    0xFFFFU   /* 未写入（Flash 擦除后默认值） */
#define PARAM_ENTRY_INVALID  0x0000U   /* 已废弃（写 0 标记，无需擦除扇区） */

#define PARAM_ENTRY_SIZE         64U
#define PARAM_AREA_BASE          0x08020000U
#define PARAM_AREA_SECTORS       16U
#define PARAM_SECTOR_SIZE        8192U
#define PARAM_ENTRIES_PER_SECTOR (PARAM_SECTOR_SIZE / PARAM_ENTRY_SIZE)   /* 128 */
#define PARAM_TOTAL_ENTRIES      (PARAM_AREA_SECTORS * PARAM_ENTRIES_PER_SECTOR) /* 2048 */
#define PARAM_DATA_MAX           52U

typedef enum {
    PARAM_TYPE_BOOT_INFO = 0,  /* 启动计数 / 模式 */
    PARAM_TYPE_APP_INFO  = 1,  /* 当前运行固件信息 */
    PARAM_TYPE_PREV_INFO = 2,  /* 上一版本固件信息（回退用） */
    PARAM_TYPE_UPG_FLAG  = 3,  /* 升级标志 */
    PARAM_TYPE_MAX       = 4,
} param_type_t;

typedef struct __attribute__((packed, aligned(4))) {
    uint16_t magic;                /* PARAM_ENTRY_MAGIC / EMPTY / INVALID */
    uint8_t  type;                 /* param_type_t */
    uint8_t  len;                  /* data 有效字节数（<= PARAM_DATA_MAX） */
    uint32_t seq;                  /* 全局递增序列号 */
    uint32_t crc32;                /* data[0..len-1] 的 CRC32 */
    uint8_t  data[PARAM_DATA_MAX]; /* 载荷 */
} param_entry_t;                   /* sizeof = 64 字节 */
```

### 3.2 数据载荷结构体

```c++
/* PARAM_TYPE_BOOT_INFO */
typedef struct {
    uint32_t boot_cnt;     /* Bootloader 已决定跳转但 APP 未确认成功的次数 */
    uint32_t last_reset;   /* 上次复位原因（读自 RCU 复位标志寄存器） */
} param_boot_info_t;       /* 8 字节 */

/* PARAM_TYPE_APP_INFO / PARAM_TYPE_PREV_INFO */
typedef struct {
    uint32_t version;      /* (major<<24)|(minor<<16)|patch */
    uint32_t build_time;   /* Unix 时间戳 */
    uint32_t image_size;   /* APP 镜像字节数 */
    uint32_t image_crc32;  /* APP 镜像 CRC32 */
    uint8_t  git_hash[8];  /* git short hash */
} param_app_info_t;        /* 24 字节 */

/* PARAM_TYPE_UPG_FLAG */
typedef struct {
    uint32_t upgrade_request; /* 1 = QSPI 暂存区有待烧录的固件 */
    uint32_t staging_size;    /* 固件包总大小（含 64 字节头） */
    uint32_t staging_crc32;   /* 固件包（仅 bin 数据部分）CRC32 */
    uint8_t  source;          /* 0=SD, 1=USB, 2=UART, 3=ETH */
    uint8_t  reserved[3];
} param_upg_flag_t;           /* 16 字节 */
```

### 3.3 写入

追加写：扫描找到第一个 `magic == EMPTY` 的槽位直接写，写满后触发 GC。

```c++
bool param_write(param_type_t type, const void *data, uint8_t len)
{
    if (len > PARAM_DATA_MAX) return false;

    /* 扫描：找到第一个空槽，同时记录最大 seq */
    uint32_t max_seq   = 0;
    int      free_slot = -1;

    for (int i = 0; i < (int)PARAM_TOTAL_ENTRIES; i++) {
        const param_entry_t *e = (const param_entry_t *)
            (PARAM_AREA_BASE + (uint32_t)i * PARAM_ENTRY_SIZE);

        if (e->magic == PARAM_ENTRY_EMPTY) {
            if (free_slot < 0) free_slot = i;
        } else if (e->magic == PARAM_ENTRY_MAGIC) {
            if (e->seq > max_seq) max_seq = e->seq;
        }
    }

    if (free_slot < 0) {
        free_slot = param_gc(&max_seq);  /* GC 后返回空槽索引 */
        if (free_slot < 0) return false;
    }

    param_entry_t e = {
        .magic = PARAM_ENTRY_MAGIC,
        .type  = (uint8_t)type,
        .len   = len,
        .seq   = max_seq + 1U,
        .crc32 = crc32_calc(data, len),
    };
    memcpy(e.data, data, len);

    uint32_t addr = PARAM_AREA_BASE + (uint32_t)free_slot * PARAM_ENTRY_SIZE;
    return flash_write_words(addr, &e, sizeof(e));
}
```

### 3.4 读取

```c++
bool param_read(param_type_t type, void *buf, uint8_t expected_len)
{
    uint32_t          best_seq = 0;
    const param_entry_t *best = NULL;

    for (int i = 0; i < (int)PARAM_TOTAL_ENTRIES; i++) {
        const param_entry_t *e = (const param_entry_t *)
            (PARAM_AREA_BASE + (uint32_t)i * PARAM_ENTRY_SIZE);

        if (e->magic != PARAM_ENTRY_MAGIC)  continue;
        if (e->type  != (uint8_t)type)      continue;
        if (e->len   != expected_len)       continue;
        if (e->seq   <= best_seq)           continue;
        if (crc32_calc(e->data, e->len) != e->crc32) continue;

        best_seq = e->seq;
        best     = e;
    }

    if (!best) return false;
    memcpy(buf, best->data, expected_len);
    return true;
}
```

### 3.5 垃圾回收（GC）

GC 触发条件：2048 个槽全部写满（包含有效和已废弃）。

```c++
/* 返回 GC 后第一个可用槽的索引；max_seq 更新为当前最大值 */
static int param_gc(uint32_t *max_seq)
{
    /* 1. 找出每种 type 最新的有效条目 */
    const param_entry_t *keep[PARAM_TYPE_MAX] = {NULL};
    uint32_t             keep_seq[PARAM_TYPE_MAX] = {0};

    for (int i = 0; i < (int)PARAM_TOTAL_ENTRIES; i++) {
        const param_entry_t *e = (const param_entry_t *)
            (PARAM_AREA_BASE + (uint32_t)i * PARAM_ENTRY_SIZE);

        if (e->magic != PARAM_ENTRY_MAGIC)             continue;
        if (e->type  >= PARAM_TYPE_MAX)                continue;
        if (crc32_calc(e->data, e->len) != e->crc32)  continue;
        if (e->seq <= keep_seq[e->type])               continue;

        keep_seq[e->type] = e->seq;
        keep[e->type]     = e;
    }

    /* 2. 把保留条目暂存到栈上（最多 PARAM_TYPE_MAX 条，每条 64 字节） */
    param_entry_t saved[PARAM_TYPE_MAX];
    int           save_cnt = 0;
    uint32_t      new_seq  = *max_seq;

    for (int t = 0; t < PARAM_TYPE_MAX; t++) {
        if (!keep[t]) continue;
        saved[save_cnt] = *keep[t];
        saved[save_cnt].seq = ++new_seq;
        save_cnt++;
    }
    *max_seq = new_seq;

    /* 3. 擦除全部参数扇区 */
    for (int s = 0; s < (int)PARAM_AREA_SECTORS; s++) {
        uint32_t addr = PARAM_AREA_BASE + (uint32_t)s * PARAM_SECTOR_SIZE;
        if (!flash_erase_sector(addr)) return -1;
    }

    /* 4. 把保留条目写回到扇区 0 开头 */
    for (int i = 0; i < save_cnt; i++) {
        uint32_t addr = PARAM_AREA_BASE + (uint32_t)i * PARAM_ENTRY_SIZE;
        if (!flash_write_words(addr, &saved[i], sizeof(saved[i]))) return -1;
    }

    /* 5. 返回第一个空槽索引 */
    return save_cnt;
}
```

### 3.6 磨损均衡分析

```
16 个扇区 × 128 条/扇区 = 2048 个槽
GC 时保留 ≤ 4 条（每种类型各一条），清空全部 16 个扇区后重写

每次 GC 之前：2048 次写操作（含重写的 4 条，净增 2044 次）
每次 GC：擦除 16 个扇区，每扇区写回 ≤ 1 条

扇区平均擦写寿命：10 万次
每次 GC 平均每扇区擦写 1 次
总 GC 次数上限：10 万次（每扇区）× 16 扇区 / 16 扇区/次 GC = 10 万次 GC
可支撑写操作总量：10 万 × 2044 ≈ 2 亿次写操作

实际场景（每次启动写 2 次，每天 10 次启动）：
2 亿 / 20 次/天 / 365 天 = 27,397 年
```

---

## 四、固件信息 Section（嵌入 APP 镜像）

Bootloader 在搬移完成后、跳转前需要验证固件有效性，同时 APP 也需要知道自己的版本号。在 APP 链接脚本中定义一个固定偏移的 `.fw_info` section，使双方都能直接访问。

Cortex-M7 向量表最大占用 `(16 + 240) × 4 = 1024 = 0x400` 字节，固件信息放在 `APP_BASE + 0x400`。

### 4.1 结构体定义（共享头文件）

```c++
#define APP_BASE         0x08040000U
#define FW_INFO_MAGIC    0x47443748U  /* "GD7H" */
#define FW_INFO_ADDR     (APP_BASE + 0x400U)

typedef struct __attribute__((packed)) {
    uint32_t magic;        /* FW_INFO_MAGIC                             */
    uint32_t version;      /* (major<<24)|(minor<<16)|patch             */
    uint32_t image_size;   /* APP 镜像字节数（post-build 脚本填充）       */
    uint32_t image_crc32;  /* APP 镜像 CRC32（post-build 脚本填充）       */
    uint32_t build_time;   /* __TIME__ / __DATE__ 转 Unix 时间戳         */
    uint8_t  git_hash[8];  /* git rev-parse --short HEAD                */
    uint32_t hw_id;        /* 0x00000001 = GD32H759IMK6                 */
    uint32_t reserved;
    uint32_t info_crc32;   /* 上述所有字段（不含 info_crc32 本身）的 CRC32 */
} fw_info_t;               /* 48 字节 */
```

### 4.2 APP 中定义

```c++
/* app/src/fw_info.c */
__attribute__((section(".fw_info"), used))
const fw_info_t g_fw_info = {
    .magic      = FW_INFO_MAGIC,
    .version    = (MAJOR << 24) | (MINOR << 16) | PATCH,
    .image_size = 0U,          /* post-build 脚本用 objcopy 填充 */
    .image_crc32= 0U,          /* post-build 脚本填充 */
    .build_time = BUILD_TIME,
    .git_hash   = GIT_HASH,
    .hw_id      = 0x00000001U,
    .reserved   = 0U,
    .info_crc32 = 0U,          /* post-build 脚本填充 */
};
```

### 4.3 post-build 脚本（CMake）

```cmake
# 生成 .bin 后填充 image_size / image_crc32 / info_crc32
add_custom_command(TARGET app POST_BUILD
    COMMAND ${OBJCOPY} -O binary $<TARGET_FILE:app> app.bin
    COMMAND ${PYTHON} ${CMAKE_SOURCE_DIR}/scripts/fill_fw_info.py
            --bin app.bin
            --fw-info-offset 0x400   # APP_BASE 内偏移
    COMMENT "Filling firmware info fields"
)
```

---

## 五、Bootloader 设计

### 5.1 自包含驱动集

Bootloader 不依赖任何外部共享库，内置以下最小驱动：

| 驱动               | 功能          | 说明                          |
|------------------|-------------|-----------------------------|
| PMU + Clock      | 基础时钟初始化     | SMPS → PLL → 600 MHz，必须最先执行 |
| UART0 (PA9/PA10) | 日志输出 + 紧急恢复 | 最小调试通道                      |
| 片内 Flash         | 扇区擦除 / 字写入  | 搬移固件用                       |
| OSPI（内存映射模式）     | 读取 QSPI 暂存区 | 0x90000000 直接访问             |
| CRC32            | 固件校验        | 优先使用 CRC 硬件加速单元             |
| IWDG             | 看门狗         | 防止 Bootloader 自身挂死          |

Bootloader 不包含：LCD、触摸屏、以太网、USB、SD 卡、DHT11 等（由 APP 的 BSP 负责）。

### 5.2 启动流程

```
上电 / 复位
    │
    ▼
【最小初始化】
  bsp_pmu_config()     ← SMPS 配置（必须在 PLL 前）
  bsp_clock_config()   ← HXTAL 25MHz → PLL → 600 MHz
  bsp_uart0_init()     ← PA9/PA10，115200，8N1
  bsp_iwdg_init()      ← 看门狗 5 秒超时
  bsp_ospi_mmap_init() ← QSPI 内存映射，0x90000000 可读
    │
    ▼
param_read(PARAM_TYPE_UPG_FLAG, &upg)
    │
    ├── upgrade_request == 1 ──────────────────────────────→ 【升级流程】
    │
    ▼
param_read(PARAM_TYPE_BOOT_INFO, &boot)
    │
    ├── boot.boot_cnt >= 3 ─────────────────────────────────→ 【恢复模式】
    │
    ▼
【正常跳转】
  param_write(PARAM_TYPE_BOOT_INFO, {.boot_cnt = boot.boot_cnt + 1})
  boot_jump_to_app(APP_BASE)
```

### 5.3 升级流程（Bootloader 侧）

```
【升级流程】
    │
    ▼
解析 QSPI 固件包头（0x90000000，64 字节）
    ├── magic 不匹配 / header_crc 错误 → uart_log("BAD HEADER") → 清除 upgrade_request → 复位
    │
    ▼
校验固件包数据：CRC32(0x90000040, upg.staging_size - 64) == pkg_hdr.image_crc32 ？
    ├── 不匹配 → uart_log("CRC FAIL") → 清除 upgrade_request → 复位
    │
    ▼
搬移循环（app_size / 8192 个扇区，app_size = pkg_hdr.image_size）：
    for (i = 0; i < num_sectors; i++) {
        src = 0x90000040 + i * 8192;       /* QSPI 内存映射直接读 */
        dst = APP_BASE  + i * 8192;
        flash_erase_sector(dst);
        flash_write_words(dst, src, 8192);
        iwdg_feed();
        uart_log(".");                      /* 每扇区打一个点 */
    }
    │
    ▼
全量校验运行区：CRC32(APP_BASE, pkg_hdr.image_size) == pkg_hdr.image_crc32 ？
    ├── 不匹配 → uart_log("MOVE FAIL") → 保留 upgrade_request → 复位（下次重新搬移）
    │
    ▼
读取运行区 fw_info（APP_BASE + 0x400）并校验 info_crc32
    │
    ▼
param_write(PARAM_TYPE_APP_INFO,  从 fw_info 构造的 param_app_info_t)
param_write(PARAM_TYPE_UPG_FLAG,  {.upgrade_request = 0})
param_write(PARAM_TYPE_BOOT_INFO, {.boot_cnt = 0})
    │
    ▼
uart_log("UPGRADE OK, RESET")
NVIC_SystemReset()
```

### 5.4 恢复模式（boot_cnt >= 3）

```
【恢复模式】
    │
    ▼
uart_log("RECOVERY MODE: waiting for firmware via UART0 (YMODEM)")
    │
    ▼
等待 UART0 YMODEM 传输：
  · 接收固件包（64 字节头 + bin）
  · 写入 QSPI 暂存区
  · 写 upgrade_request = 1
  · 复位 → 走升级流程
    │
    （超时或传输失败继续等待）
```

### 5.5 跳转实现（关键细节）

```c++
static void boot_jump_to_app(uint32_t app_base)
{
    /* 1. 验证向量表：MSP 应指向有效 SRAM 区域 */
    uint32_t msp = *(volatile uint32_t *)app_base;
    bool msp_in_axisram  = (msp - 0x24000000U) < 0x00080000U;  /* 512 KB AXI SRAM */
    bool msp_in_sharedsram = (msp - 0x20000000U) < 0x00080000U; /* 512 KB 共享 SRAM */
    if (!msp_in_axisram && !msp_in_sharedsram) {
        uart_log("INVALID MSP, ABORT\r\n");
        return;
    }

    /* 2. 停用 SysTick */
    SysTick->CTRL = 0U;
    SysTick->LOAD = 0U;
    SysTick->VAL  = 0U;

    /* 3. 关闭 IWDG（IWDG 一旦启动无法软件关闭，保持喂狗直到 APP 接管） */
    /* APP 的 bsp_system 初始化完成后自行接管 IWDG */

    /* 4. 清除所有 NVIC 使能和挂起 */
    for (int i = 0; i < 8; i++) {
        NVIC->ICER[i] = 0xFFFFFFFFU;
        NVIC->ICPR[i] = 0xFFFFFFFFU;
    }
    __DSB();
    __ISB();

    /* 5. 清理并禁用 D-Cache（防止 APP 看到脏数据）*/
    SCB_CleanDCache();
    SCB_DisableDCache();
    SCB_DisableICache();

    /* 6. 禁用 MPU（APP 会重新配置 SDRAM / ENET 区域）*/
    MPU->CTRL = 0U;
    __DSB();
    __ISB();

    /* 7. 设置 APP 向量表偏移 */
    SCB->VTOR = app_base;
    __DSB();

    /* 8. 切换 MSP 并跳转 Reset_Handler */
    uint32_t reset_addr = *(volatile uint32_t *)(app_base + 4U);
    __set_MSP(msp);
    __DSB();
    __ISB();
    ((void (*)(void))reset_addr)();

    /* 永远不会到达此处 */
    while (1) {}
}
```

---

## 六、APP 侧职责

### 6.1 启动时确认（在 main() 业务循环开始前）

```c++
int main(void)
{
    /* BSP 初始化（时钟、外设、IWDG 接管） */
    bsp_init();

    /* 确认启动成功：清零 boot_cnt */
    param_boot_info_t boot = {0};
    param_write(PARAM_TYPE_BOOT_INFO, &boot, sizeof(boot));

    /* 正常业务 */
    app_run();
}
```

### 6.2 OTA 下载（APP 侧 Phase 1）

```
APP 收到 OTA 触发（SD 卡 / USB / ETH / UART）
    │
    ▼
解析固件包头：magic / hw_id / version 检查
    ├── 版本不高于当前 → 提示用户，不继续
    │
    ▼
初始化 OSPI 为写入模式（退出内存映射，进入 SPI 命令模式）
    │
    ▼
擦除 QSPI 暂存区（前 4 MB，即扇区 0 ~ N）
逐块接收并写入（建议 4 KB/块，边写边喂看门狗）
    │
    ▼
全量 CRC32 校验 QSPI 暂存区数据部分
    ├── 校验失败 → 重试，最多 3 次，仍失败则放弃
    │
    ▼
保留当前 APP 信息到 PREV_INFO（用于日志记录，单槽暂存无法真回退）
param_write(PARAM_TYPE_PREV_INFO, &current_app_info)
    │
    ▼
param_write(PARAM_TYPE_UPG_FLAG, {
    .upgrade_request = 1,
    .staging_size    = pkg_total_size,
    .staging_crc32   = image_crc32,
    .source          = src,
})
    │
    ▼
NVIC_SystemReset()
```

### 6.3 boot_cnt 语义

| 状态               | boot_cnt 值 | 说明                    |
|------------------|------------|-----------------------|
| 全新 / 刚确认成功       | 0          | APP 在 main() 开头写 0    |
| Bootloader 已决定跳转 | 前值 + 1     | Bootloader 跳转前递增      |
| APP 连续崩溃 2 次     | 2          | 下次跳转后变 3              |
| boot_cnt >= 3    | 3          | Bootloader 不再跳转，进恢复模式 |

---

## 七、容错分析

| 场景                         | 后果                   | 恢复机制                            |
|----------------------------|----------------------|---------------------------------|
| QSPI 下载中断电                 | 暂存区不完整，CRC 不匹配       | upgrade_request 未写，正常启动         |
| 参数区写 upgrade_request 时断电   | 条目 CRC 不匹配           | 读取时跳过，upgrade_request 视为 0，正常启动 |
| Bootloader 搬移中断电（运行区已擦除）   | 运行区 CRC 不匹配          | upgrade_request 仍为 1，下次上电重新搬移   |
| APP 早期崩溃（boot_cnt 未清零）     | 连续 3 次后进恢复模式         | UART0 YMODEM 重新烧录               |
| APP 运行区 CRC 损坏（写 Flash 异常） | boot_cnt 连续递增到 3     | 进恢复模式，重新烧录                      |
| QSPI 暂存区损坏                 | 无法升级                 | SWD 重新烧录 QSPI                   |
| 参数区全部损坏                    | boot_cnt 读为 0，正常跳转   | 无影响（固件本身完好）                     |
| Bootloader 自身崩溃 / IWDG 超时  | 硬件复位，重新进入 Bootloader | IWDG 设 5 秒，给足时间搬移               |
| 无法恢复                       | 变砖                   | SWD 重新烧录全部分区                    |

---

## 八、链接脚本

### Bootloader

```ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 128K
    RAM   (rwx) : ORIGIN = 0x24000000, LENGTH = 512K
}

SECTIONS {
    .isr_vector : { KEEP(*(.isr_vector)) }                 > FLASH
    .text       : { *(.text*) *(.rodata*) }                > FLASH
    .data       : { *(.data*) } AT > FLASH                 > RAM
    .bss        : { *(.bss*)  COMMON }                     > RAM
    _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```

### APP

```ld
MEMORY {
    FLASH (rx)  : ORIGIN = 0x08040000, LENGTH = 3584K
    RAM   (rwx) : ORIGIN = 0x24000000, LENGTH = 512K
}

SECTIONS {
    .isr_vector : {
        KEEP(*(.isr_vector))
        . = ALIGN(0x400);   /* 向量表最大 1024 字节，确保 fw_info 在固定偏移 */
    } > FLASH

    .fw_info : {
        KEEP(*(.fw_info))   /* g_fw_info 位于 0x08040400 */
        . = ALIGN(16);
    } > FLASH

    .text       : { *(.text*) *(.rodata*) }                > FLASH
    .data       : { *(.data*) } AT > FLASH                 > RAM
    .bss        : { *(.bss*)  COMMON }                     > RAM
    _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```

---

## 九、固件包格式

```
┌─────────────────────────────────────────┐  ← 0x00（QSPI 暂存区起始）
│  Package Header  64 字节                 │
│  magic[4]        "GD7H" (0x47443748)    │
│  header_ver      1                      │
│  header_size     64                     │
│  fw_version      (major<<24|minor<<16|patch) │
│  fw_type         0=APP                  │
│  hw_id           0x00000001 (H759)      │
│  image_size      bin 数据字节数          │
│  image_crc32     bin 数据 CRC32          │
│  build_time      Unix 时间戳             │
│  git_hash[8]                            │
│  source_hint     保留，填 0              │
│  reserved[3]                            │
│  header_crc32    前 60 字节的 CRC32      │
├─────────────────────────────────────────┤  ← 0x40
│  APP .bin 数据（image_size 字节）         │
│  · 0x00 起为 APP 向量表（MSP + 向量）    │
│  · 0x400 起为 fw_info_t                 │
│  · 0x400 后为 .text / .rodata           │
└─────────────────────────────────────────┘
```

Bootloader 搬移时从偏移 0x40 开始读取，写入 APP_BASE（0x08040000）。

---

## 十、升级源优先级

| 来源    | 通道     | 速度（参考）   | 触发方式                             |
|-------|--------|----------|----------------------------------|
| SD 卡  | SDIO0  | ~25 MB/s | 插入含 `firmware.pkg` 的 SD 卡，APP 检测 |
| USB   | USBHS0 | ~60 MB/s | 上位机 USB DFU 工具                   |
| 以太网   | RMII   | ~10 MB/s | HTTP OTA 服务器推送                   |
| UART0 | USART0 | ~11 KB/s | Bootloader 恢复模式 YMODEM           |

恢复模式仅 UART0 可用（Bootloader 内置驱动），其余升级通道均由 APP 完成。

---

## 十一、项目目录结构（参考）

```
GD32H759/
├── bootloader/
│   ├── CMakeLists.txt
│   ├── ld/bootloader.ld
│   └── src/
│       ├── main.c            ← Bootloader 入口
│       ├── boot_jump.c       ← boot_jump_to_app()
│       ├── boot_upgrade.c    ← 搬移、校验
│       ├── boot_recovery.c   ← YMODEM 恢复模式
│       ├── drv_flash.c       ← 片内 Flash 操作
│       ├── drv_ospi_mmap.c   ← QSPI 内存映射初始化
│       ├── drv_uart0.c       ← UART0 最小实现
│       └── drv_crc32.c       ← CRC32（优先 HW 加速）
├── app/
│   ├── CMakeLists.txt
│   ├── ld/app.ld
│   └── src/
│       ├── main.c
│       ├── fw_info.c         ← g_fw_info 定义（.fw_info section）
│       └── ota/
│           ├── ota_manager.c ← OTA 下载、写 QSPI、写参数区、复位
│           └── ota_ymodem.c  ← YMODEM 协议
├── common/
│   ├── fw_info.h             ← fw_info_t（bootloader/app 共享）
│   ├── param.h / param.c     ← 参数区读写（bootloader/app 共用）
│   └── crc32.h / crc32.c
└── scripts/
    └── fill_fw_info.py       ← post-build：填充 image_size/crc32/info_crc32
```
