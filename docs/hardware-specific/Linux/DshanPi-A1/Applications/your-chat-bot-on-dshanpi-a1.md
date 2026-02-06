---
title: "Running your_chat_bot on DshanPi-A1"
---

This document guides you through running the TuyaOpen [your_chat_bot](https://tuyaopen.ai/docs/applications/tuya.ai/demo-your-chat-bot) chatbot project on the [DshanPi-A1](https://rockchip.100ask.net/en/docs/DshanPi-A1/intro/) development board.

## Build Methods

DshanPi-A1 supports two build methods:

- **Cross-compilation**: Build on a PC, then transfer and run on the board
- **Native build**: Build directly on the board

The system automatically detects the current platform and selects the appropriate build method.

> **Note**: Cross-compilation is not supported on macOS. Use Linux or build directly on the board.

## Quick Start

Running your_chat_bot on DshanPi-A1 requires two additional configuration steps:

1. Configure the onboard sound card (so the board can record and play audio)
2. Configure the voice wake-up model path

## Step 1: Configure the Onboard Sound Card

The DshanPi-A1 board has a built-in microphone and speaker, but they must be configured before use.

### 1.1 Install Audio Libraries

Install the ALSA library (for audio input and output):

```bash
sudo apt-get install libasound2-dev
```

### 1.2 Configure Speaker Output

The DshanPi-A1 has two audio output devices: a headphone jack and an onboard speaker. For convenience, we configure the onboard speaker as the default output device.

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

Save and exit the editor (press `ESC`, type `:wq`, then press `Enter`).

## Step 2: Configure the Voice Wake-up Model

Voice wake-up uses a KWS (keyword spotting) model so the device can recognize "你好涂鸦" (Hello Tuya).

### 2.1 Obtain Model Files

You can obtain the wake-up model in two ways:

**Option 1: Automatic download**

- When you select the DshanPi-A1 platform for build, the model is downloaded automatically into the project directory:
  ```
  platform/LINUX/tuyaos_adapter/src/tkl_audio/models
  ```

**Option 2: Manual download**

Download `mdtc_chunk_300ms.mnn` and `tokens.txt` to your `~/tuyaopen_models` directory with:

```bash
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/mdtc_chunk_300ms.mnn
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/tokens.txt
```

The default path in the code is `~/tuyaopen_models`. If you need to change it, follow the steps below.

### 2.2 Configure Model Paths

Configure the paths to the model files in the project:

1. Go to the project directory:
   ```bash
   cd apps/tuya.ai/your_chat_bot
   ```

2. Select DshanPi-A1 configuration:
   ```bash
   tos.py config choice
   ```
   Choose `DshanPi_A1.config` from the menu.

3. Open the configuration menu:
   ```bash
   tos.py config menu
   ```

4. Navigate to the following path in the menu:
   ```
   (Top) → Choice a board → LINUX → TKL Board Configuration
   ```

5. Set these two options to the actual paths of the model files:
   - `KWS model file path` → Enter the full path to `mdtc_chunk_300ms.mnn`
   - `KWS model token file path` → Enter the full path to `tokens.txt`

### 2.4 Configuration Example

If the model files are in `~/tuyaopen_models`, the configuration should look like the following:

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/4e3897a7-6d32-40e2-b2bd-6d2a6497076e.png)

> **Important**: The paths must be correct, or voice wake-up will not work.

## Additional Notes

### Transferring Files When Using Cross-compilation

If you cross-compile on a PC, you need to copy the built executable to the board. You can use `scp`:

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin username@192.168.1.xxx:~/
```

**Command parameters:**

- `username` — Username on the board
- `192.168.1.xxx` — IP address of the board
- `~/` — Target directory on the board

## FAQ

**Q: Voice wake-up is not working. What should I do?**  
A: Check that the model file paths are correct and the files are complete. You can use `ls -lh` to verify that the files exist and their sizes are correct.
