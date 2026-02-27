---
title: "Raspberry Pi Peripherals"
---

This document explains how to run TuyaOpen peripheral examples (`examples/peripherals`) on Raspberry Pi, including GPIO, I2C, SPI, PWM, and UART.

## Quick Start

1. Make sure you have completed the TuyaOpen base environment setup and are in the TuyaOpen repository root.
2. Open the configuration UI:
  - Run `tos.py config menu`
  - Select the board: `Choice a board → LINUX → Choice a board → RaspberryPi`
  - Select the model: `Raspberry Pi Board Configuration → Choose Raspberry Pi model → Raspberry Pi 5` (choose according to your actual model)
3. Enable peripherals as needed: go to `Choice a board → LINUX → TKL Board Configuration`, and check `ENABLE_GPIO`/`ENABLE_I2C`/`ENABLE_SPI`/`ENABLE_PWM`/`ENABLE_UART`.

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/4b6127c5-ab9f-415a-b365-cb136467efed.png)

4. Go to the corresponding example directory (e.g., `examples/peripherals/gpio`), run `tos.py build`, then run the generated `*.elf` with `sudo`.

> **Note (build mode)**: Raspberry Pi supports both cross-compilation and native compilation; the build system automatically chooses a suitable mode based on the current platform.

## General Notes

- **Permissions**: Peripheral examples usually need to access `/dev/*` or `/sys/class/*`. When running on Raspberry Pi, it is recommended to use `sudo`.
- **Device nodes**: Peripheral node names may differ across OS images (for example, UART can be `/dev/ttyAMA0` or `/dev/ttyS0`). If the nodes do not match TuyaOpen port mappings, adapt to the real nodes or adjust configuration accordingly.
- **`OPRT_NOT_SUPPORTED`**: Some TKL peripheral APIs are kept for a unified abstraction across MCU/Linux. On Raspberry Pi (Linux user space), if the underlying standard interfaces (e.g., i2c-dev/spidev/sysfs/tty/gpio-cdev) cannot provide the capability, or the capability requires additional kernel drivers/subsystems and the current adaptation is not implemented, the API will return `OPRT_NOT_SUPPORTED`.

## GPIO Example

This example shows how to operate GPIO on Raspberry Pi using TuyaOpen.

### Adaptation Notes (Linux TKL GPIO)

#### Supported (available)

- Basic read/write
  - `tkl_gpio_init()` / `tkl_gpio_deinit()`: request/release a line handle based on Linux gpio-cdev (`/dev/gpiochip*`).
  - `tkl_gpio_write()` / `tkl_gpio_read()`: write/read the level via `GPIOHANDLE_*` ioctls.
- Interrupt callback (event notification)
  - `tkl_gpio_irq_init()` / `tkl_gpio_irq_enable()` / `tkl_gpio_irq_disable()`: request an event fd via `GPIO_GET_LINEEVENT_IOCTL`, and a thread uses `poll()` to monitor and invoke the callback.

#### Notes/Limitations

- Requires `/dev/gpiochip*` provided by the system (kernel must enable the GPIO character device interface, and the current user must have access; usually run examples with `sudo`).
- On Linux, `TUYA_GPIO_NUM_E` is used as the gpiochip line offset. On Raspberry Pi it is usually consistent with BCM GPIO numbering, but it may vary with distro/kernel configuration; verify with `gpioinfo`/`pinctrl`.
- `TUYA_GPIO_IRQ_LOW/HIGH` is an “approximation”: the implementation listens for edge events, then reads the current level and filters; it is not the same as hardware level-triggered interrupts.

#### Reference

- For GPIO API definitions, parameter descriptions, and adaptation notes, see [GPIO Driver](https://tuyaopen.ai/zh/docs/tkl-api/tkl_gpio).

### Enter the Example Directory

```bash
cd examples/peripherals/gpio
```

### Configuration

Start the configuration UI:

```bash
tos.py config menu
```

After completing board/model selection per “Quick Start”, go to: `Choice a board → LINUX → TKL Board Configuration` and check `ENABLE_GPIO`.

> 
> **Tip**: For GPIO pinout and the RP1 multiplexing function table, see the [Raspberry Pi 5 GPIO Reference Manual](https://tuyaopen.ai/zh/docs/hardware-specific/Linux/raspberry-pi/Examples/raspberry-pi.md).

In `Application config`, choose suitable pins for:

- output pin
- input pin
- irq pin

Make sure the selected pins are free and match your hardware wiring.

### Build and Run

Build:

```bash
tos.py build
```

After building, an executable like `gpio_1.0.0.elf` will be generated. Run on Raspberry Pi:

```bash
sudo ./gpio_1.0.0.elf
```

### Minimal Example

The code below demonstrates:

- initializing an output pin and toggling its level once per second
- initializing an input pin and reading its level

> Note: the snippet only shows the core calls. For a complete buildable project, refer to `examples/peripherals/gpio`.

```c
#include "tal_api.h"
#include "tkl_gpio.h"

// These two macros are usually configured via Kconfig/Application config in the example project
// #define EXAMPLE_OUTPUT_PIN ...
// #define EXAMPLE_INPUT_PIN  ...

static void gpio_min_demo(void)
{
  TUYA_GPIO_BASE_CFG_T out_cfg = {
    .mode   = TUYA_GPIO_PUSH_PULL,
    .direct = TUYA_GPIO_OUTPUT,
    .level  = TUYA_GPIO_LEVEL_LOW,
  };
  TUYA_GPIO_BASE_CFG_T in_cfg = {
    .mode   = TUYA_GPIO_PULLUP,
    .direct = TUYA_GPIO_INPUT,
  };

  tkl_gpio_init(EXAMPLE_OUTPUT_PIN, &out_cfg);
  tkl_gpio_init(EXAMPLE_INPUT_PIN,  &in_cfg);

  while (1) {
    static uint8_t level = 0;
    TUYA_GPIO_LEVEL_E in_level = TUYA_GPIO_LEVEL_LOW;

    level ^= 1;
    tkl_gpio_write(EXAMPLE_OUTPUT_PIN, level ? TUYA_GPIO_LEVEL_HIGH : TUYA_GPIO_LEVEL_LOW);

    tkl_gpio_read(EXAMPLE_INPUT_PIN, &in_level);
    PR_NOTICE("GPIO in=%d out=%d", (int)in_level, (int)level);

    tal_system_sleep(1000);
  }
}
```

## I2C Example

This section shows how to operate I2C on Raspberry Pi using TuyaOpen.

### Adaptation Notes (Linux TKL I2C)

#### Supported (available)

- Basic master send/receive
  - `tkl_i2c_master_send()`: write to a device address (uses `/dev/i2c-X` + `I2C_SLAVE` + `write()`).
  - `tkl_i2c_master_receive()`: read from a device address (uses `I2C_SLAVE` + `read()`).
- Common register-read combined transaction (Repeated Start)
  - If `tkl_i2c_master_send(..., xfer_pending=true)` is immediately followed by `tkl_i2c_master_receive()`, they are merged into one `I2C_RDWR` transaction to implement “write register address/command, then repeated-start read data”.
- Address probing (scan)
  - When `tkl_i2c_master_send()` has `size==0`, it uses SMBus “quick” to probe if a device ACKs (useful for simple address scans).

#### Not supported yet (API reserved; current implementation returns `OPRT_NOT_SUPPORTED`)

- Slave mode: `tkl_i2c_set_slave_addr()`, `tkl_i2c_slave_send()`, `tkl_i2c_slave_receive()`.
- Interrupt/event callbacks: `tkl_i2c_irq_init()`, `tkl_i2c_irq_enable()`, `tkl_i2c_irq_disable()`.
- Extended control/status: `tkl_i2c_ioctl()`, `tkl_i2c_get_status()`.
  - Note: `tkl_i2c_get_status()` currently clears the output struct and returns `OPRT_NOT_SUPPORTED`; do not rely on its output fields.

#### Reference

- For I2C API definitions, parameter descriptions, and adaptation notes, see [I2C Driver](https://tuyaopen.ai/zh/docs/tkl-api/tkl_i2c).

### Enable I2C on Raspberry Pi (System Configuration)

Run on Raspberry Pi:

```bash
sudo raspi-config
```

In `raspi-config`, enable I2C via:

- `3 Interface Options` → `I5 I2C` → `Enable`

![models_path_config](https://images.tuyacn.com/fe-static/docs/img/c8daf0da-c625-472e-888f-090968719dc9.png)

Verify the device node exists:

```bash
ls /dev | grep i2c
```

### Example 1: Scan the Bus (i2c_scan)

Enter the example directory:

```bash
cd examples/peripherals/i2c/i2c_scan
```

Configuration:

```bash
tos.py config menu
```

- `Choice a board → LINUX → TKL Board Configuration`: check `ENABLE_I2C`
- `Application config`: configure `i2c port`, `sda pin`, `scl pin`

Notes:

- The Linux adaptation accesses `/dev/i2c-${port}`.
- On Raspberry Pi, it is commonly `/dev/i2c-1` (GPIO2/3), so `i2c port` usually needs to match the actual node index.

Build and run:

```bash
tos.py build
sudo ./i2c_scan_1.0.0.elf
```

If devices are found, logs similar to the following will be printed:

- `[example_i2c_scan.c:xx] i2c device found at address: 0x44`

### Minimal Example

The code below demonstrates scanning I2C 7-bit addresses (in the Linux adaptation, `size==0` uses “quick” probing):

> Note: for a complete buildable project, refer to `examples/peripherals/i2c/i2c_scan`.

```c
#include "tal_api.h"
#include "tkl_i2c.h"

static void i2c_scan_demo(TUYA_I2C_NUM_E port)
{
  for (uint8_t addr = 0x08; addr <= 0x77; addr++) {
    // size=0: probing
    if (tkl_i2c_master_send(port, addr, NULL, 0, TRUE) == OPRT_OK) {
      PR_NOTICE("I2C device found: 0x%02X", addr);
    }
  }
}
```

### Example 2: Read Temperature & Humidity (sht3x_4x_sensor)

Enter the example directory:

```bash
cd examples/peripherals/i2c/sht3x_4x_sensor
```

Use the same configuration and build steps as above. In **Application config**, choose:

- `sensor type`: sht3x or sht4x

Run:

```bash
sudo ./sht3x_4x_sensor_1.0.0.elf
```

You should see periodic temperature and humidity logs.

## SPI Example

This example shows how to operate SPI on Raspberry Pi using TuyaOpen (user-space spidev).

### Adaptation Notes (Linux TKL SPI)

#### Supported (available)

- Master mode
  - `tkl_spi_init()`: open `/dev/spidevX.Y` and configure mode/bits/speed/bitorder.
  - Only `TUYA_SPI_ROLE_MASTER` / `TUYA_SPI_ROLE_MASTER_SIMPLEX` is supported.
- Basic send/receive
  - `tkl_spi_send()`: uses `write()`.
  - `tkl_spi_recv()`: uses `read()`.
- Transfer
  - `tkl_spi_transfer()`: full-duplex TX/RX via `SPI_IOC_MESSAGE(1)`.
  - `tkl_spi_transfer_with_length()`: “send then receive” via `SPI_IOC_MESSAGE(2)`.
- Counters & status (compatibility APIs)
  - `tkl_spi_get_data_count()`: returns the number of bytes transferred in the most recent transfer.
  - `tkl_spi_get_status()`: returns `OPRT_OK` and currently only clears the struct (does not provide real status).

#### Not supported yet (API reserved; current implementation returns `OPRT_NOT_SUPPORTED`)

- Interrupt callbacks: `tkl_spi_irq_init()` / `tkl_spi_irq_enable()` / `tkl_spi_irq_disable()`.
- Extended control: `tkl_spi_ioctl()`.

#### Behavioral limitations / compatibility implementations

- Abort transfer: `tkl_spi_abort_transfer()` returns `OPRT_OK` but does not perform a real abort.
- DMA length: `tkl_spi_get_max_dma_data_length()` returns 0 (not meaningful under Linux spidev).

#### Port-to-device-node mapping (default)

| spi port | device node |
| --- | --- |
| 0 | `/dev/spidev0.0` |
| 1 | `/dev/spidev0.1` |
| 2 | `/dev/spidev1.0` |
| 3 | `/dev/spidev1.1` |
| 4 | `/dev/spidev2.0` |
| 5 | `/dev/spidev2.1` |

#### Reference

- For SPI API definitions, parameter descriptions, and adaptation notes, see [SPI Driver](https://tuyaopen.ai/zh/docs/tkl-api/tkl_spi).

### Enable SPI on Raspberry Pi (System Configuration)

```bash
sudo raspi-config
```

In `raspi-config`, enable SPI via:

- `3 Interface Options` → `I4 SPI` → `Enable`

Verify the device node exists:

```bash
ls /dev | grep spidev
```

> In TuyaOpen SPI examples, `Application config -> spi port` is a **port number**.
> The Linux adaptation maps the port number to an actual device node (see `platform/LINUX/tuyaos_adapter/src/tkl_spi.c` `prv_spi_dev_path()`):
>
> - `spi port = 0` → `/dev/spidev0.0`
> - `spi port = 1` → `/dev/spidev0.1`
> - `spi port = 2` → `/dev/spidev1.0`
> - `spi port = 3` → `/dev/spidev1.1`
> - `spi port = 4` → `/dev/spidev2.0`
> - `spi port = 5` → `/dev/spidev2.1`
>
> For example, for `spidev0.0 / spidev0.1`, set `spi port` to `0 / 1`.

### Enter the Example Directory

```bash
cd examples/peripherals/spi
```

### Configure, Build, and Run

Configuration:

```bash
tos.py config menu
```

- `Choice a board → LINUX → TKL Board Configuration`: check `ENABLE_SPI`
- `Application config`: configure `spi port`, `spi baudrate`

Suggested `spi port` selection:

- Use `/dev/spidev0.0`: set to `0`
- Use `/dev/spidev0.1`: set to `1`

For `spi baudrate` (Hz), it is recommended to start with `1000000` or `8000000` to validate loopback/communication first, then increase gradually according to your peripheral’s capability.

Build and run:

```bash
tos.py build
sudo ./spi_1.0.0.elf
```

### Minimal Example

The code below demonstrates SPI master sending a fixed string (on Linux it uses `/dev/spidevX.Y`):

> Note: for a complete buildable project, refer to `examples/peripherals/spi`.

```c
#include "tal_api.h"
#include "tkl_spi.h"

// #define EXAMPLE_SPI_PORT ...
// #define EXAMPLE_SPI_BAUDRATE ...

static void spi_min_demo(void)
{
  TUYA_SPI_BASE_CFG_T cfg = {
    .mode     = TUYA_SPI_MODE0,
    .freq_hz  = EXAMPLE_SPI_BAUDRATE,
    .databits = TUYA_SPI_DATA_BIT8,
    .bitorder = TUYA_SPI_ORDER_LSB2MSB,
    .role     = TUYA_SPI_ROLE_MASTER,
    .type     = TUYA_SPI_AUTO_TYPE,
  };

  uint8_t tx[] = "Hello Tuya";
  tkl_spi_init(EXAMPLE_SPI_PORT, &cfg);

  while (1) {
    tkl_spi_send(EXAMPLE_SPI_PORT, tx, sizeof(tx));
    tal_system_sleep(500);
  }
}
```

## PWM Example

This example shows how to operate PWM on Raspberry Pi using TuyaOpen.

### Adaptation Notes (Linux TKL PWM)

#### Supported (available)

- PWM output (`/sys/class/pwm`)
  - `tkl_pwm_init()`: export the channel and configure polarity/period/duty.
  - `tkl_pwm_start()` / `tkl_pwm_stop()`: start/stop via `enable`.
  - `tkl_pwm_duty_set()`: update duty cycle.
  - `tkl_pwm_frequency_set()`: update frequency.
  - `tkl_pwm_polarity_set()`: update polarity.
  - `tkl_pwm_info_set()` / `tkl_pwm_info_get()`: set/get the whole parameter set (get returns the cfg cached in software).
  - `tkl_pwm_multichannel_start()` / `tkl_pwm_multichannel_stop()`: start/stop multiple channels sequentially.
  - `tkl_pwm_deinit()`: stop and unexport.

#### Not supported yet (API reserved; current implementation returns `OPRT_NOT_SUPPORTED`)

- PWM capture: `tkl_pwm_cap_start()` / `tkl_pwm_cap_stop()`.
  - Note: currently returns `OPRT_NOT_SUPPORTED`.

#### Reference

- For PWM API definitions, parameter descriptions, and adaptation notes, see [PWM Driver](https://tuyaopen.ai/zh/docs/tkl-api/tkl_pwm).

### PWM Lab Steps (Example: output PWM square wave on GPIO18)

#### Enter the Example Directory

```bash
cd examples/peripherals/pwm
```

#### Enable PWM on Raspberry Pi (System Configuration)

1. Confirm the pin is not occupied:

   ```bash
   pinctrl get 18
   ```

   If it is not multiplexed, you will typically see output like:

   - `18: no    pd | -- // GPIO18 = none`

2. Enable the PWM overlay:

   Add the following to the end of `/boot/firmware/config.txt`:

   ```text
   dtoverlay=pwm,pin=18,func=2
   ```

   Reboot the Raspberry Pi for it to take effect.

3. After reboot, confirm the mapping:

   ```bash
   pinctrl get 18
   ```

   You should see output like the following (indicating it has switched to a PWM channel):

   - `18: a3    pd | lo // GPIO18 = PWM0_CHAN2`

#### Configuration

Start the configuration UI:

```bash
tos.py config menu
```

- `Choice a board → LINUX → TKL Board Configuration`: check `ENABLE_PWM`
- In the same configuration tree, set:
  - `PWM_SYSFS_CHIP = 0` (corresponds to `/sys/class/pwm/pwmchip0`)
  - `PWM_SYSFS_CHANNEL_BASE = 2` (because GPIO18 maps to `PWM0_CHAN2`)
- `Application config`: choose `pwm port = 0` (because it is `PWM0`)

#### Build and Run

```bash
tos.py build
sudo ./pwm_1.0.0.elf
```

### Minimal Example

The code below demonstrates PWM output (init + start):

> Note: for a complete buildable project, refer to `examples/peripherals/pwm`.

```c
#include "tal_api.h"
#include "tkl_pwm.h"

// #define EXAMPLE_PWM_PORT ...
// #define EXAMPLE_PWM_FREQUENCY ...
// #define EXAMPLE_PWM_DUTY ... // 1-10000

static void pwm_min_demo(void)
{
  TUYA_PWM_BASE_CFG_T cfg = {
    .duty      = EXAMPLE_PWM_DUTY,
    .frequency = EXAMPLE_PWM_FREQUENCY,
    .polarity  = TUYA_PWM_NEGATIVE,
  };

  tkl_pwm_init(EXAMPLE_PWM_PORT, &cfg);
  tkl_pwm_start(EXAMPLE_PWM_PORT);

  while (1) {
    tal_system_sleep(2000);
  }
}
```

If you want to quickly verify whether the sysfs node matches expectations, check whether the corresponding `pwm2` exists (or can be exported) under `/sys/class/pwm/pwmchip0/`.

> **Tip**: PWM sysfs depends on kernel/overlay configuration; the path to `/boot/firmware/config.txt` may differ across images. Please follow your actual system.


## UART Example

This example shows how to operate UART on Raspberry Pi using TuyaOpen.

### Adaptation Notes (Linux TKL UART)

#### Supported (available)

- Basic send/receive
  - `tkl_uart_init()`: open the serial device and configure baud/data bits/parity/stop bits via termios.
  - `tkl_uart_write()`: send via `write()`.
  - `tkl_uart_read()`: receive via `read()`.
  - `tkl_uart_deinit()`: close fd and stop the receive thread.
- RX callback notification (approximate “interrupt” semantics)
  - `tkl_uart_rx_irq_cb_reg()`: register an RX callback.
  - On Linux, a thread uses `select()` to wait for fd readability and invokes the callback.

#### Not supported yet (API reserved; current implementation returns `OPRT_NOT_SUPPORTED`)

- `tkl_uart_set_tx_int()` / `tkl_uart_set_rx_flowctrl()` / `tkl_uart_wait_for_data()` / `tkl_uart_ioctl()`.

#### Empty implementation (no effect)

- `tkl_uart_tx_irq_cb_reg()`: currently an empty implementation.

#### Device node mapping (related to the FAKE UART switch)

- When `TKL_UART_REDIRECT_LOG_TO_STDOUT = n` (UART redirection disabled; use real hardware UART), the default mapping is:
  - `port 0 -> /dev/ttyAMA0`
  - `port 1 -> /dev/ttyAMA1`
  - `port 2 -> /dev/ttyAMA2`
- When `TKL_UART_REDIRECT_LOG_TO_STDOUT = y` (UART redirection enabled), it will not access `/dev/ttyAMA*`, but uses a Dummy UART implementation instead (see below).

#### Reference

- For UART API definitions, parameter descriptions, and adaptation notes, see [UART Driver](https://tuyaopen.ai/zh/docs/tkl-api/tkl_uart).

### Hardware Wiring Notes (Physical UART)

If you use a **physical UART** (for example, Raspberry Pi UART pins connected to a USB-TTL module or another board’s UART):

- You must connect **GND to GND** between both sides. Without a common ground, typical symptoms include garbled RX data, missing bytes, or extremely unstable communication.

### UART Redirection (Dummy UART: stdin/stdout/UDP)

To make UART-related components work even **without real UART hardware**, the Linux platform provides a `TKL_UART_REDIRECT_LOG_TO_STDOUT` switch (in `TKL Board Configuration` under LINUX).

When redirection (Dummy UART) is enabled, the behavior of `tkl_uart.c` is roughly as follows (different from real UART; mainly for debug/demo):

- `TUYA_UART_NUM_0` (port 0):
  - RX: reads from the current process standard input `/dev/stdin` (i.e., the keyboard input in the terminal where you run `*.elf`).
  - TX: writes to standard output `stdout` (printed directly in the terminal).
  - Typical use: in SSH/local terminal, “keyboard input → UART RX”, and “UART TX” shows on screen, without relying on `/dev/ttyAMA*`.
- `TUYA_UART_NUM_1` (port 1):
  - RX: receives data via a UDP socket and feeds it byte-by-byte to the upper-layer RX callback.
  - TX: sends data to the peer via UDP socket.
  - Note: in the current implementation, the UDP bind/send IP and ports are fixed (environment-dependent). In different networks you usually need to modify the adaptation layer source and rebuild.

Limitations/notes for Dummy mode:

- Baud rate/parity/stop bits and other termios configs are **not equivalent to real UART** in Dummy mode (port 0 only sets stdin to non-canonical mode for immediate character reads; stdout does not have real UART timing).
- Dummy is mainly for “logic validation/interactive demos”, not for serious UART protocol timing verification.

How to choose whether to enable UART redirection:

- If you want UART examples/CLI to use Raspberry Pi real UART pins (`/dev/ttyAMA*` or `/dev/ttyS*`), go to:
  - `Choice a board → LINUX → TKL Board Configuration`
  - Set `UART redirection (stdin/stdout/UDP) instead of hardware ttyAMA*` to `n`
  - Note: when this option is **not checked**, it uses physical UART (accesses real `ttyAMA*`/`ttyS*` device nodes).
- If you just want to quickly validate UART logic and do not have USB-TTL/hardware loopback wiring yet, you can keep it as `y`.

#### Note: QR-code output channel during `your_chat_bot` provisioning (UART redirection)

When running `your_chat_bot` on Linux/Raspberry Pi for provisioning demos, it is recommended to enable `TKL_UART_REDIRECT_LOG_TO_STDOUT` so the QR code content is printed directly in the current terminal.

**1) Expected behavior (redirection enabled)**

- You can directly see the QR code (QR code string/ASCII art) in the terminal where you run `your_chat_bot*.elf`.

**2) How it works (output via UART0 TX)**

- During provisioning, `your_chat_bot` typically sends QR code content via UART0 (`TUYA_UART_NUM_0`).
- When `TKL_UART_REDIRECT_LOG_TO_STDOUT = y`: UART0 TX maps to `stdout`, so the QR code shows in the current terminal.
- When `TKL_UART_REDIRECT_LOG_TO_STDOUT = n`: UART0 TX writes to the real serial device (e.g., `/dev/ttyAMA0`/`/dev/ttyS0`), so the QR code does not appear in the terminal and is output on the serial line.

### Enable UART on Raspberry Pi (System Configuration)

```bash
sudo raspi-config
```

In `raspi-config`, configure serial via:

- `3 Interface Options` → `I6 Serial Port`

Usually it is recommended to select:

- **Disable serial login shell**
- **Enable serial port hardware**

Check serial device nodes:

```bash
ls -l /dev/ttyAMA* /dev/ttyS* 2>/dev/null
```

### Enter the Example Directory

```bash
cd examples/peripherals/uart
```

### Configure, Build, and Run

Configuration:

```bash
tos.py config menu
```

- `Choice a board → LINUX → TKL Board Configuration`: check `ENABLE_UART`

Optional: in the same menu, set `UART redirection (stdin/stdout/UDP) instead of hardware ttyAMA*` as needed (enable/disable UART redirection / Dummy UART).

- Checked (`*`): enable UART redirection (Dummy UART: stdin/stdout/UDP), no dependency on real hardware UART device nodes.
- Unchecked (` `): use physical UART (access real `ttyAMA*`/`ttyS*` device nodes).

Build:

```bash
tos.py build
```

Run:

```bash
sudo ./uart_1.0.0.elf
```

> Reminder: the example code uses `TUYA_UART_NUM_0` (UART0) by default. On Raspberry Pi, UART0 may be occupied by the system console. If there is no echo or open fails, check serial console usage and adjust the example UART port or the adaptation layer device node mapping accordingly.

### Minimal Example 1: Interactive Echo

This demo is best for quickly validating the UART path in **Dummy UART redirection** (stdin/stdout) mode: whatever you type in the terminal will be echoed back.

> Note: this approach is consistent with `examples/peripherals/uart`.

```c
#include "tal_api.h"

#include "tkl_output.h"

#define UART_NUM TUYA_UART_NUM_0

static void uart_echo_demo(void)
{
  TAL_UART_CFG_T cfg = {0};
  cfg.base_cfg.baudrate = 115200;
  cfg.base_cfg.databits = TUYA_UART_DATA_LEN_8BIT;
  cfg.base_cfg.stopbits = TUYA_UART_STOP_LEN_1BIT;
  cfg.base_cfg.parity   = TUYA_UART_PARITY_TYPE_NONE;
  cfg.rx_buffer_size    = 256;
  cfg.open_mode         = O_BLOCK;

  tal_uart_init(UART_NUM, &cfg);
  tal_uart_write(UART_NUM, (const uint8_t*)"Please input text:\r\n", sizeof("Please input text:\r\n") - 1);

  while (1) {
    uint8_t buf[128];
    int n = tal_uart_read(UART_NUM, buf, sizeof(buf));
    if (n > 0) {
      tal_uart_write(UART_NUM, buf, n);
    } else {
      tal_system_sleep(10);
    }
  }
}
```

### Minimal Example 2: Hardware Loopback Self-Test (Short TX to RX)

This demo is used to verify whether sent data can be read back unchanged (`memcmp` check). It typically requires:

- Disable Dummy redirection (use physical serial device node)
- Short **TX and RX** of the same UART (and ensure GND is common)

```c
#include <string.h>

#include "tal_api.h"
#include "tkl_uart.h"

static OPERATE_RET uart_loopback_test(TUYA_UART_NUM_E port)
{
  TUYA_UART_BASE_CFG_T cfg = {0};
  cfg.baudrate = 115200;
  cfg.databits = TUYA_UART_DATA_LEN_8BIT;
  cfg.parity   = TUYA_UART_PARITY_TYPE_NONE;
  cfg.stopbits = TUYA_UART_STOP_LEN_1BIT;
  cfg.flowctrl = TUYA_UART_FLOWCTRL_NONE;

  OPERATE_RET ret = tkl_uart_init(port, &cfg);
  if (ret != OPRT_OK) {
    return ret;
  }

  const uint32_t timeout_ms = 5000;
  const int bufsize = 8;
  uint8_t tx[bufsize];
  uint8_t rx[bufsize];

  for (int i = 0; i < bufsize; i++) {
    tx[i] = (uint8_t)('A' + i);
  }

  for (int round = 0; round < 3; round++) {
    memset(rx, 0, sizeof(rx));

    int wr = tkl_uart_write(port, tx, sizeof(tx));
    if (wr != (int)sizeof(tx)) {
      ret = OPRT_COM_ERROR;
      break;
    }

    int got = 0;
    SYS_TIME_T start = tal_system_get_millisecond();
    while (got < (int)sizeof(rx)) {
      SYS_TIME_T now = tal_system_get_millisecond();
      if ((uint32_t)(now - start) > timeout_ms) {
        ret = OPRT_TIMEOUT;
        break;
      }

      int rd = tkl_uart_read(port, rx + got, (uint32_t)sizeof(rx) - (uint32_t)got);
      if (rd > 0) {
        got += rd;
      } else {
        tal_system_sleep(5);
      }
    }

    if (ret != OPRT_OK) {
      break;
    }
    if (memcmp(tx, rx, sizeof(tx)) != 0) {
      ret = OPRT_COM_ERROR;
      break;
    }
  }

  tkl_uart_deinit(port);
  return ret;
}
```

## Button Example

This example demonstrates how to handle button input on Raspberry Pi using TuyaOpen’s Button component (TDL Button management layer).

### Adaptation Notes (Raspberry Pi: keyboard-simulated button)

On Raspberry Pi, buttons are simulated via keyboard input by default: pressing a character in the terminal where you run `*.elf` triggers a button event.

- Enabled by board Kconfig: `ENABLE_KEYBOARD_INPUT`
- Trigger character is specified by `BUTTON_NAME` (default `s`)

> Note: in the example project, the log for `TDL_BUTTON_PRESS_DOWN` is printed as `single click` (see `examples/peripherals/button/src/example_button.c`). It represents the “press down” event.

### Enter the Example Directory

```bash
cd examples/peripherals/button
```

### Configuration

```bash
tos.py config choice
```

Select the number corresponding to `RaspberryPi.config` and press Enter.

```bash
tos.py config menu
```

After completing board/model selection per “Quick Start”, go to:

- `Choice a board → LINUX → Raspberry Pi Board Configuration`
  - Confirm `Enable keyboard input for Raspberry Pi` is checked.
  - Set `Keyboard button device value`, for example: `s`

Note: `Keyboard button device value` corresponds to the board config item `BUTTON_NAME`, meaning “which keyboard character is used to simulate the button”.

- Set to `s`: pressing `s` in the terminal running `*.elf` triggers a button event named `s`.
- It is recommended to use a **single character** (e.g., `s` / `a` / `d` / `1`). Avoid multi-character strings to prevent confusion if some implementations only take the first character.

### Build and Run

Build:

```bash
tos.py build
```

Run:

```bash
sudo ./button_1.0.0.elf
```

### Expected Behavior

- Press the character corresponding to `BUTTON_NAME` (default `s`) in the terminal, and it prints `s: single click` once (press-down event).
- Hold it for about 3 seconds (in the example, `long_start_valid_time=3000ms`), and it prints `s: long press` (long-press event).


## Audio Codecs Example (audio_codecs)

This example demonstrates **recording + playback** via ALSA on Raspberry Pi (PCM 16k/16bit/mono), and shows how to use TuyaOpen’s `TDL Audio` management-layer APIs.

### Adaptation Notes (Linux ALSA)

- On Raspberry Pi (Linux), audio is accessed via ALSA `/dev/snd/*`.
- This example depends on the `src/peripherals/audio_codecs` component and uses the ALSA driver implementation (`tdd_audio_alsa.c`).
- It is recommended to run with `sudo`, or ensure the current user is in the `audio` group (otherwise opening sound card device nodes may fail).

### Pre-check (Verify USB Sound Card Is Recognized)

Using a USB audio module (e.g., YD1076/Y1076) as an example, run on Raspberry Pi:

```bash
aplay -l
arecord -l
ls -la /dev/snd/
```

You should see a device like `card 2: Y1076 ...` in the list.

### Enter the Example Directory

```bash
cd examples/peripherals/audio_codecs
```

### Configuration

Open the configuration UI:


```bash
tos.py config choice
```

Select the number corresponding to `RaspberryPi.config` and press Enter.

```bash
tos.py config menu
```

After completing board selection per “Quick Start”, go to:

- `Choice a board → LINUX → Choice a board → RaspberryPi → Raspberry Pi Board Configuration`
  - Confirm `Enable keyboard input for Raspberry Pi` is checked.
  - Set `Keyboard button device value`, for example: `s`

### Build and Run

Build:

```bash
tos.py build
```

Run (recommended with `sudo` on Raspberry Pi):

```bash
sudo ./audio_codecs_1.0.0.elf
```

Interaction: the example uses keyboard input to simulate a button by default (usually `s`). Press and hold to start recording, release to stop recording and play back (follow actual logs/behavior).

### Common Troubleshooting

1) **Failed to open `default` device**

If you see errors like:

- `ALSA lib pcm_asym.c:... capture slave is not defined`
- `Audio capture device 'default' not available: Invalid argument`

It indicates that the ALSA `default` PCM config on the current system cannot be used for recording.

Possible solution:

- Create `/etc/asound.conf` on Raspberry Pi and map `default` to the USB sound card:

```bash
sudo tee /etc/asound.conf >/dev/null <<'EOF'
pcm.!default {
    type asym
    playback.pcm "plughw:CARD=Y1076,DEV=0"
    capture.pcm  "plughw:CARD=Y1076,DEV=0"
}

ctl.!default {
    type hw
    card "Y1076"
}
EOF
```

After creating it, you can validate with:

```bash
arecord -D default -f S16_LE -c1 -r16000 -d2 /tmp/t.wav
aplay -D default /tmp/t.wav
```
