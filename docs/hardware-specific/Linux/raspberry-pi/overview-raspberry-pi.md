---
title: "Raspberry Pi Overview"
---

This document introduces how to run the tuyaopen [your_chat_bot](https://tuyaopen.ai/docs/applications/tuya.ai/demo-your-chat-bot) project on Raspberry Pi.

Raspberry Pi supports both cross-compilation and native compilation. The build system will automatically detect the current platform and select the appropriate compilation method.

Compared to the T5 platform, Raspberry Pi requires attention to the following two points:

1. Requires an external USB sound card
2. Requires manual configuration of model paths

## External Sound Card

Raspberry Pi does not have a built-in microphone or speaker by default, so an external USB sound card is required. The following model is recommended:

- **USB Audio Module YD1076**, [Taobao Link](https://e.tb.cn/h.77Vo2K5tJIaL86g?tk=lnBAUbwVNB9)

You can also choose other compatible USB sound cards. Note that the microphone should support raw audio data output and should not have built-in processing features such as noise reduction or echo cancellation.

## Model Path Configuration

### Obtain Wake Word Model

There are two ways to obtain the KWS wake word model:

**Option 1: Automatic download**

After selecting the Raspberry Pi platform and building successfully, the model is automatically downloaded to `platform/LINUX/tuyaos_adapter/src/tkl_audio/models`.

**Option 2: Manual download**

Use the following commands to download `mdtc_chunk_300ms.mnn` and `tokens.txt` to the `~/tuyaopen_models` directory:

```bash
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/mdtc_chunk_300ms.mnn
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/tokens.txt
```

### Configure Wake Word Model

After obtaining the model, configure the paths by following these steps:

1. Activate the tos.py environment and navigate to the `apps/tuya.ai/your_chat_bot` directory
2. Run the `tos.py config choice` command and select the `RaspberryPi.config` configuration
3. Run the `tos.py config menu` command and navigate to: `(Top) → Choice a board → LINUX → TKL Board Configuration`
4. Based on the actual storage paths of `mdtc_chunk_300ms.mnn` and `tokens.txt`, modify the following configuration items:
   - `KWS model file path`
   - `KWS model token file path`

> **Note**: Please ensure that the model files are placed in the Raspberry Pi's file system and the paths are configured correctly, otherwise the wake word function will not work properly.

### Configuration Example

For example, when the model files are stored in `~/tuyaopen_models`, the paths can be configured as follows:

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/4e3897a7-6d32-40e2-b2bd-6d2a6497076e.png)

> **Important**: The paths must be correct, or the voice wake word feature will not work.

## Additional Notes

### Transferring Files When Using Cross-Compilation

If you are using cross-compilation, you can use the `scp` command to copy the built executable from the build host to the Raspberry Pi. For example:

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin username@192.168.1.xxx:/home/xx/Desktop/
```

**Command parameters:**
- `username` — Username on the board
- `192.168.1.xxx` — IP address of the board
- `/home/xx/Desktop/` — Target directory on the board

## FAQ

**Q: Voice wake word is not working. What should I do?**  
A: Check that the model file paths are correct and that the files are complete. You can use `ls -lh` to verify that the files exist and that their sizes are as expected.
