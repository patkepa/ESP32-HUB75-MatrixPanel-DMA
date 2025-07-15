# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an ESP32 Arduino/IDF library for driving HUB75/HUB75E RGB LED matrix panels using DMA (Direct Memory Access) for high-performance, low-CPU overhead rendering. The library supports multiple ESP32 variants and provides both AdafruitGFX-compatible and native APIs.

## Build System and Common Commands

### Framework Support
- **Arduino IDE**: Standard Arduino library installation
- **PlatformIO**: Include in `platformio.ini` lib_deps or install via library manager
- **ESP-IDF**: Native component support with CMake

### Build Commands
For PlatformIO projects:
```bash
pio run                    # Build project
pio run -t upload          # Build and upload
pio run -t monitor         # Monitor serial output
```

For ESP-IDF projects:
```bash
idf.py build               # Build project
idf.py flash               # Flash to device
idf.py monitor             # Monitor serial output
idf.py flash monitor       # Flash and monitor
```

### Testing
- Use the `examples/PIO_TestPatterns` example for hardware testing
- No automated test suite - examples serve as integration tests
- Test on actual hardware with LED panels

## Architecture

### Core Components

#### MatrixPanel_I2S_DMA Class (`src/ESP32-HUB75-MatrixPanel-I2S-DMA.h`)
- Main library class that handles DMA-based panel driving
- Supports AdafruitGFX API when enabled
- Manages framebuffer memory allocation and DMA descriptors
- Handles color depth and refresh rate calculations

#### Platform Abstraction (`src/platforms/`)
- **ESP32 Original**: Uses I2S parallel DMA (`esp32/esp32_i2s_parallel_dma.cpp`)
- **ESP32-S2**: Uses I2S parallel DMA with different FIFO ordering
- **ESP32-S3**: Uses GDMA with LCD parallel interface (`esp32s3/gdma_lcd_parallel16.cpp`)
- **ESP32-C6/H2**: Not supported due to hardware limitations
- Platform detection via `platform_detect.hpp`

#### VirtualMatrixPanel (`src/ESP32-HUB75-VirtualMatrixPanel_T.hpp`)
- Handles panel chaining and virtual coordinate mapping
- Supports both horizontal and 2D grid panel arrangements
- Manages scan rate differences (1/16, 1/32, 1/4 scan panels)
- Template-based implementation for performance

#### LED Driver Support (`src/ESP32-HUB75-MatrixPanel-leddrivers.cpp`)
- Handles specific LED driver chip initialization
- Supports FM6126A, FM6124, ICN2038S, and other compatible chips
- Provides chip-specific configuration sequences

### Memory Management
- DMA buffers allocated in internal SRAM (ESP32/S2) or PSRAM (ESP32-S3 with OCTAL-SPI)
- Color depth directly impacts memory usage (~2.5 bits per pixel per color depth bit)
- Double buffering optional for smoother animations
- Memory calculator available in `doc/memcalc.md`

## Build Options

Critical build flags (set in `platformio.ini` build_flags or CMake):

### Performance Options
- `NO_GFX=1`: Disable AdafruitGFX API for memory savings
- `USE_GFX_LITE=1`: Use lightweight GFX implementation
- `NO_FAST_FUNCTIONS=1`: Disable optimized drawing functions
- `NO_CIE1931=1`: Disable brightness correction

### Hardware Options
- `SPIRAM_DMA_BUFFER=1`: Use PSRAM for DMA buffer (ESP32-S3 with OCTAL-SPI only)
- `PIXEL_COLOR_DEPTH_BITS=8`: Set color depth (2-8 bits, default 8)
- `FORCE_COLOR_DEPTH=1`: Force color depth regardless of refresh rate

### Panel Configuration
- `MATRIX_WIDTH=64`: Panel width in pixels
- `MATRIX_HEIGHT=32`: Panel height in pixels (set to 64 for 64x64 panels)

## Key Files and Directories

### Source Structure
- `src/ESP32-HUB75-MatrixPanel-I2S-DMA.{h,cpp}`: Main library implementation
- `src/platforms/`: Platform-specific DMA implementations
- `src/ESP32-HUB75-VirtualMatrixPanel_T.hpp`: Panel chaining support

### Examples
- `examples/1_SimpleTestShapes/`: Basic usage demonstration
- `examples/2_PatternPlasma/`: Panel chaining example
- `examples/VirtualMatrixPanel/`: 2D grid panel arrangement
- `examples/PIO_TestPatterns/`: Hardware testing patterns
- `examples/AuroraDemo/`: Advanced visual effects

### Configuration
- `library.json`: PlatformIO library metadata
- `library.properties`: Arduino IDE library metadata
- `CMakeLists.txt`: ESP-IDF component configuration
- `doc/BuildOptions.md`: Comprehensive build option documentation

## Panel Compatibility

### Supported Panels
- **Two-scan panels**: 64x32 (1/16 scan), 64x64 (1/32 scan)
- **Four-scan panels**: 32x16 (1/4 scan), 126x64 (SM5266P)
- **Driver chips**: ICND2012, RUC7258, FM6126A/ICN2038S, FM6124, SM5266P

### Unsupported Panels
- S-PWM or PWM-based chips (RUL6024, MBI6024, etc.)
- Self-PWM generating panels
- ICN2053/FM6353 based panels (use separate fork)
- RUL5358/SHIFTREG_ABC_BIN_DE based panels

## Common Development Patterns

### Basic Panel Initialization
```cpp
#include <ESP32-HUB75-MatrixPanel-I2S-DMA.h>

MatrixPanel_I2S_DMA *dma_display = nullptr;

void setup() {
    HUB75_I2S_CFG mxconfig(64, 32, 1); // width, height, chain_length
    dma_display = new MatrixPanel_I2S_DMA(mxconfig);
    dma_display->begin();
}
```

### Custom Pin Configuration
```cpp
HUB75_I2S_CFG::i2s_pins _pins = {R1, G1, B1, R2, G2, B2, A, B, C, D, E, LAT, OE, CLK};
HUB75_I2S_CFG mxconfig(64, 32, 1, _pins);
```

### Performance Optimization
- Use `NO_GFX` for maximum performance
- Reduce `PIXEL_COLOR_DEPTH_BITS` for large displays
- Enable double buffering for smooth animations
- Use FastLED or direct pixel manipulation for complex effects

## Hardware Considerations

### Power Requirements
- Use adequate power supplies (5V, high current)
- Add 1000-2000μF capacitors across VCC/GND on each panel
- Consider voltage levels (ESP32 outputs 3.3V, some panels need 5V logic)

### Wiring
- Default pin assignments in `src/platforms/*/esp32*-default-pins.hpp`
- Connect ground pins between ESP32 and panels
- Use proper ribbon cables for panel chaining

### ESP32 Variant Considerations
- **ESP32-S3**: May have WiFi interference issues with some PCBs
- **ESP32-S3 with OCTAL-SPI PSRAM**: Can use SPIRAM_DMA_BUFFER
- **ESP32-C3/C6/H2**: Not supported due to hardware limitations

## Troubleshooting

### Common Issues
- **Memory errors**: Reduce color depth or panel size
- **Flickering**: Check power supply, try different latch blanking values
- **Ghosting**: Adjust clock phase or latch blanking
- **Color issues**: Verify driver chip compatibility

### Debug Options
- Set `CORE_DEBUG_LEVEL=3` for detailed memory allocation info
- Use serial monitor to check DMA buffer allocation
- Test with `PIO_TestPatterns` example for hardware validation