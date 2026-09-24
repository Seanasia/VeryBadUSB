# VeryBadUSB调试指南
### 曾用名：BadUSB2.0.0

当前，你所拿到的VeryBadUSB是已经烧录好的，我们将直接讲述如何进行调试和烧录，并附有故障或完全处在初始化状态的从零启动指南。

---

## 1. 调试前检查（默认已检查）

### 电源
- CH552G `15(VCC/VDD)` → `+5V`
- CH552G `14(GND/VSS)` → `GND`
- CH552G `16(V33)` → `100nF` → `GND`
- `V33` 上电后应约为 `3.3V`

### USB
- `D+` → CH552G `12(P3.6/UDP)`
- `D-` → CH552G `13(P3.7/UDM)`
- D+/D- 不要互换、短路。
- USBLC6-2SC6：
  - `2` → GND
  - `5` → +5V
  - `1/6` → D-
  - `3/4` → D+

### BOOT
`V33` ── `SW_BOOT` ── `1kΩ` ── `D+`
BOOT 只用于进入下载模式。

---

注：以下均由Chatgpt编写……

## 2. 是否需要 USB-TTL

**不需要。**

CH552G 可以直接通过自身 USB Bootloader 烧录。

USB-TTL 只是可选调试工具，用来看日志：

```text
CH552 TXD(7) ──→ USB-TTL RX
CH552 GND    ──→ USB-TTL GND
```

不看日志时完全可以不接。

---

## 3. 推荐软件环境

使用：

- Arduino IDE
- ch55xduino
- WCH USB 下载驱动

ch55xduino 开发板管理器地址：

```text
https://raw.githubusercontent.com/DeqingSun/ch55xduino/ch55xduino/package_ch55xduino_mcs51_index.json
```

Arduino IDE 中选择：

```text
Board: CH552 Board
Clock: 24 MHz internal / 5V
Upload Method: USB
Bootloader Pin: P3.6 (D+) pull-up
```

---

## 4. 烧录方法

每次修改代码后都可以重新烧录。

```text
1. 先在 Arduino IDE 编译
2. 拔掉 CH552 USB
3. 按住 BOOT
4. 插入 USB
5. 松开 BOOT
6. 点击/等待上传
7. 上传完成
8. 拔插 USB，让程序正常启动
```

如果上传超时，重新执行一次即可。

> Bootloader 等待时间有限，不建议先进入 BOOT 后再慢慢编译。

---

## 5. 第一阶段：确认芯片正常运行

先不要做 HID 自动输入。

如果有 USB-TTL，可先烧录简单串口程序：

```c
#include <Arduino.h>

void setup(void) {
    Serial0_begin(115200);
    Serial0_println("BOOT");
}

void loop(void) {
    Serial0_println("ALIVE");
    delay(1000);
}
```

串口设置：

```text
115200
8N1
无流控
```

正常结果：

```text
BOOT
ALIVE
ALIVE
ALIVE
...
```

没有 USB-TTL 时可以跳过这一阶段。

---

## 6. 第二阶段：测试 HID 键盘

推荐直接从 ch55xduino 自带示例开始：

```text
Generic_Examples
└── 05.USB
    └── HidKeyboard
```

**必须保留示例中的整个 `src` 目录，不要只复制 `.ino` 文件。**

USB Settings 选择：

```text
USER CODE w/ 148B USB ram
```

测试逻辑建议：

```text
USB 初始化
↓
等待主机完成 USB 枚举
↓
延时 5~8 秒
↓
输入固定测试字符串
↓
停止
```

第一次只测试：

```text
CH552 HID TEST 1234567890
```

先在记事本中验证，不要直接测试命令执行。

---

## 7. 实际测试步骤

```text
1. 打开 Windows 记事本
2. 切换英文输入法
3. 把光标放到记事本
4. 插入 CH552 板
5. 等待几秒
6. 查看是否自动出现测试文字
```

正常结果：

```text
CH552 HID TEST 1234567890
```

连续拔插测试至少 5 次，确认：
- 每次都能识别
- 不丢字
- 不重复字符
- 不乱码
- 每次只执行一次

---

## 8. 长文本处理

长文本建议放 Flash，不要占大量 RAM：

```c
static const __code char payload[] =
    "FIRST PART "
    "SECOND PART "
    "THIRD PART";
```

逐字发送时先保守设置：

```text
每字符间隔：10~20 ms
```

稳定后再逐步降低。

---

## 9. 常见故障

### 插入完全没反应
检查：
- 5V
- V33 是否约 3.3V
- CH552 焊接方向
- D+/D-
- USB 插头
- USBLC6

### 可以进入 BOOT，但 HID 不识别
检查：
- HID 示例是否完整
- `USB Settings`
- D+/D- 是否接反
- 焊点
- USBLC6 方向/封装

### 可以识别键盘，但没有输入
检查：
- 当前窗口是否有输入焦点
- 是否真的执行到发送代码
- 延时时间是否足够

### 输入乱码
优先检查：
- Windows 当前键盘布局
- 输入法
- Caps Lock
- 特殊符号映射

### 丢字
增加字符间隔：

```text
5ms → 10ms → 20ms
```

直到稳定。

---

## 10. 推荐开发顺序

```text
① 测电源
↓
② BOOT 能识别
↓
③ 能重复烧录
↓
④ USB HID 能枚举
↓
⑤ 记事本输入短字符串
↓
⑥ 长字符串
↓
⑦ 调整输入速度
↓
⑧ 最后再加入实际按键流程
```

---

## 11. 最终验收

- [ ] +5V 正常
- [ ] V33 ≈ 3.3V
- [ ] BOOT 可以稳定进入下载模式
- [ ] 可以反复重新烧录
- [ ] Windows 能稳定识别为 HID Keyboard
- [ ] 连续 5 次拔插均正常
- [ ] 长文本无丢字、乱码
- [ ] 每次启动只执行一次
- [ ] 不依赖 USB-TTL 即可正常工作

USB-TTL 只作为故障定位工具，最终成品不需要。

## 附录1：

本硬件的完整原理图：
