# Echolink S1 PLUS NEW Hacking Project
This is a **hardware hacking** project about the *_echolink s1+ NEW_* set-top box.  
The ultimate goal is **to create a minimal linux distro / custom eCos app** for the device

## Progress
   - Identify Specs (95% - See 'Device Specifications')
   - Get the firmware (100% - See 'Firmware Analysis')
   - Analyze & Reverse enginneer the firmware (40% - See 'Firmware Analysis')
   - Get Serial connection (100% - See 'RS-232 Shell & Logs')

## Device Specifications
*Sources: [here](https://www.echolink.ma/produit/echolink-s1-plus-pro/), and me*  
   - RAM: 2GB DDR3 nanya Nt5CB128M16FP-DI SDRAM ([datasheet](https://raw.githubusercontent.com/FrenchPythonLover/echolink_hacking/refs/heads/main/datasheets/NT5CB256M8FN.PDF))
   - SoC / CPU ??: Sunplus 1507A little endian MIPS (**Please help me find a datasheet !**)
   - Storage: 16MB W25Q128FV Winbond Flash Storage -([datasheet](https://raw.githubusercontent.com/FrenchPythonLover/echolink_hacking/refs/heads/main/datasheets/W25Q128FV.PDF))
   - Ethernet: PSF-16211 EEE802.3 and ANSI X3.263 Compliant Ethernet Chip ([datasheet](https://raw.githubusercontent.com/FrenchPythonLover/echolink_hacking/refs/heads/main/datasheets/PM4-11BP.pdf))
   - Wireless: Seen mtk wifi chip in firmware but couldn't find it on PCB, 2G Modem with SIM reader on the side
   - USB: 2 USB2(?) Ports
   - Serial: RS-232 (See RS-232 Shell & Logs)
   - Video OUT: HDMI, Composite
   - Front panel: LCD, IR, 7 BUTTONS, connected by *3V3,CLK,DATA,STB,GND* Pins
   - Power: 2 12V Power cables.

## Firmware Analysis
The firmware file is named `20170213_ECHOLINK-S1+NEW_1506A_1100_M10_N0_UI0_V`, and can be found [here](https://up.sansat.net/MORESAT/Echolink-Series/Echolink-S1+NEW/20170213_ECHOLINK-S1+NEW_1506A_1100_M10_N0_UI0_V.bin).  
The `binwalk` firmware analyzer tool output is found [here](https://raw.githubusercontent.com/FrenchPythonLover/echolink_hacking/refs/heads/main/firmware/binwalkfw.txt).  
By this line:  
`512 0x200 | eCos kernel exception handler, MIPS little endian`  
We can identify the **OS, the architecture and the endianess** of the device (**eCos RTOS, MIPS 32 Little Endian**).   
These lines
:  
`153600                             0x25800                            LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, compressed size: 213661 bytes, uncompressed size: 558448 bytes`  
`
528384                             0x81000                            LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, compressed size: 512065 bytes, uncompressed size: 1302488 bytes`  
`
1044480                            0xFF000                            LZMA compressed data, properties: 0x5D, dictionary size: 16777216 bytes, compressed size: 6696374 bytes, uncompressed size: 42320448 bytes`  
Tells me that the OS is divided into three LZMA files..
*Im currently porting all of my findings from my project directory to github, so it takes time...*

## RS-232 Shell & Logs
There is a nicely fit connector labeled 'RS-232' on the outside, but i preferred to use the tx & rx pins on the PCB.  
### Serial Specs
After a lot of trial and error, i found out that the rs-232 pins are **inverted**, so i quickly found out the specs of this: **115200 8N1 Inverted**
### Boot logs
The boot logs are available through [here](https://raw.githubusercontent.com/FrenchPythonLover/echolink_hacking/refs/heads/main/firmware/serial/bootlog.txt), here's all the steps:

1. **Jump to second boot** — `19:01:20.239`  
   Boot ROM decides to hand off to the "second boot" stage.

2. **Boot from SPI (bootloader banner)** — `19:01:20.239–19:01:20.246`  
   SPI boot firmware/version printed (`Boot form SPI v1.08`), SPI chip-select / flow initialized.

3. **DDR initialization start** — `19:01:20.407`  
   DDR controller/firmware reports version (`DDR-V6.3.01.02`), package/SDRAM setup begins (`Packeg-01`, `SDRAM_Set`).

4. **DRAM training / timing sweep** — `19:01:20.407–19:01:20.461`  
   Multiple MCPP/CK offsets are tested (CK=00, CK+4=04, CK+4=08, CK+4=0C).  
   For each test the firmware logs results: GPRD, WLx, WDQx, QSx, Gx, PHAx, RSLx and a pass/fail marker (`789-PASS`, `PHA-non-equ2`, `PHA-X`, `E-L-I`).  
   This is the memory PHY calibration and timing tuning phase.

5. **Trimming / final DRAM adjustments** — `19:01:20.461–19:01:20.466`  
   `Trim-O` printed and stamps/versions logged (`STAMP`, `VER`, `BIT`).

6. **Copy firmware from SPI to RAM** — `19:01:20.466–19:01:20.503`  
   Bootloader copies the firmware image from SPI flash into DRAM (`copy fw to RAM`), then signals success (`GO!!`).

7. **Jump to first-stage firmware (1st boot / firmware entry)** — `19:01:20.507`  
   Execution transfers into the in-RAM firmware: zero BSS, initialize interrupt controller (`IntCtrl`).

8. **Load and start eCos kernel** — `19:01:20.510–19:01:20.622`  
   Boot prints `LD eCos v1.09 1506`, then `Go eCos` and shows kernel features (`MMU`, `FPU`, `IntC`, `Cache`, `Timer`).

9. **eCos runtime/platform initialization** — `19:01:20.622–19:01:20.664`  
   Cache regions, monitor, zero BSS again, variable initialization, platform init, read IOP (IO parameters) data, run constructors (`Ctor Init`) — a list of init function addresses is printed.

10. **Cygwin/cyg_start (eCos entry point)** — `19:01:20.667`  
    `cyg_start` called to start eCos threads/services.

11. **eCos relaunch / second eCos banner** — `19:01:24.928–19:01:25.072`  
    A later eCos banner shows `eCos 2.071T` and repeats CPU/MMU/FPU/Cache/Timer initialization and platform init (likely a later kernel/service restart or a second CPU core init sequence), followed by `cyg_start`.

12. **Application/module loading & UI stage** — `19:01:34.649–19:01:46.556`  
    Modules are probed/loaded (`GetModuleTypeFrHeader==LOGO_0`), and final runtime action opens the install menu (`open install menu`).

---