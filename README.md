# ESP32-C3 ROM Boot process

## Introduction

The Espressif ESP32 family of system-on-chip (SOC) microcontrollers are highly capable, easy to use devices for projects requiring connectivity. I often reach for an ESP32 in professional and personal projects because of their flexibility, capacity and ease of use. Particularly, for rapidly evolving prototype work, I can usually rely on one of the Espressif chips to meet the initial brief, and also future requirements of the project without stressing whether the choice of SOC will become a technical constraint. However, one of the often cited weaknesses of the Espressif chips is the fact that not all of the ESP-IDF is open-source, including the radio drivers (provided as a pre-compiled blob), the first-stage ROM bootloader (factory configured) and several other ROM functions.

These are limitations I am fully aware of and the potential downsides of this opacity have never stopped me from using the devices where they were the appropriate selection for the job. However, during a recent 'bare metal' [project](https://github.com/TristanWebber/garage_monitor/tree/main/esp_bare_c) using the Espressif ESP32-C3, curiosity got the better of me. Using the [Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32-c3_technical_reference_manual_en.pdf) (TRM), any type of project that doesn't require the radio peripherals can be built using 100% user code - with one exception - the ROM bootloader will always run before user code and is immutable. The [ESP-IDF Programming Guide](https://docs.espressif.com/projects/esp-idf/en/v5.2.4/esp32c3/api-guides/startup.html) provides a good summary of what the bootloader is doing, however, one burning question remained - if it were physically possible, could a user replicate the functionality of the ROM bootloader using publicly available documents alone? Or is there something happening in the boot process that is undocumented?

## Objectives

The objective of this investigation was to decompile the ROM bootloader and discover:
- The general program flow
- The peripherals and registers in use during the boot process

## Method

The following method was used:

1. Obtain an image of the ESP32-C3 rev3 ROM
2. Disassemble the text sections using gcc readelf from a riscv32 cross compiler
3. Disassemble the data sections using gcc objdump from a riscv32 cross compiler
4. Reverse engineer using Ghidra
5. Follow program flow from entry point (0x4000_0000) to the start of user code
6. Cross reference registers used in the ROM bootloader to those documented in the TRM
7. Ignore functions in the ROM that are not called as part of the bootload process
8. Document the program flow in pseudocode

The documented program flow has been expressed in pseudocode expressing the general nature of operations. High level operations have typically been described with a label. Some labels have been taken from the image and some have been given names by me. The pseudocode makes most labels appear like a function call however this is often done out of convenience to aid reading and interpretation. In reality, many of the behaviours I have grouped under labels actually appear in the image as an inlined group of instructions. In many cases, I have omitted code like error checking and runtime asserts to enhance readability.

I have not included a copy of the image or the disassembly of the image to avoid the presentation of almost inevitable errors of interpretation at the granular level. This project is intended to share high level details of a fun side project, and is not designed to be a faithful instruction by instruction reversing of the ROM image to source code.

## Results

Using readily available tools, it was possible to reverse the ESP32-C3 ROM image and traverse all paths from the program entry point to the start of user code execution. Findings are presented in two sections - the program flow in pseudocode, and a list of undocumented registers accessed in the boot process.

### Program flow

The key findings from the investigation of program flow are as follows:

- The bootloader entry point is address 0x4000_0000
- An `init` function performs basic hardware config, including:
    - Loading an interrupt vector table that jumps to a default interrupt handler for all 32 CPU interrupts
    - Sets the stack pointer
    - Unpacks the data section from ROM to DRAM
    - Clears .bss data
    - Calls the bootloader `main` function
- The bootloader `main` function ensures user code is read from RAM, serial or SPI Flash:
    - If waking from deep sleep the bootloader checks if a valid user code exists in IRAM and starts execution if present. If not, normal power-on program flow is continued.
    - If strapping pins are configured correctly, attempt to load a user code image over serial to flash, then do a software restart
    - If the previous two states are not present, attempt to load user code from flash via SPI, the start execution of user code

```c
// _start label is at 0x4000_0000 and simply jumps to _init
_start() {
    _init();
}

// Perform basic hardware configuration - set interrupt vector and stack
// pointer, copy ROM data to RAM, clear .bss sections
_init() {
    // Disable machine mode and user mode interrupts by writing to CSR Registers
    global_machine_interrupt_disable();
    global_user_interrupt_disable();       // (Undocumented)

    // Disable CPU interrupts by writing to an Interrupt Matrix register
    interrupt_core0_cpu_int_disable();

    // Set Bit 0 of two custom CSR registers (Undocumented)
    custom_csr_register_set_0x800(1);
    custom_csr_register_set_0x801(1);

    // Set interrupt vector table. All 32 rows use identical handler
    // j _interrupt_handler
    load_vector_table_to_mtvec_csr(_vector_table);

    // Set CPU interrupt threshold to minimum (Configure CPU to respond to all 
    // interrupts with priority greater than 0)
    interrupt_core0_cpu_int_thresh(1);

    // Enable machine mode and user mode interrupts by writing to CSR registers
    global_machine_interrupt_enable(); // Bit 0 (reserved) is also set
    global_user_interrupt_enable();

    // Set stack pointer to 0x3FCD_E710 (0x2000 available)
    set_stack_pointer_rom_bootloader();

    // Copy ROM data to DRAM (0x3FCD_E710 to 0x3FCD_FFFF)
    unpackloop();

    // Clear .bss for ROM Bootloader
    clearloop();

    main();
}

// Bootloader reads user code image from RAM, Serial or flash
main() {
    // Get reset reason by reading from a Low-Power Management register
    rtc_reset_reason = rtc_get_reset_reason();

    // Physical configuration to prepare for boot
    // 1. UART debug is enabled
    // 2. CPU frequency is set
    // 3. Static variables containing the state of strapping pins are set
    // 4. Debug assistant is started
    apb_freq = boot_prepare(rtc_reset_reason);

    // At this point, the following configurations have occurred:
    // 1. CPU interrupts are enabled
    // 2. There is a stack and ROM data is available
    // 3. UART debug is enabled
    // 4. CPU frequency is set
    // 5. Static variables containing the state of strapping pins are set
    // 6. Debug assistant is started

    // Bootloader action determined based on rtc_reset_reason and strapping pins
    // Invalid values in register trigger software reset
    if(is_invalid_reset(rtc_reset_reason) {
        ets_printf("invalid_reset.");
        software_reset();
    }

    // For reset reasons where RAM was not powered off, check if code can be
    // executed from RAM
    rtc_reset_reason_masked = 1 << (rtc_reset_reason & 0x1F)
    if (rtc_reset_reason_masked & 0xFFBF8A == 0) {
        bypass_bootloader();
    }

    // When strapping pin 9 is low, download mode should be entered
    if (!strapping_pin_9_set) {
        download_mode();
    } else {
        // Download mode not set - read image from flash
        // 1. configure spi
        // 2. flash boot attach
        // 3. run flash bootloader
        flash_bootloader();
    }

    // If the function pointer to user code is valid, run user code
    if (user_code != 0) {
        *user_code();
    }

    ets_printf("user code done");
    return 0;
}

// Check for valid user code in RAM
bypass_bootloader() {
    if (reset_reason_is_deep_sleep) {
        software_reset();
    }

    // When strapping pin 9 is high, boot mode is not entered
    if (strapping_pin_9_set) {
        // If there's a valid function pointer (nonzero value with valid CRC) to
        // user code in an RTC scratch register, run the user code
        user_code = rtc_boot_control(rtc_reset_reason);
        if (user_code != 0) {
            *user_code();
            user_code = 0;
        }
    }
}

// Check efuse (is secure mode enabled?) and strapping pins for download mode 
// (UART, USB SPI) 
download_mode() {
    if (ets_efuse_download_modes_disabled()) {
        ets_printf("Download boot modes disabled.");
        software_reset();
    }

    uartAttach();

    // Download in secure boot mode if enabled and valid
    if (ets_efuse_security_download_modes_enabled()) {
        if (secure_boot_mode_invalid()) {
            ets_printf("Invalid mode for secure boot download.");
            software_reset();
        }

        download_secure_boot();
        software_reset();
    }

    download_mode = (_GPIO_STRAP_REG & 0xf);

    // Configure UART for serial download if appropriate strapping pins are set
    if (download_mode == DOWNLOAD_MODE_UART) {
        ets_printf("wait uart download (secure mode).");
        download_uart_init();
    }

    // Configure USB for serial download if appropriate strapping pins are set
    if (download_mode == DOWNLOAD_MODE_USB) {
        ets_printf("wait usb download.");
        download_usb_init();
    }

    // Configure SPI for serial download if appropriate strapping pins are set
    if (download_mode == DOWNLOAD_MODE_SPI) {
        ets_printf("wait spi download.");
        download_spi_init();
    }

    // Download image over configured peripheral
    do_download();
    ets_printf("waiting for download.");

    // If no user code pointer after downloads, busyloop. The watchdog timer
    // will trigger a reset if this code is reached.
    if (user_code == 0) {
        ets_printf("ets_main.c 688");
        while(1);
    }
}
```

### Undocumented registers

In the boot process, some undocumented register operations were observed. The intended function of these can mostly be inferred from the context of the operation.

#### CSRs

|CSR   | function | called_from |
|------|----------|-------------|
|0x000 | ucause   | _init       |
|0x800 | unknown  | _init       |
|0x801 | unknown  | _init       |

Three read/writes to undocumented CSRs were observed in the init code during the configuration of interrupts. The 0x000 CSR is not documented in the ESP32 TRM, however it appears in the RISC-V Specification as the `ucause` (User Cause) register. This would make sense if there were user-level interrupts in the implementation. It appears that this is not used at present because all code seems to be executed with the machine privilege level. However, it appears there may be some [future intent](https://docs.espressif.com/projects/esp-privilege-separation/en/latest/esp32c3/index.html) by Espressif to implement a seperate user code in the IDF.

The CSRs 0x800 and 0x801 are reserved as custom CSRs in the RISC-V Specification. There is no reference to these registers in the ESP32-C3 TRM, however there context of usage in the ROM bootloader suggests they are related to the interrupt system. In both cases, the only operation on those registers is to set bit 0.

#### Module/Peripheral

Two entirely undocumented base addresses were identified.

1. **Base address 0x600C_4000 (CACHE)**

The base address of this register is marked as Reserved in the TRM. The context of labels referencing this area of code suggest this is related to the instruction and data caches.

2. **Base address 0x600C_5000 (MMU)**

The base address of this register is marked as Reserved in the TRM. The context of labels referencing this area of code suggest this is related to the Memory Management Unit (MMU).

#### Registers

A small number of undocumented registers, or writes to a part of a register marked as 'reserved', were found within the ROM bootloader.

1. **Base address 0x6000_8000 (LOW_POWER)**

A series of offsets were accessed to read efuse values:

|offset | function                   |
|-------|----------------------------|
|0x0830 | EFUSE_RD_REPEAT_DATA0_REG  |
|0x0838 | EFUSE_RD_REPEAT_DATA2_REG  |
|0x083C | EFUSE_RD_REPEAT_DATA3_REG  |
|0x0844 | EFUSE_RD_MAC_SPI_SYS_0_REG |
|0x0848 | EFUSE_RD_MAC_SPI_SYS_1_REG |

The bits and registers (with offset minus 0x0800) were consistent with the TRM documentation for the eFuse controller documented under base address 0x6001_A000. It appears that these registers are duplicated in the LOW_POWER address range. Eg, 0x6000_8830 appears to be the same as 0x6001_A030.

2. **Base address 0x6000_3000 (SPI0)**

|offset | function                  |
|-------|---------------------------|
|0x0054 | Wait_SPI_Idle             |

The Wait_SPI_Idle function reads from an undocumented register under the SPI0 base address. This register appears to contain busy flags in bits 4, 5 and 6.

## Conclusion

The program flow for the ESP32-C3 ROM bootloader as determined by reverse engineering was found to follow the process documented by Espressif. There were several registers used in the ROM bootloader that are not documented in the TRM. The functionality of most of these registers was able to be inferred from the context of the code being executed in that region and for the most part, the lack of knowledge of those registers would not prevent a sufficiently motivated developer from replicating the basic functionality of the ROM bootloader, were it hypothetically possible to implement truly 'bare metal' code on the ESP32-C3. However, the lack of documentation on the Cache, MMU and two custom CSRs means that in all likelihood, a user would be unable to match the performance of the factory ROM bootloader, since they would not be able to correctly configure the instruction and data caches. Given that the instruction cache and data cache are being used by the bootloader, it would be wise to avoid the use of the internal SRAM0 instruction address range (0x4037_C000 to 0x4037_FFFF) when linking 'bare metal' applications. The ROM bootloader appeared to perform minimal configuration of peripherals, and most peripherals were subsequently shut down after being used unless critical to the operation of the SOC beyond the completion of the ROM bootloader.
