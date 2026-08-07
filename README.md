# ESP32-C3 外部固件烧录指南 -小吴同学电气设计

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Platform](https://img.shields.io/badge/platform-ESP32--C3-blue)](https://www.espressif.com/zh-hans/products/socs/esp32-c3)

本仓库提供了一份全面的 **ESP32-C3** 外部烧录教程，旨在帮助开发者在不依赖特定开发板集成功能的情况下，通过 **UART** 或 **USB** 方式将固件（`.bin` 文件）烧录到 ESP32-C3 芯片或模组中。

无论你使用的是 ESP32-C3 开发板、还是基于 ESP32-C3 的自制硬件，这份指南都能为你提供清晰、可操作的步骤。

---

## 📁 项目结构

```
.
├── firmware/                         # 存放待烧录的固件文件
│   ├── bootloader.bin                # 引导加载程序 (通常烧录地址: 0x0)
│   ├── partitions.bin                # 分区表 (通常烧录地址: 0x8000)
│   └── firmware.bin                  # 主应用程序 (通常烧录地址: 0x10000)
├── docs/
│   └── hardware_connection.md        # 详细的硬件连接图解
├── scripts/                          # 一键烧录脚本
│   ├── flash.bat                     # Windows 批处理脚本
│   └── flash.sh                      # Linux / macOS Shell 脚本
└── README.md                         # 本文档
```

---

## 🚀 快速开始

### 1. 硬件准备

ESP32-C3 支持两种外部烧录方式：**UART** 和 **USB**。

#### 选项 A：UART 烧录（通用性最强）

这种方式适用于所有 ESP32-C3 模组和开发板，需要使用外部 USB 转串口工具。

| ESP32-C3 管脚 | 连接至 USB 转串口工具 |
| :--- | :--- |
| **3V3** | **VDD** (3.3V) |
| **GND** | **GND** |
| **EN** (CHIP_PU) | **VDD** (3.3V)  |
| **GPIO2** | **VDD** (3.3V，拉高)  |
| **GPIO8** | **VDD** (3.3V，拉高)  |
| **GPIO9** | **GND** (拉低，进入下载模式)  |
| **GPIO20** (U0RXD) | **TXD** (串口工具的发送端)  |
| **GPIO21** (U0TXD) | **RXD** (串口工具的接收端)  |

> **⚠️ 重要提示**：
> - 电源需提供 **3.3V** 电压，且输出电流能力建议达到 **500mA 及以上**。
> - **GPIO9** 是进入下载模式的关键：拉低后上电，芯片即进入下载模式。烧录完成后，需将 GPIO9 拉高或悬空并复位芯片，才能正常启动固件。
> - **EN (CHIP_PU)** 管脚不可悬空，必须为高电平芯片才能工作。

#### 选项 B：USB 烧录（更便捷）

ESP32-C3 支持通过其内置的 USB 外设直接烧录，无需外部 USB 转 UART 桥。

| ESP32-C3 管脚 | 连接至 USB 线 |
| :--- | :--- |
| **3V3** | **VDD** (3.3V) |
| **GND** | **GND** |
| **EN** (CHIP_PU) | **VDD** (3.3V) |
| **GPIO2** | **VDD** (3.3V，拉高) |
| **GPIO8** | **VDD** (3.3V，拉高) |
| **GPIO9** | **GND** (拉低) |
| **GPIO18** | **USB_D-**  |
| **GPIO19** | **USB_D+**  |

> **💡 提示**：许多 ESP32-C3 开发板（如 ESP32-C3-DevKitM-1）已集成 USB 接口，只需将开发板通过 USB 线连接电脑即可，无需额外接线。

---

### 2. 进入下载模式

无论使用哪种烧录方式，都需要让芯片进入下载模式。

- **对于模组或自制硬件**：按照上述接线要求，将 **GPIO9 拉低**后给芯片上电或复位，芯片即进入下载模式。成功进入后，串口会输出如下日志：
  ```
  ESP-ROM:esp32c3-api1-20210207 Build:Feb 7 2021
  rst:0x1 (POWERON),boot:0x4 (DOWNLOAD(USB/UART0/1))
  waiting for download
  ```
  

- **对于开发板**：通常只需按住 **BOOT** 按钮（连接 GPIO9），同时按一下 **RESET** 按钮，然后松开 BOOT 按钮即可。

---

### 3. 软件准备

#### 安装 esptool.py（推荐）

`esptool.py` 是乐鑫官方推出的 Python 烧录工具，功能强大且跨平台。

```bash
pip install esptool
```

验证安装：
```bash
esptool.py version
```

#### 下载乐鑫 Flash 下载工具（备选）

如果你偏好图形化界面，可以从乐鑫官网下载 [Flash 下载工具](https://www.espressif.com/zh-hans/support/download/other-tools)。

---

## 🔥 烧录方法一：使用 esptool.py 命令行（推荐）

这是最灵活、最适合自动化场景的方式。

### 一键脚本

在仓库根目录下，根据你的系统运行：

- **Windows**：双击 `scripts/flash.bat`
- **Linux / macOS**：
  ```bash
  chmod +x scripts/flash.sh
  ./scripts/flash.sh
  ```

### 手动命令

打开终端，进入存放 `.bin` 文件的目录，执行以下命令：

```bash
esptool.py --chip esp32c3 -p [PORT] -b 460800 \
  --before=default_reset --after=hard_reset \
  write_flash --flash_mode dio --flash_freq 80m --flash_size 4MB \
  0x0 bootloader.bin \
  0x8000 partitions.bin \
  0x10000 firmware.bin
```

**将 `[PORT]` 替换为实际的串口号**：

| 操作系统 | 常见端口名 |
| :--- | :--- |
| Windows | `COM3`、`COM4` 等 |
| Linux | `/dev/ttyUSB0`、`/dev/ttyACM0` |
| macOS | `/dev/cu.usbserial-*` 或 `/dev/cu.wchusbserial-*` |

#### 参数说明

| 参数 | 说明 |
| :--- | :--- |
| `--chip esp32c3` | 指定芯片类型为 ESP32-C3 |
| `-p [PORT]` | 指定串口端口 |
| `-b 460800` | 烧录波特率（可调整为 115200 以提高稳定性） |
| `write_flash` | 烧录命令 |
| `--flash_mode dio` | SPI Flash 工作模式 |
| `--flash_freq 80m` | SPI Flash 频率 |
| `--flash_size 4MB` | SPI Flash 大小，请根据实际芯片调整 |
| `0x0 bootloader.bin` | 将 `bootloader.bin` 烧录到地址 `0x0` |
| `0x8000 partitions.bin` | 将 `partitions.bin` 烧录到地址 `0x8000` |
| `0x10000 firmware.bin` | 将 `firmware.bin` 烧录到地址 `0x10000` |

---

## 🖥️ 烧录方法二：使用乐鑫 Flash 下载工具（图形化界面）

适合不熟悉命令行的用户。

1. **下载并运行工具**：从乐鑫官网下载 [Flash 下载工具](https://www.espressif.com/zh-hans/support/download/other-tools)，解压后运行 `flash_download_tool_xxx.exe`。

2. **选择芯片类型**：在工具界面中，选择 **ESP32-C3**。

3. **配置烧录文件及地址**：按以下表格添加 `.bin` 文件并填写对应的烧录地址：

   | 文件名称 | 烧录地址 (Offset) |
   | :--- | :--- |
   | `bootloader.bin` | **0x0** |
   | `partitions.bin` | **0x8000** |
   | `firmware.bin` | **0x10000** |

   > **务必勾选每个文件前的复选框**。

4. **选择烧录方式并开始**：
   - 选择 **UART** 或 **USB** 模式。
   - 选择正确的 **COM** 端口。
   - 点击 **START** 按钮开始烧录。

5. **烧录完成**：烧录成功后，将 GPIO9 拉高或悬空，按一下 RESET 键复位芯片，新固件即可运行。

---

## ❓ 常见问题（FAQ）

### 1. 芯片无法进入下载模式

- **检查 GPIO9 电平**：确保 GPIO9 在启动时被拉低。
- **检查 EN 管脚**：确保 EN (CHIP_PU) 为高电平。
- **检查电源**：确保供电电压为 3.3V，且电流充足。
- **对于开发板**：严格按照“按住 BOOT → 按 RESET → 松开 BOOT”的顺序操作。

### 2. 烧录时报错 `Failed to connect`

- **串口被占用**：关闭其他占用串口的软件（如串口监视器）。
- **波特率过高**：尝试降低烧录波特率，如 `-b 115200`。
- **驱动问题**：检查并安装正确的 USB 转串口驱动程序。

### 3. 烧录成功后程序不运行

- **GPIO9 电平不对**：烧录完成后，确保 GPIO9 已拉高或悬空。
- **分区表不匹配**：确认 `partitions.bin` 与 `firmware.bin` 是配套的。
- **未复位**：按一下 RESET (EN) 键复位芯片。

### 4. 如何查看串口设备名称？

- **Windows**：打开设备管理器，查看“端口 (COM 和 LPT)”。
- **Linux**：运行 `ls /dev/tty*`。
- **macOS**：运行 `ls /dev/cu.*`。

---

## 📜 许可证

本项目基于 **MIT 许可证** 开源。

---

## 🔗 相关资源

- [乐鑫 ESP32-C3 技术规格书](https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_cn.pdf)
- [esptool.py 官方文档](https://docs.espressif.com/projects/esptool/en/latest/esp32c3/)
- [ESP-IDF 编程指南 (ESP32-C3)](https://docs.espressif.com/projects/esp-idf/zh_CN/latest/esp32c3/)

---

**Happy Flashing!** 🚀
```

---
