---
title: "Raspberry Pi Provisioning Troubleshooting"
sidebar_label: "WiFi/Bluetooth Troubleshooting"
slug: /hardware-specific/Linux/raspberry-pi/wifi-bluetooth
---

This page helps troubleshoot WiFi/Bluetooth issues when running TuyaOpen applications (for example `apps/tuya.ai/your_chat_bot`) on Raspberry Pi (Linux), such as: provisioning QR code not printed, Bluetooth unavailable, or WiFi/Bluetooth blocked by the OS.

## Prerequisites

Read the Quick Start and related sections first:

- Quick Start: https://tuyaopen.ai/docs/quick-start
- Set up the TuyaOpen development environment: https://tuyaopen.ai/docs/quick-start/enviroment-setup
- Get the TuyaOpen license key (recommended: authorize by modifying the header file): https://tuyaopen.ai/docs/quick-start/equipment-authorization
- Device provisioning: https://tuyaopen.ai/docs/quick-start/device-network-configuration

## Common symptoms

- The app does not enter provisioning, or the terminal does not print the provisioning QR code.
- Bluetooth scan/provisioning fails (BLE not working / device not found).
- WiFi or Bluetooth is disabled by the OS (shows blocked in `rfkill`).

## Provisioning QR code is not printed in the terminal

On Raspberry Pi, to print provisioning info (like QR codes) directly to the current terminal, enabling **fake UART (stdin/UDP)** is recommended.

1. Activate the `tos.py` environment and enter the application directory (using `your_chat_bot` as an example):

```bash
cd apps/tuya.ai/your_chat_bot
```

2. Open the configuration menu:

```bash
tos.py config menu
```

3. Navigate to `Choice a board → LINUX → TKL Board Configuration`, then enable:

- `Enable UART`
- `Use fake UART (stdin/UDP) instead of hardware ttyAMA*`

Example:

![Raspberry Pi fake UART config example](https://images.tuyacn.com/fe-static/docs/img/842c4b01-3d4b-487b-973d-4744e82935e9.png)

After enabling fake UART, the application typically prints the QR code to the current terminal. You can then use the Smart Life app to scan and provision.

## Check whether WiFi/Bluetooth is blocked (rfkill)

Run:

```bash
rfkill list
```

Example output:

```text
0: hci0: Bluetooth
        Soft blocked: no
        Hard blocked: no
1: phy0: Wireless LAN
        Soft blocked: yes
        Hard blocked: no
```

- `Soft blocked: yes` means the device is disabled at the software level.
- `Hard blocked: yes` usually indicates a physical switch / BIOS setting / platform-level lock.

## Unblock wireless devices

If you see `Soft blocked: yes`, unblock it and re-check:

```bash
sudo rfkill unblock all
rfkill list
```

## Enable Bluetooth service in TuyaOpen and disable NimBLE

In TuyaOpen Kconfig, enable the Bluetooth service and make sure the NimBLE stack is disabled.

Open the configuration menu:

```bash
tos.py config menu
```

Then navigate:

```text
configure tuyaopen  --->
  configure tuya cloud service  --->
    [*] ENABLE_BT_SERVICE: enable tuya bt iot function  --->
      [ ] ENABLE_NIMBLE: enable nimble stack instead of ble stack in board
```

- `ENABLE_BT_SERVICE` must be enabled.
- `ENABLE_NIMBLE` must be **disabled** (unchecked).

## Notes about sudo (administrator privileges)

Some operations require administrator privileges on Linux, for example:

- Unblocking devices via `rfkill`
- Managing system services like `bluetooth`
- Running binaries that access hardware via `/dev/*`

If you encounter `Permission denied`, try running the command with `sudo`.
