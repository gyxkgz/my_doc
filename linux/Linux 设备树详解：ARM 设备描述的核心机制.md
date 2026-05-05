# Linux 设备树：ARM 设备描述的核心机制

## 引言

设备树（Device Tree）是 Linux 系统中描述硬件资源配置的重要数据结构，它将硬件平台信息从内核代码中分离出来，实现了内核与硬件的解耦。在运行时，Bootloader将设备树二进制文件（`.dtb`）独立加载到内存，再将地址传递给内核，由内核进行解析和实例化。本文将解析设备树的核心概念、结构以及在 ARM 嵌入式系统中的应用。

## 为什么需要设备树

在传统的嵌入式开发中，硬件信息通常硬编码在内核源码中：

```c
// 传统方式：硬件信息写在代码里
static struct platform_device led_device = {
    .name = "my-led",
    .id = -1,
    .resource = {
        .start = 0x40021000,
        .end = 0x4002100f,
        .flags = IORESOURCE_MEM,
    },
};
```

这种方式的问题：

- 换一个硬件平台就需要修改内核代码
- 内核代码与硬件绑定，无法复用
- 维护困难，难以适应硬件变化

设备树的出现解决了这个问题，它使用独立的数据文件来描述硬件，使得同一个内核镜像可以支持多个不同的硬件平台。

## 设备树基本结构

设备树是一种树形数据结构，由节点（nodes）和属性（properties）组成：

```
/ {
    model = "My ARM Board";
    compatible = "vendor,myboard";

    chosen {
        bootargs = "console=ttyS0,115200";
    };

    memory@0 {
        device_type = "memory";
        reg = <0x80000000 0x10000000>;
    };

    leds {
        compatible = "gpio-leds";
        led0 {
            gpios = <&gpioa 5 0>;
            default-state = "on";
        };
    };

    soc {
        #address-cells = <1>;
        #size-cells = <1>;

        gpioa: gpio@40021000 {
            compatible = "st,stm32-gpio";
            reg = <0x40021000 0x400>;
            clocks = <&rcc 0>;
        };
    };
};
```

## 关键概念解析

### 1. 节点（Node）

每个节点代表一个设备或总线，用名字标识：

- `leds` - LED 设备节点
- `gpio@40021000` - GPIO 控制器（@后面是地址）

### 2. 属性（Property）

属性描述节点的特性：

- `compatible` - 驱动匹配字符串
- `reg` - 寄存器地址和大小
- `gpios` - GPIO 引脚引用
- `#address-cells` - 地址单元格数
- `#size-cells` - 大小单元格数

### 3. 引用（Phandle）

使用 phandle 引用其他节点：

```dts
// 引用 gpio 节点
gpios = <&gpioa 5 0>;
// &gpioa 指向 gpio@40021000 节点
```

一个SoC往往对应多个电路板。因此，内核将SoC的通用硬件描述提炼为 `.dtsi` 文件（类似于C语言的头文件），而将具体电路板的差异化描述放在 `.dts` 文件中。

## 设备树编译

设备树源文件（.dts）需要编译成二进制格式（.dtb）才能被内核使用：

```bash
# 编译设备树
dtc -I dts -O dtb -o board.dtb board.dts

# 反编译（调试用）
dtc -I dtb -O dts -o board.dts board.dtb
```

如果设备树中用#include包含了其他dtsi文件或者内核头文件，直接用dtc命令是无法编译的，可以先用cpp命令预处理，然后再用dtc编译设备树文件。

```
cpp -nostdinc -I /media/g/kf/workspace/linux-6.6.41/include/ -undef -D__DTS__ -x assembler-with-cpp stm32mp257f-ev1.dts|dtc -I dts -O dtb -o test.dtb -
```

## 驱动匹配机制

内核通过 `compatible` 属性匹配设备与驱动：

```c
// 内核驱动中的匹配表
static const struct of_device_id led_of_match[] = {
    { .compatible = "gpio-leds", },
    { .compatible = "ns2-led", },
    { /* sentinel */ }
};

MODULE_DEVICE_TABLE(of, led_of_match);
```

当设备树的 `compatible` 与驱动的匹配表一致时，内核会自动实例化设备。

## 设备树在 ARM 的应用

ARM 架构是设备树的主要应用场景。以 STM32MP157 为例：

```dts
&uart4 {
    pinctrl-0 = <&uart4_pins_a>;
    pinctrl-names = "default";
    status = "okay";
};

&i2c1 {
    pinctrl-0 = <&i2c1_pins_a>;
    pinctrl-names = "default";
    i2c-scl-rising-time-ns = <18500>;
    i2c-scl-falling-time-ns = <300>;
    status = "okay";

    eeprom@50 {
        compatible = "atmel,24c256";
        reg = <0x50>;
    };
};
```

## 总结

设备树是现代嵌入式 Linux 开发的核心技术：

1. **解耦** - 硬件描述与内核代码分离
2. **复用** - 同一内核镜像支持多平台
3. **灵活** - 可在启动时或运行时动态修改
4. **标准化** - 成为 ARM/嵌入式的事实标准




