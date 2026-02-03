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

1. **Automatic Download**: After selecting the Raspberry Pi platform and successful compilation, the model will be automatically downloaded to the `platform/LINUX/tuyaos_adapter/src/tkl_audio/models` directory.
2. **Manual Download**: Download the model files from the `tuyaos_adapter/src/tkl_audio/models` directory in the [TuyaOpen-ubuntu repository](https://github.com/tuya/TuyaOpen-ubuntu/tree/platform_ubuntu/tuyaos_adapter/src/tkl_audio/models).

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

For example, when storing the model files in the `/home/pi/Desktop/models/` directory, the path configuration is as follows:

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/eba887ff-d37f-4161-8f08-b641863ab9a6.png)

## Other Considerations

If you are using cross-compilation, you can use the scp command to transfer the compiled executable from the build host to the Raspberry Pi. For example:

```bash
scp ./dist/your_chat_bot_1.0.1/your_chat_bot_QIO_1.0.1.bin pi@192.168.1.xxx:/home/pi/Desktop/
```
