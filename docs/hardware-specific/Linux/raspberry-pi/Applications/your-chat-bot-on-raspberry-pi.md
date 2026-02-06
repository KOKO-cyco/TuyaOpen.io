---
title: "Running your_chat_bot on Raspberry Pi"
---

This document describes how to run the [your_chat_bot](https://tuyaopen.ai/docs/applications/tuya.ai/demo-your-chat-bot) project from tuyaopen on a Raspberry Pi.

Raspberry Pi supports both cross-compilation and native compilation. The build process automatically detects the current platform and selects the appropriate compilation method.

When using a Raspberry Pi, note the following:

1. An external USB sound card is required (the sound card must support both audio input and output)
2. The model path must be configured manually

## External Sound Card

The Raspberry Pi does not have a built-in microphone or speaker by default, so an external USB sound card is required. The following models are recommended:

- **USB Audio Module YD1076**, [Taobao link](https://e.tb.cn/h.77Vo2K5tJIaL86g?tk=lnBAUbwVNB9)
- **Waveshare USB Sound Card**, [link](https://www.waveshare.com/wiki/USB_TO_AUDIO?srsltid=AfmBOoqQpLSG-qO8REhn6lDsAIOOjskHyjkyJv0_4BKBo3_vqFqoTisL)

You may also choose other compatible USB sound cards. Note that the microphone should support outputting raw audio data and should not include built-in noise reduction, echo cancellation, or similar processing.

## Model Path Configuration

### Obtaining the Wake Word Model

The KWS (Keyword Spotting) wake word model can be obtained in two ways:

**Option 1: Automatic download**

After selecting the Raspberry Pi platform and building successfully, the model is automatically downloaded to `platform/LINUX/tuyaos_adapter/src/tkl_audio/models`.

**Option 2: Manual download**

Use the following commands to download `mdtc_chunk_300ms.mnn` and `tokens.txt` to the `~/tuyaopen_models` directory:

```bash
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/mdtc_chunk_300ms.mnn
wget -P ~/tuyaopen_models https://github.com/tuya/TuyaOpen-ubuntu/raw/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models/tokens.txt
```

The default path in the code is `~/tuyaopen_models`. If you need to change it, follow the steps below.

### Configuring the Wake Word Model

After obtaining the model, configure the path as follows:

1. Activate the tos.py environment and go to the `apps/tuya.ai/your_chat_bot` directory
2. Run `tos.py config choice` and select the `RaspberryPi.config` configuration
3. Run `tos.py config menu` and navigate: `(Top) → Choice a board → LINUX → TKL Board Configuration`
4. According to the actual location of `mdtc_chunk_300ms.mnn` and `tokens.txt`, set the following options:
   - `KWS model file path`
   - `KWS model token file path`

> **Note**: Ensure that the model files are placed in the Raspberry Pi filesystem and that the paths are configured correctly; otherwise, the wake word feature will not work.

### Configuration Example

For example, when the model files are stored in `~/tuyaopen_models`, the paths can be configured as follows:

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/4e3897a7-6d32-40e2-b2bd-6d2a6497076e.png)

> **Important**: The paths must be correct, or the voice wake word feature will not work.

## Additional Notes

### Transferring Files When Using Cross-Compilation

If you are using cross-compilation, you can use the `scp` command to copy the built executable from the build host to the Raspberry Pi. For example:

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin username@192.168.1.xxx:~/
```

**Command parameters:**
- `username` — Username on the board
- `192.168.1.xxx` — IP address of the board
- `~/` — Target directory on the board

## FAQ

**Q: Voice wake word is not working. What should I do?**  
A: Check that the model file paths are correct and that the files are complete. You can use `ls -lh` to verify that the files exist and that their sizes are as expected.
