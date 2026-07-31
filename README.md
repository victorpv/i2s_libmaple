# STM32F1 I2S LIBRARY WITH MP3 Player

# This DOC is an AI generated draft and may contain errors. 

![License](https://img.shields.io/badge/license-MIT-blue.svg)
![Platform](https://img.shields.io/badge/platform-STM32F1-blue.svg)
![Core](https://img.shields.io/badge/core-libmaple%20%2F%20stm32duino-green.svg)

A high-performance i2S and MP3 player implementation for STM32F1 microcontrollers using the **Adafruit_MP3** decoder library and **I2S** output for the MAX98357A I2S Class D amplifier. Features DMA-driven audio playback, SD card support via SDIO, and real-time sample rate adaptation.

---

## 📋 Table of Contents

- [Project Overview](#-project-overview)
- [Hardware Requirements](#-hardware-requirements)
- [Wiring Diagram](#-wiring-diagram)
- [Software Dependencies](#-software-dependencies)
- [Installation & Setup](#-installation--setup)
  - [Arduino IDE](#arduino-ide)
  - [Eclipse CDT](#eclipse-cdt)
- [Quick Start Guide](#-quick-start-guide)
- [Usage Examples](#-usage-examples)
  - [SD Card + I2S Playback](#sd-card--i2s-playback)
  - [I2S Output Configuration](#i2s-output-configuration)
- [API Quick Reference](#-api-quick-reference)
- [Troubleshooting](#-troubleshooting)
- [Customization Guide](#-customization-guide)
- [Pin Configuration](#-pin-configuration)
- [Performance Notes](#-performance-notes)
- [Related Resources](#-related-resources)
- [License](#-license)

---

## 🎯 Project Overview

This project implements a standalone MP3 player on STM32F1 series microcontrollers (STM32F103, STM32F105, STM32F107) using:

- **Adafruit_MP3** — A lightweight, integer-only MP3 decoder based on the public domain `minimp3`/`mp3dec` library
- **I2S Class Driver** — DMA-driven I2S peripheral driver for STM32F1 SPI2/SPI3 peripherals
- **SDIO SD Card** — High-speed SD card access via STM32F1 SDIO peripheral

### Key Features

| Feature | Description |
|---------|-------------|
| **Real-time MP3 Decoding** | Integer-only decoder, no floating-point required |
| **DMA-Driven I2S** | Zero-CPU overhead audio output via double-buffered DMA |
| **Sample Rate Adaptation** | Automatic I2S clock reconfiguration on MP3 sample rate change |
| **SDIO Support** | High-speed SD card access (up to 24 MHz) |
| **Callback Architecture** | Non-blocking design with transmit/receive callbacks |
| **Multiple I2S Modes** | Philips, Left-Justified, Right-Justified, PCM modes supported |

### Supported Boards

- **STM32F103C8T6** (Blue Pill) — SPI2 I2S
- **STM32F103VET6** — SPI2/SPI3 I2S
- **STM32F105/107** — SPI2/SPI3 I2S with enhanced connectivity
- Any STM32F1 with libmaple/stm32duino core and SPI2/SPI3

---

## 🔧 Hardware Requirements

### Bill of Materials

| Component | Quantity | Notes |
|-----------|----------|-------|
| STM32F103C8T6 (Blue Pill) or compatible | 1 | Minimum 64KB Flash, 20KB RAM |
| MAX98357A I2S Class D Amplifier Breakout | 1 | 3.2W mono, 2.5V–5.5V supply |
| Micro SD Card Module (SDIO compatible) | 1 | Level-shifted for 3.3V logic |
| Micro SD Card (FAT32, ≤32GB) | 1 | Class 10 recommended |
| 4Ω–8Ω Speaker | 1 | 3W+ recommended |
| 100µF Electrolytic Capacitor | 1 | Power supply decoupling |
| 0.1µF Ceramic Capacitors | 2–3 | VDD/GND decoupling near ICs |
| Breadboard / Protoboard | 1 | For prototyping |
| Jumper Wires | Assorted | Male-to-male, male-to-female |

### Power Requirements

- **STM32F1**: 3.3V @ ~50mA (MCU) + I2S peripheral
- **MAX98357A**: 2.5V–5.5V @ up to 1.5A (at 3.2W into 4Ω)
- **SD Card**: 3.3V @ up to 100mA (during write)
- **Total**: ~500mA @ 5V typical, 1A peak recommended

---

## 🔌 Wiring Diagram

### STM32F1 + MAX98357A + SD Card (SDIO)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        STM32F103C8T6 (Blue Pill)                           │
│                                                                             │
│   ┌─────────────┐                    ┌─────────────────────────────────┐   │
│   │   SPI2/I2S  │                    │           SDIO                  │   │
│   │             │                    │                                 │   │
│   │  PB12 ──┬──►│── LRCK (LRC)       │  PC8  ──┬──► D0 (DAT0)          │   │
│   │  PB13 ──┼──►│── BCLK (BCK)       │  PC9  ──┬──► D1 (DAT1)          │   │
│   │  PB15 ──┼──►│── DIN (DATA)       │  PC10 ──┬──► D2 (DAT2)          │   │
│   │         │   │                    │  PC11 ──┬──► D3 (DAT3)          │   │
│   │         │   │                    │  PC12 ──┬──► CLK                │   │
│   │         │   │                    │  PD2  ──┬──► CMD                │   │
│   └─────────│───┘                    └─────────│────────────────────────┘   │
│             │                                  │                            │
│             ▼                                  ▼                            │
│   ┌─────────────────────┐          ┌─────────────────────┐                │
│   │    MAX98357A        │          │    Micro SD Card    │                │
│   │                     │          │      Module         │                │
│   │  LRC  ◄─────────────┤          │                     │                │
│   │  BCK  ◄─────────────┤          │  DAT0 ◄─────────────┤                │
│   │  DIN  ◄─────────────┤          │  DAT1 ◄─────────────┤ (4-bit mode)   │
│   │                     │          │  DAT2 ◄─────────────┤                │
│   │  GAIN ◄── GND (15dB)│          │  DAT3 ◄─────────────┤ (CD/DAT3)      │
│   │  SD   ◄── 3.3V (EN) │          │  CLK  ◄─────────────┤                │
│   │  GND  ◄── GND       │          │  CMD  ◄─────────────┤                │
│   │  VDD  ◄── 5V/3.3V   │          │  VCC  ◄── 3.3V      │                │
│   │                     │          │  GND  ◄── GND       │                │
│   │  OUT+ ──► Speaker + │          └─────────────────────┘                │
│   │  OUT- ──► Speaker - │                                                │
│   └─────────────────────┘                                                │
│                                                                             │
│   Power:                                                                   │
│   ┌─────────────┐                                                          │
│   │  5V ────────┼────► MAX98357A VDD, STM32 5V pin (via regulator)       │
│   │  3.3V ──────┼────► MAX98357A SD, SD Card VCC, STM32 3.3V             │
│   │  GND ───────┼────► All GND pins (common ground)                       │
│   │             │                                                          │
│   │  100µF ─────┤ (across 5V/GND near MAX98357A)                          │
│   └─────────────┘                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pin Mapping Summary

| Function | STM32F1 Pin | MAX98357A Pin | SD Card Pin | Notes |
|----------|-------------|---------------|-------------|-------|
| I2S_LRCK (WS) | **PB12** (SPI2_NSS/I2S2_WS) | LRC | — | Left/Right Clock |
| I2S_BCLK (CK) | **PB13** (SPI2_SCK/I2S2_CK) | BCLK | — | Bit Clock |
| I2S_SD (SD) | **PB15** (SPI2_MOSI/I2S2_SD) | DIN | — | Serial Data Out |
| SDIO_D0 | **PC8** | — | DAT0 | Data Line 0 |
| SDIO_D1 | **PC9** | — | DAT1 | Data Line 1 |
| SDIO_D2 | **PC10** | — | DAT2 | Data Line 2 |
| SDIO_D3 | **PC11** | — | DAT3 | Data Line 3 (Card Detect) |
| SDIO_CK | **PC12** | — | CLK | Clock (up to 24 MHz) |
| SDIO_CMD | **PD2** | — | CMD | Command Line |

> **Note**: PB14 (SPI2_MISO) is not used in I2S master transmit-only I2S master transmit mode.

### MAX98357A Gain Selection

| GAIN Pin | Gain | Typical Use |
|----------|------|-------------|
| GND | 15 dB | Line-level input |
| 3.3V | 12 dB | Standard |
| SDA (47k to GND) | 9 dB | Higher voltage sources |
| SCL (47k to GND) | 6 dB | Maximum headroom |
| Floating | 3 dB | Minimum gain |

---

## 📦 Software Dependencies

### Required Libraries

| Library | Version | Source | Purpose |
|---------|---------|--------|---------|
| **stm32duino / libmaple core** | Latest | [GitHub](https://github.com/stm32duino/Arduino_Core_STM32) | STM32F1 Arduino core |
| **Adafruit_MP3** | Included | `./Adafruit_MP3` | Integer MP3 decoder |
| **I2S Driver** | Included | `./I2S.h` / `./I2S.cpp` | STM32F1 I2S peripheral driver |
| **SdioF1 / SdFat** | 2.x | [GitHub](https://github.com/greiman/SdFat) | SDIO SD card driver |

### Toolchain Requirements

| Tool | Minimum Version | Notes |
|------|-----------------|-------|
| **ARM GCC** | 10.x+ | `arm-none-eabi-gcc` |
| **OpenOCD** | 0.11+ | For SWD debugging/flash |
| **dfu-util** | 0.11+ | For USB DFU bootloader |
| **make** | 4.x+ | Build automation |

---

## ⚙️ Installation & Setup

### Arduino IDE

#### 1. Install STM32duino Core

1. Open **File → Preferences**
2. Add to **Additional Boards Manager URLs**:
   ```
   https://github.com/stm32duino/BoardManagerFiles/raw/main/STM32/package_stm_index.json
   ```
3. Open **Tools → Board → Boards Manager**
4. Search "STM32" and install **STM32 MCU based boards** (by STMicroelectronics)

#### 2. Install Library Dependencies

**Option A: Manual Installation (Recommended)**

```bash
# Clone this repository
git clone https://github.com/your-repo/stm32f1-mp3-player.git
cd stm32f1-mp3-player

# Copy libraries to Arduino libraries folder
# Linux/macOS:
cp -r Adafruit_MP3 ~/Arduino/libraries/
cp -r I2S ~/Arduino/libraries/

# Windows:
xcopy Adafruit_MP3 %USERPROFILE%\Documents\Arduino\libraries\ /E /I
xcopy I2S %USERPROFILE%\Documents\Arduino\libraries\ /E /I
```

**Option B: Library Manager**

- Search for "SdFat" and install **SdFat** by Bill Greiman (v2.x)
- Note: Adafruit_MP3 and I2S are project-specific and must be installed manually

#### 3. Configure Board Settings

```
Tools → Board:     "Generic STM32F103C series"
Tools → Variant:   "STM32F103C8 (20k RAM, 64k Flash)"
Tools → Upload:    "STM32CubeProgrammer (SWD)" or "Serial"
Tools → CPU Speed: "72 MHz"
Tools → Optimize:  "Fast" or "Fastest"
```

#### 4. Open and Upload

1. Open `mp3_test.ino` in Arduino IDE
2. Connect STM32F1 via ST-Link (SWD) or USB (bootloader)
3. Click **Upload** (Ctrl+U)

---

### Eclipse CDT (with GNU ARM Eclipse / STM32CubeIDE)

#### 1. Prerequisites

- **STM32CubeIDE** (recommended) or **Eclipse IDE for C/C++** + **GNU ARM Eclipse plugins**
- **STM32CubeMX** for peripheral initialization (optional)

#### 2. Project Setup

```bash
# Import as Makefile project or create new STM32 project
# Using STM32CubeIDE:
File → New → STM32 Project → Select MCU (STM32F103C8Tx) → Next
Project Name: STM32F1_MP3_Player → Finish
```

#### 3. Configure Middleware (CubeMX)

1. Open `.ioc` file
2. **Connectivity → SDIO**: Mode "SD 4-bit Wide bus", DMA enabled
3. **Connectivity → SPI2**: Mode "I2S_Master_TX", Data 16-bit, Audio Clock "I2S_CKIN" disabled
   - I2S Standard: "Philips" / "MSB Justified" / "LSB Justified" / "PCM"
   - Clock Source: "PLLI2S" (configure in RCC)
4. **RCC**: Enable PLLI2S, set PLLI2SN/PLLI2SR for 44.1kHz/48kHz
5. **DMA**: 
   - SPI2_TX: Circular, Half/Full Transfer interrupts enabled
   - SDIO_RX/TX: Normal mode
6. Generate Code

#### 4. Add Project Sources

```bash
# Copy project sources to Eclipse project
cp -r Adafruit_MP3/* <project>/Inc/   # Headers
cp -r Adafruit_MP3/* <project>/Src/   # Sources (if any .c/.cpp)
cp I2S.h I2S.cpp <project>/Inc/ <project>/Src/
cp mp3_test.ino <project>/Src/main.cpp  # Rename and adapt
```

#### 5. Build Configuration

**C/C++ Build → Settings:**

```
Compiler Flags:
  -std=gnu++14 -Os -mcpu=cortex-m3 -mthumb -mfloat-abi=soft
  -DSTM32F103xB -DHSE_VALUE=8000000
  -I"${workspace_loc:/${ProjName}/Inc}"
  -I"${workspace_loc:/${ProjName}/Adafruit_MP3/src}"
  -I"${workspace_loc:/${ProjName}/I2S}"

Linker Flags:
  -T"${workspace_loc:/${ProjName}/STM32F103C8Tx_FLASH.ld}"
  -Wl,--gc-sections -specs=nano.specs -specs=nosys.specs
```

#### 6. Flash & Debug

```
Run → Debug Configurations → GDB OpenOCD Debugging
  - Interface: ST-Link
  - Target: STM32F1
  - Start address: 0x08000000
```

---

## 🚀 Quick Start Guide

### Step 1: Hardware Assembly

1. **Wire the circuit** per the [Wiring Diagram](#-wiring-diagram)
2. **Insert SD card** formatted as FAT32 (32GB max)
3. **Copy MP3 files** to SD card root directory
4. **Connect power** (5V to MAX98357A VDD, 3.3V to logic)
5. **Connect ST-Link** to SWD pins (SWCLK/PA14, SWDIO/PA13, GND)

### Step 2: Prepare Test File

```bash
# Copy a test MP3 to SD card root
cp test.mp3 /media/user/SDCARD/
# Or use the example file name expected by code:
# filename = "test.mp3"
```

### Step 3: Configure Code (if needed)

Edit `mp3_test.ino`:
```cpp
// Change filename if different
const char *filename = "your_song.mp3";

// Adjust volume (0-1023 for 10-bit DAC, but MAX98357A is digital)
#define VOLUME_MAX 1023  // Not used in I2S path, kept for compatibility
```

### Step 4: Build & Flash

**Arduino IDE:**
```
Ctrl+U (Upload)
```

**Eclipse/STM32CubeIDE:**
```
Ctrl+B (Build) → F11 (Debug/Flash)
```

### Step 5: Verify Operation

1. Open Serial Monitor (9600 baud)
2. Expected output:
   ```
   Native MP3 decoding!
   Initializing SD card...
   card initialized.
   Playing!
   ```
3. Audio should play from speaker
4. LED (PC13 on Blue Pill) may blink during playback

---

## 💡 Usage Examples

### SD Card + I2S Playback (Main Example)

**File: `mp3_test.ino`**

```cpp
#include <SdioF1.h>           // SDIO SD card driver
#include "./I2S.h"            // I2S driver for STM32F1
I2SClass I2S(SPI2);          // Use SPI2 for I2S2

#include "./Adafruit_MP3/src/Adafruit_MP3.h"

#define VOLUME_MAX 1023
const char *filename = "test.mp3";

SdFatSdio SD;                // SDIO interface
File dataFile;
Adafruit_MP3 player;

// Double-buffer pointers for DMA
volatile uint16_t buffersize16;
volatile int16_t * bufferptr;
volatile bool getNextFrame;

// I2S DMA Callback - fills half-buffer on each interrupt
void i2sCallback() {
    volatile uint16_t offset;
    switch(I2S.event) {
        case DMA_TRANSFER_COMPLETE:
            offset = buffersize16/2;  // Second half
            break;
        case DMA_TRANSFER_HALF_COMPLETE:
            offset = 0;               // First half
            getNextFrame = true;
            break;
        default:
            return;
    }
    // Decode MP3 directly into I2S DMA buffer
    player.getSamples(&bufferptr[offset], buffersize16/2);
}

// Sample rate change callback (for VBR MP3s)
void rateCallback(uint32_t rate) {
    I2S.setSampleRate(rate);
}

// SD card read callback
int getMoreData(uint8_t *writeHere, int thisManyBytes) {
    int bytesRead = 0;
    if (thisManyBytes > 512) 
        thisManyBytes = 512*(thisManyBytes/512);  // Sector-aligned reads
    if (dataFile.available()) {
        bytesRead += dataFile.read(writeHere, thisManyBytes);
    }
    return bytesRead;
}

void setup() {
    Serial.begin(9600);
    while (!Serial);

    // Initialize I2S: Left-Justified, 44.1kHz, 16-bit
    if (!I2S.begin(I2S_LEFT_JUSTIFIED_MODE, MP3_SAMPLE_RATE_DEFAULT, 16)) {
        Serial.println("Failed to initialize I2S!");
        while (1);
    }
    
    buffersize16 = I2S.bufferSize()/2;  // Size in int16_t samples
    bufferptr = (int16_t *)I2S.bufferPtr();
    I2S.onTransmit(i2sCallback);

    Serial.println("Native MP3 decoding!");
    Serial.print("Initializing SD card...");

    while (!SD.begin()) {
        Serial.println("Card failed, or not present");
        delay(2000);
    }
    Serial.println("card initialized.");

    dataFile = SD.open(filename);
    if (!dataFile) {
        Serial.println("could not open file!");
        while(1);
    }

    player.begin();
    player.setBufferCallback(getMoreData);
    player.setChangeRateCallback(rateCallback);
    
    player.play();
    player.tick();
    I2S.start(true);  // Start DMA transmission
    Serial.println("Playing!");
}

void loop() {
    player.tick();  // Feed decoder, handle callbacks
}
```

### I2S Output Configuration

The I2S driver supports multiple standards. Configure in `I2S.begin()`:

```cpp
// Mode options (from I2S.h)
I2S_PHILIPS_MODE           // Standard I2S (MSb first, left=0)
I2S_LEFT_JUSTIFIED_MODE    // MSB justified (used in example)  ← MAX98357A default
I2S_RIGHT_JUSTIFIED_MODE   // LSB justified
I2S_PCM_MODE               // PCM mode (short/long frame sync)

// Master mode (generates BCLK/LRCK)
I2S.begin(I2S_LEFT_JUSTIFIED_MODE, 44100, 16, true);  // With MCLK
I2S.begin(I2S_LEFT_JUSTIFIED_MODE, 44100, 16, false); // Without MCLK

// Slave mode (receives BCLK/LRCK from external master)
I2S.begin(I2S_LEFT_JUSTIFIED_MODE, 16);
```

### Sample Rate Adaptation

The `rateCallback` automatically reconfigures I2S clocks when MP3 sample rate changes (e.g., 44.1kHz → 48kHz):

```cpp
void rateCallback(uint32_t rate) {
    // Recalculate PLLI2S dividers for new rate
    // I2S.setSampleRate() handles PLL reconfiguration
    I2S.setSampleRate(rate);
}
```

Supported rates: **8, 11.025, 16, 22.05, 32, 44.1, 48 kHz**

---

## 📚 API Quick Reference

### I2SClass (I2S.h)

| Method | Description | Parameters | Returns |
|--------|-------------|------------|---------|
| `I2SClass(spi_dev*)` | Constructor | `SPI2` or `SPI3` | — |
| `begin(mode, rate, bits, mclk)` | Init master mode | `i2s_mode_t`, `sampleRate`, `bits(16/24/32)`, `mclk` | `int` (0=ok) |
| `begin(mode, bits)` | Init slave mode | `i2s_mode_t`, `bits` | `int` |
| `start(isTX)` | Start DMA | `true`=TX, `false`=RX | `bool` |
| `stop()` | Stop DMA | — | `bool` |
| `pause(pause)` | Pause/resume | `true`=pause | `bool` |
| `setSampleRate(rate, mclk)` | Change sample rate | `rate` Hz, `mclk` enable | `bool` |
| `end()` | Deinit peripheral | — | — |
| `bufferPtr()` | Get DMA buffer address | — | `void*` |
| `bufferSize()` | Get buffer size (bytes) | — | `uint16_t` |
| `onTransmit(cb)` | Set TX callback | `void(*)(void)` | — |
| `onReceive(cb)` | Set RX callback | `void(*)(void)` | — |
| `write(data, size)` | Write to TX buffer | `const void*`, `size_t` | `size_t` |
| `read(buf, size)` | Read from RX buffer | `void*`, `size_t` | `int` |
| `available()` | RX bytes available | — | `int` |
| `availableForWrite()` | TX buffer space | — | `size_t` |

### Adafruit_MP3 (Adafruit_MP3.h)

| Method | Description | Parameters | Returns |
|--------|-------------|------------|---------|
| `begin()` | Initialize decoder | — | `bool` |
| `play()` | Start decoding | — | — |
| `stop()` | Stop playback | — | — |
| `resume()` | Resume after stop | — | — |
| `tick()` | Process decode step | — | `int` (samples decoded) |
| `setBufferCallback(cb)` | Set data feed callback | `int(*)(uint8_t*,int)` | — |
| `setSampleReadyCallback(cb)` | Set PCM output callback | `void(*)(int16_t,int16_t)` | — |
| `setChangeRateCallback(cb)` | Set rate change callback | `void(*)(uint32_t)` | — |
| `getSamples(buf, count)` | **STM32F1 only** Direct DMA fill | `volatile int16_t*`, `uint16_t` | — |

### Callbacks

```cpp
// Buffer feed callback - called when decoder needs more MP3 data
int getMoreData(uint8_t *writeHere, int thisManyBytes) {
    // Read from SD card, network, etc.
    return bytesRead;  // 0 = EOF
}

// Sample ready callback - called per stereo frame (alternative to getSamples)
void writeDacs(int16_t left, int16_t right) {
    // Output to DAC, PWM, etc.
}

// Rate change callback - called when MP3 sample rate changes
void rateCallback(uint32_t newRate) {
    I2S.setSampleRate(newRate);
}
```

---

## 🔧 Troubleshooting

### SD Card Not Detected

| Symptom | Cause | Solution |
|---------|-------|----------|
| "Card failed, or not present" | Wiring error | Check SDIO_D0-D3, CLK, CMD connections; verify 3.3V level |
| | Wrong pin mapping | Ensure `SdioF1` uses correct pins (PC8-PC12, PD2) |
| | Card not FAT32 | Reformat as FAT32 (32KB clusters) |
| | Card >32GB | Use ≤32GB card (SDHC) or update SdFat for SDXC |
| | No pull-up on CMD | Add 10kΩ pull-up on PD2 (CMD) to 3.3V |
| | SDIO clock too high | Reduce SDIO clock in `SD.begin(clock_hz)` |

**Debug:** Enable SDIO debug in `SdFatConfig.h`:
```cpp
#define SD_FAT_DEBUG 1
```

---

### No Audio Output

| Symptom | Cause | Solution |
|---------|-------|----------|
| Silence, no noise | I2S not started | Call `I2S.start(true)` after setup |
| | MAX98357A not powered | Check VDD (5V) and SD (3.3V = enabled) |
| | GAIN pin floating | Tie GAIN to GND (15dB) or 3.3V (12dB) |
| | Wrong I2S mode | MAX98357A requires **Left-Justified** (MSB) |
| | MCLK missing | Some boards need MCLK; try `begin(..., true)` |
| | Speaker wiring | Check OUT+/OUT- to speaker (not grounded) |

**Debug:** Probe BCLK (PB13), LRCK (PB12), DIN (PB15) with oscilloscope:
- BCLK: 2.822 MHz @ 44.1kHz/16-bit (64×fs)
- LRCK: 44.1 kHz (50% duty)
- DIN: Data on falling BCLK edge

---

### Choppy / Distorted Playback

| Symptom | Cause | Solution |
|---------|-------|----------|
| Clicks/pops | Buffer underrun | Increase `I2S_BUFFER_SIZE` (default 1152) |
| | SD card too slow | Use Class 10/UHS-I card; sector-aligned reads |
| | CPU overload | Reduce `Serial.print` in callbacks; optimize `tick()` |
| Distortion | Wrong sample rate | Verify `rateCallback` updates I2S correctly |
| | Clipping | Reduce gain or check MP3 volume |
| Speed issues | PLLI2S miscalculated | Check `setSampleRate()` math for target rate |

**Buffer Size Tuning:**
```cpp
// In I2S.h - increase for more margin
#define I2S_BUFFER_SIZE 2304  // Double default (more RAM used)
```

---

### Wrong Sample Rate

| Symptom | Cause | Solution |
|---------|-------|----------|
| Playback too fast/slow | PLLI2S not updated | Ensure `rateCallback` calls `I2S.setSampleRate(rate)` |
| | Fixed 44.1kHz only | MP3 may be 48kHz; decoder calls callback |
| | Clock source wrong | Verify PLLI2S enabled in RCC (CubeMX or register) |

**Manual Rate Test:**
```cpp
void setup() {
    I2S.begin(I2S_LEFT_JUSTIFIED_MODE, 48000, 16);  // Force 48kHz
    // ...
}
```

---

## 🛠️ Customization Guide

### Different STM32F1 Boards

#### STM32F103VET6 (100-pin, SPI2 + SPI3)

```cpp
// Use SPI3 for I2S (PB3, PB4, PB5, PC7, PC10, PC11, PC12)
#include "./I2S.h"
I2SClass I2S(SPI3);  // Instead of SPI2

// Pins for SPI3/I2S3:
// PB3  - I2S3_CK (BCLK)
// PB4  - I2S3_WS (LRCK)  
// PB5  - I2S3_SD (DIN)
// PC7  - I2S3_MCK (MCLK, optional)
// 
// Note: SPI3 shares pins with other peripherals; check AF mapping
```

#### STM32F105/107 (Connectivity Line)

- Same as F103 but with **OTG_FS**, **Ethernet**, **CAN2**
- SPI2/SPI3 I2S identical
- More DMA channels available

#### Custom Board Pin Remapping

If using non-standard pins, modify `I2S.cpp` GPIO initialization:

```cpp
// In I2SClass::begin(), replace GPIO config:
// For SPI2/I2S2 (default):
// PB12=WS, PB13=CK, PB15=SD, (PB14=MISO unused), PC6=MCK (optional)

// For remapped SPI2 (requires AFIO_MAPR):
// AFIO->MAPR |= AFIO_MAPR_SPI2_REMAP;
// Then: PB12→PB12, PB13→PB13, PB15→PB15 (same pins, different AF)
// Actually SPI2 remap moves to: PB12, PB13, PB14, PB15, PC6 - same!
```

### Adjusting Buffer Sizes

**I2S Buffer (I2S.h):**
```cpp
#define I2S_BUFFER_SIZE 1152  // Samples per buffer (×2 buffers = 2304 total)
// Increase for: slower SD, higher bitrate MP3, more CPU load
// Decrease for: lower RAM (minimum ~576 for 44.1kHz)
```

**MP3 Decoder Buffers (Adafruit_MP3.h):**
```cpp
#define OUTBUF_SIZE (2 * 1152)   // Output PCM buffer (stereo)
#define INBUF_SIZE (6 * 512)     // Input MP3 buffer (3KB)
#define IN_BUFFER_LOWER_THRESH (1 * 1024)
#define OUT_BUFFER_LOWER_THRESH (4 * 1152)
```

### Adding Volume Control

The MAX98357A has **no digital volume control**. Options:

1. **Software attenuation** (in `i2sCallback`):
```cpp
void i2sCallback() {
    // ... existing code ...
    player.getSamples(&bufferptr[offset], buffersize16/2);
    
    // Apply volume (0.0-1.0)
    float vol = 0.5f;  // -6dB
    for (int i = 0; i < buffersize16/2; i++) {
        bufferptr[offset + i] = (int16_t)(bufferptr[offset + i] * vol);
    }
}
```

2. **Hardware**: Add digital potentiometer on speaker output or use MAX98357A's GAIN pin (3dB steps)

---

## 📌 Pin Configuration Reference

### Default Pinout (STM32F103C8T6, SPI2/I2S2)

| Signal | STM32 Pin | Port | AF | Function |
|--------|-----------|------|----|----------|
| I2S2_WS (LRCK) | PB12 | GPIOB | AF0 | Word Select / Left-Right Clock |
| I2S2_CK (BCLK) | PB13 | GPIOB | AF0 | Serial Clock |
| I2S2_SD (DIN) | PB15 | GPIOB | AF0 | Serial Data Out |
| I2S2_MCK | PC6 | GPIOC | AF0 | Master Clock (optional) |
| SDIO_D0 | PC8 | GPIOC | AF12 | Data 0 |
| SDIO_D1 | PC9 | GPIOC | AF12 | Data 1 |
| SDIO_D2 | PC10 | GPIOC | AF12 | Data 2 |
| SDIO_D3 | PC11 | GPIOC | AF12 | Data 3 / Card Detect |
| SDIO_CK | PC12 | GPIOC | AF12 | Clock |
| SDIO_CMD | PD2 | GPIOD | AF12 | Command |

### GPIO Configuration (Auto in I2S.begin/SD.begin)

```cpp
// I2S pins: AF Push-Pull, 50MHz, No Pull
// SDIO pins: AF Push-Pull, 50MHz, Pull-Up (CMD, D0-D3)
```

### Alternative: SPI3/I2S3 (F103VE/VG/VT/ZG only)

| Signal | STM32 Pin | Port | AF |
|--------|-----------|------|----|
| I2S3_WS | PB4 / PA15 | GPIOB/GPIOA | AF0/AF6 |
| I2S3_CK | PB3 / PC10 | GPIOB/GPIOC | AF0/AF6 |
| I2S3_SD | PB5 / PC12 | GPIOB/GPIOC | AF0/AF6 |
| I2S3_MCK | PC7 / PA8 | GPIOC/GPIOA | AF0/AF6 |

---

## ⚡ Performance Notes

### Memory Usage (STM32F103C8: 64KB Flash, 20KB RAM)

| Component | Flash | RAM |
|-----------|-------|-----|
| Adafruit_MP3 decoder | ~25 KB | ~4 KB (buffers) |
| I2S Driver | ~3 KB | 4.5 KB (2×1152×2 bytes) |
| SdFat/SDIO | ~15 KB | ~1.5 KB |
| Application + overhead | ~5 KB | ~2 KB |
| **Total** | **~48 KB** | **~12 KB** |
| **Remaining** | **16 KB** | **8 KB** |

### CPU Load

| Task | Cycle Count (est.) | % @ 72 MHz |
|------|-------------------|------------|
| MP3 Decode (44.1kHz stereo) | ~1.5M cycles/sec | ~2% |
| I2S DMA Interrupt (2×/frame) | ~500 cycles × 88k | ~0.6% |
| SDIO Read (sector) | ~10k cycles × 86/sec | ~1.2% |
| **Total** | | **~4%** |

> Plenty of headroom for additional tasks (UI, networking, etc.)

### Maximum Supported MP3

| Parameter | Limit |
|-----------|-------|
| Bitrate | 320 kbps CBR / VBR |
| Sample Rate | 48 kHz (8-48 kHz supported) |
| Channels | Stereo (Mono auto-upmixed) |
| Layer | Layer III (MP3), Layer II, Layer I |
| ID3 Tags | ID3v1, ID3v2 (skipped automatically) |

---

## 🔗 Related Resources

### Documentation & References

- **STM32F1 Reference Manual** (RM0008): [ST.com](https://www.st.com/resource/en/reference_manual/rm0008.pdf)
- **STM32F1 I2S Peripheral**: Section 26 of RM0008
- **MAX98357A Datasheet**: [Analog Devices](https://www.analog.com/media/en/technical-documentation/data-sheets/MAX98357A.pdf)
- **SDIO Specification**: [SD Association](https://www.sdcard.org/downloads/pls/)
- **minimp3 / mp3dec**: [GitHub](https://github.com/lieff/minimp3)

### Library Sources

| Library | Repository |
|---------|------------|
| STM32duino Core | https://github.com/rogerclarkmelbourne/Arduino_STM32 |
| SdFat (SDIO) | https://github.com/greiman/SdFat/tree/1.1.4 |
| Adafruit_MP3 | https://github.com/adafruit/Adafruit_MP3 |
| libmaple (legacy) | https://github.com/leaflabs/libmaple |

### Tools

- **STM32CubeIDE**: https://www.st.com/en/development-tools/stm32cubeide.html
- **STM32CubeMX**: https://www.st.com/en/development-tools/stm32cubemx.html
- **OpenOCD**: http://openocd.org/
- **VS Code + PlatformIO**: https://platformio.org/

### Community

- **STM32duino Forum**: https://github.com/stm32duino/Arduino_Core_STM32/discussions
- **Arduino Forum - STM32**: https://forum.arduino.cc/c/hardware/STM32/
- **EEVBlog Forum**: https://www.eevblog.com/forum/microcontrollers/

---

## 📄 License

This project is licensed under the **MIT License** - see the [LICENSE](LICENSE) file for details.

### Third-Party Licenses

| Component | License |
|-----------|---------|
| Adafruit_MP3 | MIT |
| mp3dec (minimp3) | Public Domain / MIT |
| I2S Driver (Victor Perez) | MIT |
| SdFat | MIT / BSD-3-Clause |
| STM32duino Core | Apache-2.0 / BSD |

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

### Coding Standards

- Follow existing code style (4-space indent, CamelCase for classes)
- Add comments for public APIs
- Test on actual hardware before submitting
- Update README for new features

---

## 📞 Support

- **Issues**: [GitHub Issues](https://github.com/your-repo/stm32f1-mp3-player/issues)
- **Discussions**: [GitHub Discussions](https://github.com/your-repo/stm32f1-mp3-player/discussions)
- **Email**: your-email@example.com

---

> **Note**: This project is designed for STM32F1 series with libmaple/stm32duino core. For STM32F4/F7/H7, consider using the native STM32HAL/I2S drivers with DMA and the same Adafruit_MP3 decoder.

---

*Last Updated: 2024*  
*Version: 1.0.0*
