# Build and run ELF application

ESP32 toolchain distributed in [crosstool-NG project releases](https://github.com/espressif/crosstool-NG/releases) is able to build ELF executables without using IDF dependencies (e.g.: bootloader and other components).

All the minimal but essential code for chip initialization and syscall functions are contained in `libgloss` which is a part of `xtensa-esp32-elf` toolchain.
It means that generated ELF file can be executed on a bare-metal ESP32 chip or on QEMU simulation.

# Spec files

To link an application with `libgloss` library use spec files which are contained in xtensa ESP32 toolchain.

Specs files to enable semihosting could be used.

* Semihosting - mechanism that enables code running on an ESP32 to communicate with and use the I/O of the host computer. For example, it can be used for writing to host's terminal or host's file during a run on ESP32 or QEMU.

> NOTE: when semihosting operation is executed the target processor is simply stopped until data is being transferred to host.
>
> This could slow down the performance of an executed application.


## QEMU specs

* `sim.elf.specs` - build ELF for running on QEMU
* `sys.qemu.specs` - use I/O operations using semihosting in QEMU

## OpenOCD specs

* `board.elf.specs` - build ELF for running on bare-metal ESP32
* `sys.openocd.specs` - use I/O operations using semihosting in openOCD

# Example

main.c listing:

```c
#include <stdio.h>
int main() {
   printf("Hello, World!\n");
}
```

# QEMU

## Build

```
xtensa-esp32-elf-gcc main.c -specs=sim.elf.specs [-specs=sys.qemu.specs]
```

## Run

```
qemu-system-xtensa -nographic -monitor null -cpu esp32 -M esp32 -kernel ./a.out [--semihosting]
```

# Bare-metal ESP32

## Build

```
xtensa-esp32-elf-gcc main.c -specs=board.elf.specs [-specs=sys.openocd.specs]
```

## Run

#### Step 1:

[Connect ESP32 to host](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-guides/jtag-debugging/configure-ft2232h-jtag.html#configure-devkit-name-jtag-interface) via JTAG interface.

#### Step 2:

Run OpenOCD session.

```
openocd -c 'set ESP_RTOS none' -f board/esp32-wrover-kit-3.3v.cfg
```

#### Step 3:

Load ELF file to the RAM through JTAG and run the application.

```
  xtensa-esp32-elf-gdb ./a.out
  (gdb) target remote :3333
  (gdb) mon reset halt
  (gdb) load
  (gdb) continue
```
