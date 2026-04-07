# **OBC**

## --------------------------------------------------------------------------------------------

# 板子启动流程：

```
上电 → ROM code → 读取启动头 → 加载 SPL → SPL 初始化DDR → U-Boot  → kernel → rootfs → 启动用户空间init 进程
```




### 第 0 阶段：BootROM (MaskROM) —— 硬件固化的“本能”

- **位置**：芯片内部固化的代码，不可修改。
- **动作**：
  1. CPU 上电复位，第一个执行的程序。
  2. 检测启动引脚（EMMC、SD、USB、SPI Flash）。
  3. 从存储介质的 **第 64 扇区（32KB 偏移处）** 加载第一份代码：idbloader.img。
  4. 将其加载到内部 **SRAM** 中运行。

### 第 1 阶段：TPL (Tiny Program Loader) —— 唤醒内存

- **位置**：idbloader.img 的前半部分（通常是瑞芯微提供的闭源 ddr.bin）。
- **动作**：
  - **核心任务：初始化 DDR 内存。**
  - 因为 SRAM 空间极小（几百 KB），装不下完整的 U-Boot。TPL 负责把外部的 LPDDR4/DDR4 内存初始化好。
  - 内存一旦就绪，TPL 将下一棒 SPL 加载到 **DDR** 中运行。

### 第 2 阶段：SPL (Secondary Program Loader) —— 搬运工

- **位置**：idbloader.img 的后半部分。
- **动作**：
  - 此时程序已经在 DDR 运行，空间变大。
  - SPL 负责从存储介质中寻找并加载 **trust.img** (包含 ATF) 和 **uboot.img**。
  - **关键点**：它不仅搬运，还会通过哈希校验确保镜像没被篡改（Secure Boot）。

### 第 3 阶段：ATF (ARM Trusted Firmware) —— 核心枢纽 (BL31)

- **这是 ARMv8 启动的灵魂，面试必考。**
- **位置**：trust.img 中的 **BL31** 固件。
- **权限级别**：**EL3 (最高权限)**。
- **动作**：
  1. 初始化系统安全服务、中断控制器 (GIC)。
  2. **电源管理服务 (PSCI)**：以后 Linux 想要关机、休眠、调频，都要通过系统调用向 ATF 请求权限。
  3. **身份切换**：这是最关键的一步。ATF 准备好环境后，执行一个“异常返回”指令，人为地将 CPU 权限从 **EL3 降级到 EL2**，然后跳转到 U-Boot 运行。

### 第 4 阶段：U-Boot (BL33) —— 系统管家

- **权限级别**：**EL2 (非安全模式/虚拟机模式)**。
- **动作**：
  - 这部分你很熟悉了：初始化串口、显示 Logo、扫描 USB、加载 Kernel 和 DTB 到内存。
  - **跨代区别**：在 RK3566 这种 64 位系统上，U-Boot 加载的是 **Image**（解压后的内核镜像），而不是 i.MX6ULL 常用的 zImage（自解压镜像）。
  - 最后，U-Boot 再次执行降级，将 CPU 从 **EL2 降级到 EL1**，跳转到内核入口。

### 第 5 阶段：Linux Kernel —— 主人

- **权限级别**：**EL1 (内核态)**。
- **动作**：
  - 内核解开 DTB，挂载 Rootfs，启动 /sbin/init，进入用户态 (**EL0**)。





### 关于uboot阶段细节

（**bootcmd**：是给 **U-Boot** 用的。它规定了 U-Boot **如何把内核跑起来**（比如去哪读文件、读到哪）。

**bootargs**：是给 **Linux 内核** 用的。它规定了内核 **启动后的行为**（比如控制台串口是谁、根文件系统在哪、内存限制多少）。它告诉内核的是eMMC上的地址和大小，需要与我们当时烧录时候的地址分布一致

举例：bootargs console=ttymxc0,115200 root=/dev/mmcblk1p5 rootwait rw blkdevparts=mmcblk1:512K@0x80000(fdt0),512K@0x100000(fdt1),16M@0x200000(kernel0),16M@0x1200000(kernel1),64M@0x2200000(rootfs0),

64M@0x6200000(rootfs1),64M@0xa200000(appfs0),64M@0xe200000(appfs1)

1.console=ttymxc0,115200  ---> 告诉使用ttymxc0串口 波特率为115200

2.root=/dev/mmcblk1p5 --->当前主系统为rootfs1

3.rootwait ---->等待指令

4.rw  读写权限

5.blkdevparts=mmcblk1:  ------>告诉内核第1号 eMMC 设备里地址分区的情况

### 两套系统 （作用1.如果主系统（A）崩了，备份系统（B）立刻顶上 2.在线升级后 (版本交替) 

**套装 A**：fdt0 + kernel0 + rootfs0

**套装 B**：fdt1 + kernel1 + rootfs1

### U-Boot 是如何决定跳 A 还是跳 B 的？

U-Boot 内部通常维护着几个“魔术变量”：

- **boot_slot**：记录当前应该启动 0 还是 1。

- **boot_count**：记录启动失败的次数。

  如果启动 rootfs1 时，连续 3 次在 1 分钟内重启（说明新系统崩溃了），U-Boot 会自动把 boot_slot 改回 0，实现**自动回滚**。

备注提问：如何切换系统？

1.root=/dev/mmcblk1p5 修改为root=/dev/mmcblk1p4 对应rootfs0

2.在**bootcmd**里配合修改地址 

```
# 从 0x200000 (kernel0) 读 16MB 到内存
mmc read 0x80800000 0x1000 0x8000 
# 从 0x80000 (fdt0) 读 512KB 到内存
mmc read 0x83000000 0x400 0x400
# 设置 root 指向 p4
setenv bootargs "... root=/dev/mmcblk1p4 ..."
# 启动
bootz 0x80800000 - 0x83000000
```



备注提问：uboot是如何知道要去eMMC的哪里找zImage和DTB，又如何知道该放到RAM的哪里？

回答：当你把上述两者结合起来，就形成了 bootcmd。以下是一个真实的 i.MX6 启动流程解析：

code

```
# U-Boot 内部执行的一行指令：
setenv bootcmd "fatload mmc 0:1 ${loadaddr} zImage; fatload mmc 0:1 ${fdt_addr} imx6.dtb; bootz ${loadaddr} - ${fdt_addr}"
```

**拆解逻辑：**

1. **第一步**：去 eMMC（mmc 0:1）找名为 zImage 的文件，把它搬运到内存地址 ${loadaddr} (即 0x80800000)。
2. **第二步**：去 eMMC（mmc 0:1）找名为 imx6.dtb 的文件，把它搬到内存地址 ${fdt_addr} (即 0x83000000)。
3. **第三步**：执行 bootz，告诉 CPU：“去 0x80800000 找内核，去 0x83000000 找设备树，开始跑吧！”



<img src="C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260402210326195.png" alt="image-20260402210326195"  />

为什么 uboot 的起始地址是 0x00000400（1024 字节处）而不是 0？

补习点：因为 EMMC 的前 1024 字节通常留给分区的 MBR 或特定的启动头。

（**引导芯片的BootROM正确加载并启动U-Boot** ，**分区的 MBR** 就是磁盘的第一个扇区中那个**包含分区表和引导代码的数据结构**

- 如果设备是作为 **PC 硬盘**使用，偏移 0x0 就是 MBR。
- 如果设备是作为 **嵌入式启动介质**（比如存放 U-Boot），偏移 0x0 就是芯片厂商的启动头。





ROM 代码把SPL搬入SRAM

SPL初始化DDR，运行内存

把uboot拉入DDR内存

uboot是个程序，查看bootcmd 把zImage DTB 拉入DDR

通过设置bootargs 记到DTB /chosen结点  emmc分区情况 信息打印到哪个串口 rw权限

将DTB的地址记到R2寄存器 然后进入zImage的首地址开始运行 地址头部会有解压的代码

调用head.S 进入kernel 开始解析dtb 根据设备树信息初始化各种驱动 时钟 中断等

然后调用函数运行第一个用户程序 PID=1



当敲入upfs命令时 会触发do_upfs函数

```
U_BOOT_CMD(
    upfs,         // 命令名称
    1,            // 最大参数个数（包括命令本身，即执行时不需要额外参数）
    0,            // 是否可重复（0 表示不可重复）
    do_upfs,      // 命令对应的执行函数
    "updatex up <uboot/kernel/all>",   // 简短帮助信息
    "updatex up <uboot/kernel\n"       // 详细帮助信息
);
```



/include/configs/mx6ullevk.h 告诉uboot 如何为这个板子进行裁剪



如何通过tftp实现烧录zImage和DTB？

设置IP之后 用tftp直接把zImage下载到内存的指定地址 

```
setenv ipaddr 192.168.1.100      # 开发板 IP
setenv serverip 192.168.1.10     # 主机（TFTP 服务器）IP
setenv gatewayip 192.168.1.1     # 网关（可选）
setenv netmask 255.255.255.0
```

```
tftp 0x80800000 zImage
# 2. 加载设备树到另一内存地址（例如 0x83000000）
tftp 0x83000000 imx6ull-14x14-evk.dtb
# 3. 启动内核
bootz 0x80800000 - 0x83000000
```

任务：编译出来镜像 然后加打印 可以的话改动一下



从RK最新的SDK独立完成移植



入口：

![image-20260403230050599](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260403230050599.png)

之后 会跳转到obc-1.0.0  代码就都在这了

![image-20260403230224743](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260403230224743.png)

![image-20260404113224124](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260404113224124.png)

出口 加载内核

![image-20260403230441140](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260403230441140.png)

分区信息解耦到设备树里



代码调用流程：

1.入口：D:\OBC_Code-master\2-Imx6ull-Board\sdk-source\uboot\uboot-nxp-2024.04\common\board_r.c里的init_sequence_r[]里的do_obcboot

该init_sequence_r作用是：在 U-Boot 把自己从 Flash/SRAM 搬运到 **DDR（内存）** 运行之后，它需要按照顺序执行这个列表里的函数，把硬件驱动、环境变量、网络系统等一个个“唤醒”。

2.通过do_obcboot 调用里面的obc_board_init函数，

实现板级分离，针对每个板子有单独的初始化，板级抽象层（BAL），目前是对6u的四个初始化

#if defined(CONFIG_BOARD_CONFIG_IMX6ULL)
    g_obc_ability_manager.pstBoard = &g_imx6ull_board;
#endif

    /* 1# 板级硬件初始化 */
    知道当前是从 SD 卡还是 eMMC 启动
    获得块设备的操作句柄
    明确关键数据在内存中的存放位置
    (void)obc_board_hw_init();
    
    /* 2# 环境变量设置*/
    网络配置（TFTP 升级用）
    内存地址映射
    启动介质自适应的自动启动命令
    它使得板子在 SD 卡启动时自动从 FAT 分区加载内核，在 eMMC 启动时调用 OBC 框架的 bootk 命令从裸分区引导，实现了灵活的启动策略。
    (void)obc_board_env_init();
    
    /* 3# 设备树解析 */
    /* 1# 加载设备树,检查设备树是否有效 */
    /* 2# 解析设备树填充ability的dev part分区信息 */
    涉及调用obc_fdt_load_to_mem 写死了从fdt0获取设备树并加载到内存 --->  obc_blk_read_part_by_name --->obc_blk_find_part_by_name
    obc_blk_parse_fdt
    
    (void)obc_board_fdt_init();
    
    /* 3# 启动参数配置解析 */
    /* 设置启动的bootargs参数，包含console、mmcblk */
    涉及调用 obc_bootargs_set
    (void)obc_board_args_init();





问题：

```
typedef struct BOARD_CONFIG_TABLE
{
    /*
        board_hw_init:板级硬件初始化
        board_env_init：板级设备树初始化
        board_fdt_init：板级设备树初始化
        board_args_init：板级bootargs参数初始化
    */
    int (*board_hw_init)(BOARD_ABILITY_TABLE_T *);
    int (*board_env_init)(BOARD_ABILITY_TABLE_T *);
    int (*board_fdt_init)(BOARD_ABILITY_TABLE_T *);
    int (*board_args_init)(BOARD_ABILITY_TABLE_T *);
}BOARD_CONFIG_TABLE_T; 
```

该函数指针 在哪里被赋值？？

D:\OBC Code-master\2-Imx6ull-Boardlobc-1.0.0\bootloader\obcbase\board\imx\board_ config_imx6ull.c的

```
BOARD_CONFIG_TABLE_T g_imx6ull_board = {
    .board_hw_init          = imx6ull_board_hw_init,
    .board_env_init         = imx6ull_board_env_init,
    .board_fdt_init         = imx6ull_board_fdt_init,
    .board_args_init        = imx6ull_board_args_init,
};  //赋值
```

四个函数内有obc_fdt_load_to_mem，obc_bootargs_set函数 定义在D:\OBC Code-master\2-Imx6ull-Board\obc-1.0.0\bootloader\obcbase\commonlobc blk.c

其中  obc_bootargs_set 实现了    // 将当前bootargs和blkdevparts拼接 

len = snprintf(bootargs, sizeof(bootargs), "setenv bootargs %s blkdevparts=%s", pConsole, blkdevparts);



如何从设备树里获取partitions节点信息、分区情况？？

obc_board_fdt_init -->imx6ull_board_fdt_init --->obc_blk_parse_fdt--> obc_blk_parse_partitions 解析节点 填充信息

通过

```
typedef struct BOARD_ABILITY_BLK_PARTS
{
    char lable[16];
    uint32_t addr;
    uint32_t size;
    uint8_t flag;
}BOARD_ABILITY_BLK_PARTS_T;
```

该全局结构体 在obc_bootargs_blkparts_set中传递给blkdevparts 最后通过setenv bootargs 写入环境变量

![image-20260406164929222](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260406164929222.png)



![image-20260406203221415](C:\Users\Admin\AppData\Roaming\Typora\typora-user-images\image-20260406203221415.png)





***\*2025.12-2026.03\****                    ***\*RK3566嵌入式系统框架（OBC）设计与实现\****

**项目描述：**针对工业级RK3566平台，设计并实现了一套高可靠、低时延的嵌入式系统引导框架，核心解决启动时序优化、固件原子升级、硬件抽象解耦等遇到的实际问题。

**技术栈：**技术栈：U-Boot、设备树（DTB）、eMMC RPMB分区、Buildroot、TFTP协议栈

**工作内容****：**

● 通过存储介质分层策略（SD->eMMC），结合U-Boot子系统级裁剪（移除USB/PCIE/DM驱动栈），将冷启动时序从5.6s压缩2.3s（优化59%）

● 设计设备树驱动的动态分区映射层，在引导阶段解析DTB分区节点，实现分区布局的运行时动态重构，实现修改分区无需重编Bootloader工程化目标

● 高可靠A/B冗余升级架构，构建覆盖FDT、Kernel、RootFS的全镜像双分区（A/B）升级体系，支持差分/全量两种更新模式

● 设计TFTP+本地双通道升级链路，适配现场OTA场景

● 实现掉电安全的状态机管理，升级过程中引入CRC校验、签名验证，确保异常中断后可自愈恢复

