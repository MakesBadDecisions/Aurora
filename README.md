# 🌅 Aurora Lights

**Automatic LED effects for Klipper 3D printers.**

Aurora monitors your printer state and automatically changes LED colors/effects. No macro calls needed - just configure and print.

---

## Features

- **Automatic state detection** - Responds to idle, heating, printing, cooldown, error
- **Group-based configuration** - Assign different effects to different LED zones
- **Performance-first design** - Minimal CPU impact, safe for Pi Zero 2W
- **40+ built-in colors** - Or define your own

---

## Quick Start

### 1. Install
Copy `Aurora/` folder to your Klipper config directory:
```
~/printer_data/config/Aurora/
```
### 2. Move Aurora.cfg
```
Move Aurora.cfg out of Aurora folder and into main config folder. Next to printer.cfg :
```
### 3. Include
Add to `printer.cfg`:
```
[include Aurora.cfg]
```
### 4. Configure Aurora
Edit `Aurora.cfg` - update the `[neopixel]` sections to match your hardware:
```
[neopixel toolhead_leds]
pin: EBBCan:PD3       # Your pin
chain_count: 3         # Your LED count
color_order: GRB       # Usually GRB for WS2812

Set groups according to Aurora.cfg based on your physical neopixel location and how you want them grouped.
Then assign colors and effects as needed. Beware of limitations of Aurora, clearly outlined in the documentation. 
```

### 5. Add AURORA_WAKE to Macros
Add to your `PRINT_START` (after heating, before printing):
```
AURORA_WAKE   # Sync Aurora after blocking operations
```

Add to your `PRINT_END` (after heaters off):
```
AURORA_WAKE   # Detect cooldown state

Add anywhere you need to wake Aurora to update your groups based on events and effects
```

### 6. Restart
```
FIRMWARE_RESTART
```

Aurora starts automatically. LEDs will respond to printer state changes.

---

## Events

Aurora automatically detects these printer states:

| Event | Trigger | Example Use |
|-------|---------|-------------|
| `idle` | Printer cool and ready | Soft accent lighting |
| `heating` | Heaters on, warming up | Orange pulse |
| `printing` | Active print (at temp) | Blue/white work lights |
| `cooldown` | Print done, still hot | Pastel breathing |
| `error` | Klipper shutdown/error | Red alert |
| `bored` | Idle too long (5 min) | Party mode! |
| `sleep` | Bored too long (5 min) | LEDs off |

---

## Configuration

### LED Groups

Aurora organizes LEDs into groups and sub-groups:

```
Group A (toolhead_leds)
├── A1: emblem (LED 1)
└── A2: worklights (LEDs 2-3)

Group B (chamber_lights)  
├── B1: left_strip (LEDs 1-18)
└── B2: right_strip (LEDs 19-36)
```

### Assigning Effects

In `Aurora.cfg`, set what happens for each event per sub-group:

```
#---------- Sub-Group A1: Emblem ----------
variable_group_a1_on_idle: "seafoam"
variable_group_a1_on_heating: "pulse"
variable_group_a1_on_printing: "portal_blue"
variable_group_a1_on_cooldown: "baby_blue"
variable_group_a1_on_error: "red"
variable_group_a1_on_bored: "disco"
variable_group_a1_on_sleep: "off"
```

**Values can be:**
- Color name: `"green"`, `"cyan"`, `"portal_blue"` → solid color
- Effect name: `"pulse"`, `"heartbeat"`, `"disco"` → animated effect (uses effect's default color)
- `"off"` → LEDs off

---

## Available Colors

### Basic
`red`, `green`, `blue`, `white`, `black`

### Warm
`orange`, `yellow`, `gold`, `amber`, `coral`, `salmon`, `warm_white`

### Cool
`cyan`, `teal`, `aqua`, `sky`, `ice`, `cool_white`

### Purple/Pink
`purple`, `magenta`, `pink`, `hot_pink`, `violet`, `lavender`, `plum`

### Green Family
`lime`, `mint`, `emerald`, `forest`

### Special
`fire`, `lava`, `sunset`, `sunrise`, `ocean`, `steel`, `copper`, `bronze`
`rose`, `peach`, `cream`, `electric`, `neon_green`, `blood`, `royal`, `cobalt`

### Neon
`neon_pink`, `neon_blue`, `neon_orange`, `neon_yellow`, `neon_purple`

### Pastels
`baby_blue`, `baby_pink`, `seafoam`, `lilac`, `buttercup`

### Earth
`sand`, `clay`, `moss`, `bark`, `stone`

### Gaming
`vaporwave`, `cyberpunk`, `matrix`, `portal_blue`, `portal_orange`

---

## Available Effects

| Effect | Description |
|--------|-------------|
| `solid` | Static color (default for color names) |
| `pulse` | Smooth breathing animation |
| `heartbeat` | Double-pulse pattern |
| `kit` | KITT scanner sweep |
| `disco` | Random color party |
| `progress` | Print progress bar |

**Note:** Effect colors are configured in the Effect Settings section of `Aurora.cfg`

---

## Settings

```
[gcode_macro _AURORA_SETTINGS]
variable_debug: 1                    # 0=silent, 1=state changes
variable_max_brightness: 0.4         # Global cap (0.0-1.0)
variable_update_rate: 0.1            # Poll rate when idle (seconds)
variable_update_rate_printing: 1.0   # Slower during prints (CPU safety)
variable_temp_floor: 25              # Ambient temp baseline (°C)
variable_bored_timeout: 300          # Seconds idle → bored (5 min)
variable_sleep_timeout: 300          # Seconds bored → sleep (5 min)
```

---

## Debug Commands

```gcode
AURORA_STATUS              # Show current state and timers
AURORA_STOP                # Stop the dispatcher
AURORA_START               # Restart the dispatcher  
AURORA_WAKE                # Force state detection (used in macros)
AURORA_TEST_EVENT EVENT=heating    # Force an event for testing
AURORA_SHOW_COLOR COLOR=cyan       # Preview a color
AURORA_LIST_COLORS                 # Show all available colors
```

---

## File Structure

```
Aurora Lights/
├── Aurora.cfg             # YOUR CONFIG - edit this!
├── macros.cfg             # PRINT_START/END with AURORA_WAKE
├── README.md              
└── Aurora/
    ├── includes.cfg       # Module loader
    ├── core/
    │   ├── state.cfg      # Runtime state variables
    │   ├── colors.cfg     # 40+ color definitions
    │   ├── led_output.cfg # LED abstraction layer
    │   ├── dispatcher.cfg # State detection brain
    │   └── debug.cfg      # Debug/utility commands
    └── effects/
        ├── solid.cfg      # Static color
        ├── pulse.cfg      # Breathing animation
        ├── heartbeat.cfg  # Double-pulse pattern
        ├── kit.cfg        # KITT scanner sweep
        ├── disco.cfg      # Random party colors
        └── progress.cfg   # Print progress bar
```

---

## How It Works

1. **Boot** - Aurora initializes, detects current state, sets LEDs
2. **Dispatcher loop** - Polls printer state every 0.1-1.0 seconds
3. **State detection** - Uses `print_stats.state` (same as Mainsail/KlipperScreen)
4. **Event trigger** - When state changes, fires effects for all groups
5. **Effects run** - Solid colors set immediately, animations loop until next event

### State Detection Priority
```
1. error     → webhooks.state == "shutdown" or "error"
2. printing  → print_stats.state == "printing" AND at temp
3. heating   → heaters on AND not at target temp
4. cooldown  → heaters off AND still hot (print done)
5. idle      → cool and ready
6. bored     → idle for 5 minutes
7. sleep     → bored for 5 minutes
```

**Key insight:** During PRINT_START, even though `print_stats.state == "printing"`, Aurora shows "heating" until temps are reached. This matches user expectations.

---

## Technical Notes

### Why `print_stats.state` instead of `idle_timeout.state`?

`idle_timeout.state` is unreliable - it flickers between states during cooldown and doesn't accurately reflect print status. `print_stats.state` (from Klipper's virtual_sdcard) is what Mainsail and KlipperScreen use.

**Source:** [KlipperScreen printer.py](https://github.com/KlipperScreen/KlipperScreen/blob/master/ks_includes/printer.py)

### Performance

- Single-threaded polling (no async complexity)
- 0.1s rate when idle/heating, 1.0s during prints
- Minimal Jinja2 - simple comparisons only
- No impact on print quality

### Limitations

- **Animated effects (pulse, heartbeat, kit) can only run on ONE group at a time** - if multiple groups use the same effect, only the last one will animate. Use solid colors for other groups.
- `G4` (dwell) blocks all animations
- Effects don't blend/stack

---

## Adding Custom Events (Advanced)

Want Aurora to show a special color during bed mesh calibration? Here's how!

### Understanding Blocking Commands

Some G-code commands **block** - Klipper stops and waits for them to finish:
- `G28` - Homing
- `BED_MESH_CALIBRATE` - Bed meshing  
- `Z_TILT_ADJUST` - Gantry leveling
- `M109` / `M190` - Wait for temperature

During blocking commands, Aurora's dispatcher **pauses**. It can't update LEDs until the command finishes.

Other commands are **instant** and don't block:
- `G90` / `G91` - Set positioning mode
- `G0` / `G1` - Movement commands
- `M104` / `M140` - Set temperature (doesn't wait)

### Example: Bed Mesh Event

Let's make LEDs turn purple during bed mesh calibration.

**Step 1: Add the event to Aurora.cfg**

Find the group assignments and add a new line:
```
variable_group_a1_on_meshing: "purple"
variable_group_b1_on_meshing: "purple"
```

**Step 2: Modify your macro**

In your `PRINT_START` (or wherever you call bed mesh):

```
# BEFORE (no Aurora feedback)
BED_MESH_CALIBRATE

# AFTER (Aurora shows purple during mesh)
AURORA_TEST_EVENT EVENT=meshing    # Set purple LEDs
BED_MESH_CALIBRATE                 # Blocking - LEDs stay purple
AURORA_WAKE                        # Resume normal state detection
```

**Why this works:**
1. `AURORA_TEST_EVENT` forces the "meshing" event → LEDs turn purple
2. `BED_MESH_CALIBRATE` blocks → dispatcher paused, LEDs stay purple
3. `AURORA_WAKE` restarts dispatcher → detects current state (probably "heating")

### The Pattern

For ANY blocking operation you want Aurora to highlight:

```
AURORA_TEST_EVENT EVENT=your_event   # Force your custom event
BLOCKING_COMMAND                      # The slow command
AURORA_WAKE                           # Return to normal
```

### Another Example: Homing

```
variable_group_a1_on_homing: "kit:cyan"    # KITT sweep during homing!

# In your macro:
AURORA_TEST_EVENT EVENT=homing
G28
AURORA_WAKE
```

### Quick Reference

| Command | Blocking? | Needs WAKE after? |
|---------|-----------|-------------------|
| `G28` | ✅ Yes | ✅ Yes |
| `BED_MESH_CALIBRATE` | ✅ Yes | ✅ Yes |
| `Z_TILT_ADJUST` | ✅ Yes | ✅ Yes |
| `M109` / `M190` | ✅ Yes | ✅ Yes |
| `G0` / `G1` | ❌ No | ❌ No |
| `M104` / `M140` | ❌ No | ❌ No |
| `G90` / `G91` | ❌ No | ❌ No |

**Rule of thumb:** If the command makes you wait, it's blocking. Add `AURORA_WAKE` after it!

---

## Roadmap

### v1.0 (Current)
- [x] Solid, pulse, heartbeat, kit, disco effects
- [x] Print progress effect
- [x] State detection (error/heating/printing/cooldown/idle/bored/sleep)
- [x] AURORA_WAKE for macro integration
- [x] 40+ built-in colors

### Future
- [ ] Thermal effect (temp-reactive colors)
- [ ] RGBW native support
- [ ] Brightness per-group
- [ ] Layer-change flash

### Python "LUMEN" Version (v2.0)
*Lights Under My Enclosure Now*
- [ ] Standalone Python service (not blocked by G-code)
- [ ] Direct GPIO control via rpi_ws281x (bypass MCU for high-FPS)
- [ ] Moonraker WebSocket for real-time state updates
- [ ] True 60fps animations without macro limitations
- [ ] Dual backend: GPIO for host-attached LEDs + Moonraker for MCU LEDs

---

## Troubleshooting

### LEDs stuck on "printing" after print ends
- Add `AURORA_WAKE` to your PRINT_END macro (see Quick Start step 5)
- The dispatcher can die during long prints - AURORA_WAKE rescues it

### LEDs don't change during PRINT_START
- Normal! Klipper blocks delayed_gcode during PRINT_START
- AURORA_WAKE triggers at the end of PRINT_START to catch up

### Thermal effect doesn't update during homing/probing
- Normal! Klipper freezes ALL delayed_gcode during blocking commands (G28, Z_TILT_ADJUST, BED_MESH, etc.)
- Thermal shows correct initial state when started, but can't update until blocking ends
- After blocking commands complete, AURORA_WAKE restarts updates
- This is a fundamental Klipper limitation, not an Aurora bug

### No "heating" during preheat
- Check: Is the dispatcher running? Try `AURORA_STATUS`
- Try `AURORA_START` to restart dispatcher

### LEDs stuck on last color
- If state is "sleep": Any movement/heating will wake it
- If effect is `"off"`: This properly turns LEDs off (expected)

### Rapid state flickering
- Shouldn't happen with v1.0 - we use `print_stats.state`
- If you see it, run `AURORA_STATUS` and report what you see

### MCU communication errors
- Not Aurora's fault! This is a CB2/connection issue
- Consider upgrading to CM4 if using BTT CB2---

## License

MIT License - Use it, modify it, share it.

---

*Made with 🍵 and too many late nights.*
