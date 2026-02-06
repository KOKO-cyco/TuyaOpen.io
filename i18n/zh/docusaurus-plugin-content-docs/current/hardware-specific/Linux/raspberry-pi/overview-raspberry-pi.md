---
title: "Raspberry Pi 概述"
---

本文档介绍如何在树莓派上运行 tuyaopen 的 [your_chat_bot](https://tuyaopen.ai/zh/docs/applications/tuya.ai/demo-your-chat-bot) 项目。

Raspberry Pi 支持交叉编译和本地编译两种方式，编译的时候会自行判断当前平台并选择合适的编译方式。

与 T5 平台相比，树莓派需要注意以下两点：

1. 需要外接 USB 声卡
2. 需要手动配置模型路径

## 外置声卡

树莓派默认没有内置麦克风和扬声器，因此需要使用外置 USB 声卡。推荐使用以下型号：

- **USB音频模块 YD1076**，[淘宝链接](https://e.tb.cn/h.77Vo2K5tJIaL86g?tk=lnBAUbwVNB9)

也可以自行选择其他兼容的 USB 声卡。需要注意的是，麦克风应支持输出原始音频数据，不要自带降噪、回声消除等处理功能。

## 模型路径配置

### 获取唤醒模型

KWS 唤醒模型有两种获取方式：

**方式一：自动下载**

选择树莓派平台并编译成功后，模型会自动下载到 `platform/LINUX/tuyaos_adapter/src/tkl_audio/models` 目录。

**方式二：手动下载**

使用下面的命令下载 `mdtc_chunk_300ms.mnn` 和 `tokens.txt` 文件到 `~/tuyaopen_models` 目录下：

```bash
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/mdtc_chunk_300ms.mnn
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/tokens.txt
```

### 配置唤醒模型

获取模型后，按以下步骤配置路径：

1. 激活 tos.py 环境，进入 `apps/tuya.ai/your_chat_bot` 目录
2. 执行 `tos.py config choice` 命令，选择 `RaspberryPi.config` 配置
3. 执行 `tos.py config menu` 命令，依次进入：`(Top) → Choice a board → LINUX → TKL Board Configuration`
4. 根据 `mdtc_chunk_300ms.mnn` 和 `tokens.txt` 的实际存放路径，修改以下配置项：
   - `KWS model file path`
   - `KWS model token file path`

> **注意**：请确保模型文件已放置在树莓派的文件系统中，且路径配置正确，否则唤醒功能无法正常工作。

### 配置示例

例如，将模型文件存放在 `~/tuyaopen_models` 目录下时，路径配置如下：

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/4e3897a7-6d32-40e2-b2bd-6d2a6497076e.png)

> **重要提示**：路径必须填写正确，否则语音唤醒功能无法正常工作。

## 补充说明

### 使用交叉编译时的文件传输

如果您使用的是交叉编译方式，可以借助于 scp 命令将编译得到的可执行文件从编译主机传输到树莓派。例如：

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin username@192.168.1.xxx:/home/xx/Desktop/
```

**命令说明：**
- `username` - 开发板的用户名
- `192.168.1.xxx` - 开发板的 IP 地址
- `/home/xx/Desktop/` - 开发板上的目标目录

## 常见问题

**Q: 语音唤醒不工作怎么办？**  
A: 请检查模型文件路径是否正确，文件是否完整。可以使用 `ls -lh` 命令查看文件是否存在及大小是否正常。

