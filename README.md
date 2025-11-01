# HUB75 RGB LED Matrix Display Component for ESPHome

This ESPHome component provides support for HUB75 RGB LED matrix panels using the [esp-hub75](https://github.com/stuartparmenter/esp-hub75) library, which uses DMA (Direct Memory Access) for efficient, low-CPU-overhead driving of LED matrix panels.

HUB75 displays are RGB LED matrix panels that use parallel row updating to create dynamic, colorful displays. They are commonly available in sizes like 64×32, 64×64, and can be chained together to create larger displays.

## Hardware Requirements

### Supported ESP32 Variants
- ✅ **ESP32-S3 (GDMA)** - Working
- ⏳ **ESP32/S2 (I2S)** - Implemented, untested on hardware
- ✅ **ESP32-P4 (PARLIO)** - Working (uses PSRAM via EDMA)
- ⏳ **ESP32-C6 (PARLIO)** - Implemented, untested on hardware
- ❌ **ESP32-C3, C2, H2** - Not currently supported

### Memory Considerations
- Uses significant internal SRAM for DMA buffering
- Memory usage increases with panel resolution and number of panels
- Larger configurations may require careful memory management

### Power Supply
- **Use a dedicated 5V power supply** for the LED panels
- LED matrices can draw significant current (several amps for larger displays)
- Proper grounding between ESP32 and panel is essential
- Do not power the panels from the ESP32's 5V pin

## Pin Connections

Standard HUB75 panels use the following connections:

| HUB75 Pin | Description | Default GPIO | ESPHome Config |
|-----------|-------------|--------------|----------------|
| R1 | Red data (top half) | 25 | `r1_pin` |
| G1 | Green data (top half) | 26 | `g1_pin` |
| B1 | Blue data (top half) | 27 | `b1_pin` |
| R2 | Red data (bottom half) | 14 | `r2_pin` |
| G2 | Green data (bottom half) | 12 | `g2_pin` |
| B2 | Blue data (bottom half) | 13 | `b2_pin` |
| A | Row select bit 0 | 23 | `a_pin` |
| B | Row select bit 1 | 19 | `b_pin` |
| C | Row select bit 2 | 5 | `c_pin` |
| D | Row select bit 3 | 17 | `d_pin` |
| E | Row select bit 4 (optional) | - | `e_pin` |
| LAT | Latch | 4 | `lat_pin` |
| OE | Output Enable (active low) | 15 | `oe_pin` |
| CLK | Clock | 16 | `clk_pin` |
| GND | Ground | - | - |

**Notes:**
- The E pin is only required for 1/32 scan panels (64 pixels high). Most 32-pixel high panels don't need it.
- Some default pins are strapping pins on ESP32 - consider changing them if you experience boot issues

## Board Presets

Instead of manually configuring all 14 pins, you can use a board preset that automatically sets the correct pin mappings for your hardware:

```yaml
display:
  - platform: hub75
    board: apollo-automation-m1-rev4  # Automatic pin configuration!
    panel_width: 64
    panel_height: 64
    lambda: |-
      it.print(0, 0, id(font), "Hello World!");
```

### Supported Boards

| Board Name | Description | Pins Configured |
|------------|-------------|-----------------|
| `apollo-automation-m1-rev4` | Apollo Automation M1 Rev4 | All 14 HUB75 pins |
| `apollo-automation-m1-rev6` | Apollo Automation M1 Rev6 | All 14 HUB75 pins |
| `adafruit-matrix-portal-s3` | Adafruit Matrix Portal S3 | All 14 HUB75 pins |
| `huidu-hd-wf2` | Huidu HD-WF2 Controller | All 14 HUB75 pins |

### Board Preset Features

- **Automatic pin mapping** - No need to look up and configure 14 individual pins
- **Strapping warnings handled** - Boards with strapping pins (like Adafruit) automatically include ignore_strapping_warning flags
- **Pin overrides** - You can still override individual pins if needed:

```yaml
display:
  - platform: hub75
    board: apollo-automation-m1-rev4
    oe_pin: GPIO16  # Override just the OE pin
    panel_width: 64
    panel_height: 64
```

- **Manual configuration still supported** - If you don't specify a board, all pins must be configured manually (backwards compatible)

## Basic Usage

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 32
    lambda: |-
      it.print(0, 0, id(font), "Hello World!");
```

**Important:** This component does **NOT** require an SPI bus configuration. It uses I2S and DMA internally.

## Configuration Variables

### Required Parameters

- **panel_width** (**Required**, int): Width of a single panel in pixels (e.g., `64`)
- **panel_height** (**Required**, int): Height of a single panel in pixels (e.g., `32` or `64`)

### Multi-Panel Layout

Chain multiple panels together horizontally or arrange them in a 2D grid:

- **layout_rows** (*Optional*, int): Number of panel rows in the grid. Defaults to `1`.
- **layout_cols** (*Optional*, int): Number of panel columns in the grid. Defaults to `1`.
- **layout** (*Optional*, enum): Panel layout pattern. Defaults to `HORIZONTAL`.
  - `HORIZONTAL` - Single row, left to right (default)
  - `TOP_LEFT_DOWN` - Grid starting top-left, going down each column
  - `TOP_RIGHT_DOWN` - Grid starting top-right, going down each column
  - `BOTTOM_LEFT_UP` - Grid starting bottom-left, going up each column
  - `BOTTOM_RIGHT_UP` - Grid starting bottom-right, going up each column
  - `TOP_LEFT_DOWN_ZIGZAG` - Serpentine pattern, alternating direction
  - `TOP_RIGHT_DOWN_ZIGZAG` - Serpentine pattern, alternating direction
  - `BOTTOM_LEFT_UP_ZIGZAG` - Serpentine pattern, alternating direction
  - `BOTTOM_RIGHT_UP_ZIGZAG` - Serpentine pattern, alternating direction

### Display Configuration

- **brightness** (*Optional*, int): Initial brightness level (0-255). Defaults to `128`.
- **bit_depth** (*Optional*, int): Color bit depth (6-12). Higher values = better color at the cost of refresh rate. Defaults to `8`.
- **min_refresh_rate** (*Optional*, int): Minimum panel refresh rate in Hz (30-200). Controls how fast the panel hardware refreshes, independent of ESPHome's update_interval. Defaults to `60`.
- **double_buffer** (*Optional*, boolean): Enable double buffering to reduce tearing. Defaults to `false`.
- **update_interval** (*Optional*, [Time](https://esphome.io/guides/configuration-types.html#config-time)): How often ESPHome calls the display update lambda. Set to `never` for LVGL. Defaults to `1s`.

### Panel Hardware Configuration

- **scan_wiring** (*Optional*, enum): Panel scan pattern. Defaults to `STANDARD_TWO_SCAN`.
  - `STANDARD_TWO_SCAN` - Standard 1/16 or 1/32 scan (most common)
  - `FOUR_SCAN_16PX_HIGH` - 1/4 scan for 16px high panels
  - `FOUR_SCAN_32PX_HIGH` - 1/4 scan for 32px high panels
  - `FOUR_SCAN_64PX_HIGH` - 1/4 scan for 64px high panels

- **shift_driver** (*Optional*, enum): Shift register driver IC type. Defaults to `GENERIC`.
  - `GENERIC` - Standard shift registers (default)
  - `FM6126A` - Common improved driver chip
  - `ICN2038S` - High-performance driver
  - `FM6124` - Older FM driver
  - `MBI5124` - Macroblock driver (requires `clock_phase: true`)
  - `DP3246` - Another common driver

### Pin Configuration

All pins are optional and have defaults (see Pin Connections table above):

```yaml
r1_pin: GPIO25
g1_pin: GPIO26
b1_pin: GPIO27
r2_pin: GPIO14
g2_pin: GPIO12
b2_pin: GPIO13
a_pin: GPIO23
b_pin: GPIO19
c_pin: GPIO5
d_pin: GPIO17
e_pin: GPIO18  # Only needed for 1/32 scan panels
lat_pin: GPIO4
oe_pin: GPIO15
clk_pin: GPIO16
```

### Advanced Timing Configuration

- **clock_speed** (*Optional*, enum): I2S clock speed for data transfer.
  - `8MHZ` - 8 MHz (slowest, most compatible)
  - `10MHZ` - 10 MHz
  - `16MHZ` - 16 MHz
  - `20MHZ` - 20 MHz (fastest, default)

- **latch_blanking** (*Optional*, int): Latch blanking time. Defaults to `1`.
- **clock_phase** (*Optional*, boolean): Invert clock phase. Required for `MBI5124` driver. Defaults to `false`.

### Standard Display Options

All standard ESPHome [Display](https://esphome.io/components/display/index.html#config-display) configuration options are also available.

## Examples

### Using a Board Preset (Simplest)

```yaml
display:
  - platform: hub75
    board: apollo-automation-m1-rev4  # Automatic pin configuration
    id: matrix_display
    panel_width: 64
    panel_height: 64
    brightness: 128
    lambda: |-
      it.print(0, 0, id(font), "Hello!");
```

This is the simplest configuration - the board preset automatically configures all 14 pins!

### Single 64×32 Panel (Manual Pin Configuration)

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 32
    brightness: 128
    lambda: |-
      it.print(0, 0, id(font), "Hello!");
```

When no board is specified, default pins are used (see Pin Connections table).

### Horizontal Chain (3 panels = 192×32)

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 32
    layout_cols: 3
    layout_rows: 1
    layout: HORIZONTAL
    brightness: 100
    lambda: |-
      it.print(0, 0, id(font), "Wide Display!");
```

The total display size will be 192×32 pixels (3 panels × 64 pixels wide).

### 2×2 Panel Grid (128×64)

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 32
    layout_cols: 2
    layout_rows: 2
    layout: TOP_LEFT_DOWN_ZIGZAG
    brightness: 128
    lambda: |-
      it.filled_rectangle(0, 0, 128, 64, COLOR_ON);
      it.print(64, 32, id(font), TextAlign::CENTER, "Grid!");
```

The total display size will be 128×64 pixels (2×2 grid of 64×32 panels).

### Custom Pin Configuration

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 32
    # Custom pins to avoid strapping pins
    r1_pin: GPIO2
    g1_pin: GPIO15
    b1_pin: GPIO4
    r2_pin: GPIO16
    g2_pin: GPIO27
    b2_pin: GPIO17
    a_pin: GPIO5
    b_pin: GPIO18
    c_pin: GPIO19
    d_pin: GPIO21
    lat_pin: GPIO26
    oe_pin: GPIO25
    clk_pin: GPIO22
    lambda: |-
      it.print(0, 0, id(font), "Custom pins!");
```

### Advanced Configuration with FM6126A Driver

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 64
    shift_driver: FM6126A
    scan_wiring: STANDARD_TWO_SCAN
    bit_depth: 8
    min_refresh_rate: 120  # Higher refresh rate for smoother display
    brightness: 128
    clock_speed: 20MHZ
    double_buffer: true
    e_pin: GPIO18  # Required for 64px high panels
    lambda: |-
      it.print(0, 0, id(font), "FM6126A");
```

### LVGL Integration

When using with LVGL, configure as follows:

```yaml
display:
  - platform: hub75
    id: matrix_display
    panel_width: 64
    panel_height: 64
    update_interval: never  # LVGL controls updates
    auto_clear_enabled: false  # LVGL handles clearing
    double_buffer: false  # LVGL has its own buffering
    min_refresh_rate: 60  # Panel refresh rate (independent of LVGL)
    brightness: 128

lvgl:
  displays:
    - display_id: matrix_display
  # ... rest of LVGL config
```

**Key points for LVGL:**
- Set `update_interval: never` - LVGL controls when to update
- Set `auto_clear_enabled: false` - LVGL handles clearing
- Set `double_buffer: false` - LVGL has its own buffering
- `min_refresh_rate` controls panel hardware refresh, not LVGL update rate

## Important Notes

### Memory Usage
- Each panel requires significant RAM for DMA buffers
- 64×32 panel with 8-bit depth ≈ 24 KB RAM
- Multiple panels or higher bit depths increase memory usage
- Large configurations may require `CONFIG_ESP32_SPIRAM_SUPPORT=y`

### Strapping Pins
Some default GPIOs are strapping pins that affect boot behavior:
- **GPIO0** - Boot mode selection (avoid if possible)
- **GPIO2** - Boot mode on some modules
- **GPIO5** - JTAG, timing source (used by default for C pin)
- **GPIO12** - Flash voltage (used by default for G2 pin)
- **GPIO15** - Boot message enable (used by default for OE pin)

If you experience boot issues, consider remapping these pins using the pin configuration options.

### Driver Compatibility
- Use `shift_driver: GENERIC` if unsure - works with most panels
- `FM6126A` and `ICN2038S` are common upgrade chips with better color
- `MBI5124` **requires** `clock_phase: true` to function properly
- Check your panel's specifications or driver IC markings

### Power Requirements
- Calculate power: (width × height ÷ 2) × 0.06A at maximum brightness white
- Example: 64×32 panel ≈ 0.96A at full brightness
- Use power supplies with adequate current capacity and proper voltage regulation
- Add capacitors (1000µF+) near panel power inputs

### E Pin Requirement
- **1/16 scan (32px high):** E pin typically not needed
- **1/32 scan (64px high):** E pin required, must be configured

### Rotation Support
- **Single panel rotation** (0°, 90°, 180°, 270°) is not currently supported
- **Panel chaining layouts** can handle per-panel orientation via `layout` configuration (e.g., `TOP_LEFT_DOWN`, `BOTTOM_RIGHT_UP`, etc.)
- For multi-panel displays, physical rotation and electrical chaining patterns are handled by the layout options

## Troubleshooting

### Display is blank or flickering
- Check power supply voltage and current capacity
- Verify all pin connections match your configuration
- Try lowering `clock_speed` to `8MHZ` or `10MHZ`
- Ensure `shift_driver` matches your panel's driver IC

### Wrong colors or corrupted image
- Verify RGB pin connections (R1/G1/B1/R2/G2/B2)
- Check if your panel uses a specific `shift_driver` (try `FM6126A` or `ICN2038S`)
- Try different `clock_speed` settings

### ESP32 won't boot / boot loops
- Strapping pins may be interfering - reconfigure pins to avoid GPIO0, GPIO2, GPIO5, GPIO12, GPIO15
- Check serial output for boot messages

### Panel layout is wrong with multiple panels
- Verify `layout_cols` and `layout_rows` match physical arrangement
- Try different `layout` patterns (HORIZONTAL, TOP_LEFT_DOWN, etc.)
- For serpentine wiring, use `_ZIGZAG` variants

## See Also

- [ESPHome Display Component](https://esphome.io/components/display/index.html)
- [ESPHome LVGL Component](https://esphome.io/components/lvgl.html)
- [esp-hub75 Library](https://github.com/stuartparmenter/esp-hub75)
- [HUB75 Panel Information](https://www.sparkfun.com/sparkfun-rgb-led-matrix-panel-hookup-guide)
