---
title: "DshanPi-A1 Overview"
---

This guide will walk you through running TuyaOpen's [your_chat_bot](https://tuyaopen.ai/docs/applications/tuya.ai/demo-your-chat-bot) chatbot project on the [DshanPi-A1](https://rockchip.100ask.net/en/docs/DshanPi-A1/intro/) development board.

## Compilation Methods

DshanPi-A1 supports two compilation approaches:
- **Cross-compilation**: Compile on your PC, then transfer the executable to the development board
- **Native compilation**: Compile directly on the development board

The system will automatically detect your platform and select the appropriate compilation method.

> **Note**: Cross-compilation is not supported on macOS. Please use Linux or compile directly on the development board.

## Quick Start Guide

Compared to the T5 platform, running on DshanPi-A1 requires two additional configurations:

1. Configure the onboard audio (to enable recording and audio playback)
2. Configure the voice wake-up model path

## Configure the Onboard Audio

The DshanPi-A1 development board comes with a built-in microphone and speaker, but they require configuration before use.

### 1.1 Install Audio Support Library

First, install the ALSA audio library (used for audio input and output processing):

```bash
sudo apt-get install libasound2-dev
```

### 1.2 Configure Speaker Output

DshanPi-A1 has two audio output devices: a headphone jack and an onboard speaker. For convenience, we'll configure the onboard speaker as the default output device.

Edit the audio configuration file:

```bash
sudo vi /etc/asound.conf
```

Add the following content to the file:

```plaintext
pcm.speaker_r {
  type route
  slave.pcm "hw:0,0"
  slave.channels 2
  ttable.0.1 1
  ttable.1.1 0
  ttable.0.0 0
  ttable.1.0 0
}
```

Save and exit the editor (press `ESC`, type `:wq`, and press `Enter`).

### 2.1 Obtain Model Files

There are two ways to obtain the wake-up model:

**Method 1: Automatic Download (Recommended)**
- When you select the DshanPi-A1 platform for compilation, the model will be automatically downloaded to the project directory:
  ```
  platform/LINUX/tuyaos_adapter/src/tkl_audio/models
  ```

**Method 2: Manual Download**
- Download the following two files from the [TuyaOpen-ubuntu repository](https://github.com/tuya/TuyaOpen-ubuntu/tree/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models):
  - `mdtc_chunk_300ms.mnn` - Wake-up model file
  - `tokens.txt` - Model token file

### 2.2 Upload Model to Development Board

Upload the model files to a directory on the development board, for example `/home/pi/Desktop/models/`.

### 2.3 Configure Model Path

Configure the model file paths in your project:

1. Navigate to the project directory:
   ```bash
   cd apps/tuya.ai/your_chat_bot
   ```

2. Select DshanPi-A1 configuration:
   ```bash
   tos.py config choice
   ```
   Select `DshanPi_A1.config` from the menu

3. Open the configuration menu:
   ```bash
   tos.py config menu
   ```

4. Navigate to the configuration items following this path:
   ```
   (Top) → Choice a board → LINUX → TKL Board Configuration
   ```

5. Update the following two configuration items with the actual paths to your model files:
   - `KWS model file path` → Enter the full path to `mdtc_chunk_300ms.mnn`
   - `KWS model token file path` → Enter the full path to `tokens.txt`

### 2.4 Configuration Example

If you placed the model files in the `/home/pi/Desktop/models/` directory, the configuration should look like this:

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/eba887ff-d37f-4161-8f08-b641863ab9a6.png)

> **Important**: The paths must be correct, otherwise the voice wake-up feature will not work. We recommend using absolute paths (full paths starting with `/`).

## Additional Information

### File Transfer for Cross-compilation

If you're using cross-compilation on your PC, you'll need to transfer the compiled executable to the development board. You can use the `scp` command:

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin username@192.168.1.xxx:/home/pi/Desktop/
```

**Command parameters:**
- `username` - Username on the development board
- `192.168.1.xxx` - IP address of the development board
- `/home/xxx/Desktop/` - Target directory on the development board

## Frequently Asked Questions

**Q: No sound after configuring the audio device. What should I do?**  
A: Use the `aplay -l` command to view available audio devices and confirm the device number is correct.

**Q: Voice wake-up is not working. What should I do?**  
A: Check that the model file paths are correct and the files are complete. Use the `ls -lh` command to verify the files exist and their sizes are normal.

**Q: How can I check the IP address of the development board?**  
A: Run `ip addr` or `ifconfig` on the development board to view its IP address.
