<div align="center">

# The Channel F Cartridge Image Format (.chf)
 
![badge](https://badgen.net/badge/version/v1.0/orange?style=flat-square)

A file format to store games made for the Fairchild Channel F. Based on the [Cartridge Image](http://unusedino.de/ec64/technical/formats/crt.html) format from the CCS64 emulator.
  
<div align = "center">
  <img width="563" alt="image" src="https://user-images.githubusercontent.com/44975876/168489152-900b3204-42b1-4325-b72c-ad9df8860520.png">
 
  *Art by Rossil Fuel*
</div>
  
[File Overview](#file-overview) •
[File Header](#file-header) •
[Packet Overview](#packet-overview) •
[Packet Header](#packet-header) •
[Credits](#credits)
  
</div>

# File Overview

The `.chf` file format was created to solve three major issues:
- Confusion caused by multiple undocumented `.bin` formats. Detecting different banking schemes can be difficult or sometimes even impossible
- Simplify feature detection for emulators/flashcarts (such as the expected presence of SRAM). Currently SRAM is provided if a Videocart attempts to write to memory. But this method can cause issues if a bug writes to the ROM area
- `.bin` files contain no information on the games themselves

To solve these issues, the `.chf` file format needs to provide the necessary information, while also being future proof in the event of new hardware/banking schemes. The [Cartridge Image](http://unusedino.de/ec64/technical/formats/crt.html) format from the CCS64 emulator was used as inspiration, as it had similar goals. Note this file format is little-endian to simplify use by arduino/x86/RP2040 architectures. Some other advantages to a custom format are that it can be associated with MESS/MAME/etc to open with no fuss (since the `.chf` extension is unique), and that the title is always available regardless of the system filename (since it's stored in the file metadata like tags in various media files).

# File Header

The file header contains basic information on the Videocart (name/hardware), as well as file format information that's necessary for interpretting the data, while allowing for future expansion. It's followed by a list of packets, described in the next section. Extra data can be included by extending the *file header length* beyond the Videocart name, as this will always be ignored.

<div align = "center">
  <img width="512" src="https://user-images.githubusercontent.com/44975876/187844771-8c3c6df8-f2cd-4ed9-ba71-65c6b6a4e37a.png">

  *A `.chf` file storing a ROM+RAM Videocart called "3D MONSTER MAZES 2"*
</div>

| Name                    | Address | Length (bytes) | Comments                                                     |
| ----------------------- | ------- | -------------- | ------------------------------------------------------------ |
| Cartridge signature     | \$0000  | 16             | `CHANNEL F`. Used to detect a valid file. Padded with spaces |
| File header length      | \$0010  | 4              | `$xxxxxxx0` as the header is zero-padded to be 16-byte aligned |
| File format version       | \$0014  | 2              | The version of the file format being used. Typically v1.00. Implementations should refuse to run games with major version numbers unknown by them. |
| Cartridge hardware type | \$0016  | 2              | [Described below](#designated-hardware-types)                |
| Reserved for future use | \$0018  | 8              |                                                              |
| Videocart name length   | \$0020  | 1              |                                                              |
| Videocart name          | \$0021  | 0 - 255        | UTF-8 encoded                                                |

### Designated Hardware types

| Name                    | Hardware Type Value | [Memory-mapped](#designated-chip-types) | [Port-mapped](#supported-io-port-devices) | Comments |
| ----------------------- | ------------------- | ------------- | ----------- | -------- |
| Videocart               | \$0000              | ROM           |             | Used by all official Videocarts except 10, 18, and 20 (SABA) |
| Videocart 10 (Maze)     | \$0001              | ROM           | 2102 SRAM   | Uses ports $24/$25 |
| Videocart 18 (Hangman)  | \$0002              | ROM           | 2102 SRAM   | Uses ports $20/$21 |
| ROM+RAM (With 3853)     | \$0003              | ROM, RAM      | 3853 SMI    | Used by some homebrew videocarts |
| SABA Videoplay 20       | \$0004              | ROM, RAM, LED | 3853 SMI    |          |
| Flashcart               | \$0005              | All           | All         | Used by the Videocart-π to provide any unofficial memory/port types |

### Supported I/O Port Devices

| Name                                  | Port(s)   | Comments                                                  |
| ------------------------------------- | --------- | --------------------------------------------------------- |
| Interrupt control                     | $6      | From the MK 3870/3871 IC                                  |
| Binary Timer                          | $7      | From the MK 3870/3871 IC                                  |
| 2102 SRAM                             | $20 / $21 / $24 / $25 | 1-bit RAM. Ports described [here](http://seanriddle.com/mazepat.asm) |
| Programmable interrupt vector address | $C / $D | From the 3853 SMI IC                                      |
| Programmable timer                    | $E      | From the 3853 SMI IC                                      |
| Interrupt control                     | $F      | From the 3853 SMI IC                                      |



# Packet Overview

Packets serve as hardware descriptors, providing information on what hardware the game expects to be present. Some packets are special, as they provides the data that is (or would be) present in the on-board ROM / NVRAM. Packets are always zero-padded to be 16-byte aligned.

# Packet Header

The packet header contains basic information on how the expected hardware is accessed.

<div align = "center">
  <img width="512" src="https://user-images.githubusercontent.com/44975876/172023103-309797bd-a0e2-4caa-ae79-497148057ab7.png">

  *A ROM chip packet (with data) at \$2000 - \$2800*
</div>

### Packets

| Name                    | Length (bytes) | Memory-mapped Description                                    |
| ----------------------- | -------------- | ------------------------------------------------------------ |
| Signature               | 4              | `CHIP`. Used to detect a valid file.                         |
| Total packet length     | 4              | Header + Data(only some chip types) + Padding                |
| Chip type               | 2              | [Described below](#designated-chip-types)                                              |
| Bank number             | 2              | Used for banking. Always `$0000` when no banking scheme is used |
| Starting load address   | 2              | Where the memory region starts                               |
| Memory size in bytes    | 2              | The size of the memory region                                |
| Data                    | 0 - 63,487     | Only present for some chip types. Technically supports up to 65,535 bytes but the first 2K of memory (\$0000 - \$07FF) should always be the BIOS, so the largest practical range is \$0800 - \$FFFF |

### Designated Chip Types

| Name          | Chip Type Value | Comments                                                          | Typical Range   | Has data |
| ------------- | --------------- | ----------------------------------------------------------------- | --------------- | :------: |
| ROM           | $0000           | This memory range is Read-only                                    | \$0800 - \$FFFF |    Y     |
| RAM           | $0001           | This memory range is read/write-able                              | \$2800 - \$3000 |          |
| LED           | $0002           | Similar to ROM, but writing to this memory range toggles the LED  | \$3800 - \$4000 |    Y     |
| NVRAM         | $0003           | This memory range is non-volatile and read/write-able             |                 |    Y     |

# Credits

Developed by e5frog (from AtariAge) and 3DMAZE (from AtariAge)
