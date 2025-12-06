# Aurora Lights - Future Python Implementation Notes

## Moonraker Component Structure

When we build the Python version, it would be a Moonraker component like this:

```
aurora-lights/
├── install.sh                    # One-liner installation
├── uninstall.sh
├── moonraker/
│   └── components/
│       └── aurora.py             # Main Moonraker component
├── config/
│   └── aurora_macros.cfg         # Minimal Klipper macros (user commands only)
└── README.md
```

## Installation Script Pattern (Like KAMP)

```bash
#!/bin/bash
# install.sh

REPO_URL="https://github.com/YourName/aurora-lights"
MOONRAKER_COMPONENTS="$HOME/moonraker/moonraker/components"
KLIPPER_CONFIG="$HOME/printer_data/config"

# Clone or update
if [ -d "$HOME/aurora-lights" ]; then
    cd "$HOME/aurora-lights" && git pull
else
    git clone "$REPO_URL" "$HOME/aurora-lights"
fi

# Link component
ln -sf "$HOME/aurora-lights/moonraker/components/aurora.py" "$MOONRAKER_COMPONENTS/"

# Copy config template if not exists
if [ ! -f "$KLIPPER_CONFIG/aurora.cfg" ]; then
    cp "$HOME/aurora-lights/config/aurora_macros.cfg" "$KLIPPER_CONFIG/aurora.cfg"
fi

echo "Add to moonraker.conf:"
echo "[aurora]"
echo "# See documentation for options"
echo ""
echo "Add to printer.cfg:"
echo "[include aurora.cfg]"
```

## moonraker.conf Section Format

```ini
# moonraker.conf

[aurora]
# LED Hardware
group_a_neopixel: toolhead_leds
group_a1_name: emblem
group_a1_range: 1-1
group_a2_name: worklights  
group_a2_range: 2-3

group_b_neopixel: chamber_lights
group_b1_name: left_strip
group_b1_range: 1-18
group_b2_name: right_strip
group_b2_range: 19-36

# Brightness (0.0-1.0)
max_brightness: 0.4

# Timeouts (seconds)
bored_timeout: 300
sleep_timeout: 600

# Event Colors (color or effect:color)
idle_a1: baby_blue
idle_a2: lilac
idle_b1: seafoam
idle_b2: baby_pink

heating_a1: pulse:neon_orange
heating_a2: sunset
heating_b1: portal_orange
heating_b2: neon_yellow

printing_a1: portal_blue
printing_a2: warm_white
printing_b1: ocean
printing_b2: cobalt

# ... etc

[update_manager aurora-lights]
type: git_repo
path: ~/aurora-lights
origin: https://github.com/YourName/aurora-lights.git
managed_services: moonraker
```

## Python Component Skeleton

```python
# aurora.py - Moonraker Component

from __future__ import annotations
import logging
from typing import TYPE_CHECKING, Dict, Any, Optional

if TYPE_CHECKING:
    from moonraker.common import WebRequest
    from moonraker.confighelper import ConfigHelper

COLORS = {
    "red": (1.0, 0.0, 0.0),
    "green": (0.0, 1.0, 0.0),
    "blue": (0.0, 0.0, 1.0),
    # ... all colors
}

class Aurora:
    def __init__(self, config: ConfigHelper) -> None:
        self.server = config.get_server()
        self.klippy_apis = self.server.lookup_component('klippy_apis')
        
        # Load configuration
        self.max_brightness = config.getfloat('max_brightness', 1.0)
        self.bored_timeout = config.getfloat('bored_timeout', 300)
        self.sleep_timeout = config.getfloat('sleep_timeout', 600)
        
        # State
        self.current_event = "idle"
        self.previous_event = "idle"
        
        # Subscribe to printer objects
        self.server.register_event_handler(
            "server:klippy_ready", self._on_klippy_ready
        )
        self.server.register_event_handler(
            "server:status_update", self._on_status_update
        )
        
        # Register API endpoints
        self.server.register_endpoint(
            "/aurora/status", ["GET"], self._handle_status
        )
        
        logging.info("Aurora Lights initialized")
    
    async def _on_klippy_ready(self) -> None:
        """Called when Klippy enters ready state"""
        # Subscribe to objects we care about
        await self.klippy_apis.subscribe_objects({
            "print_stats": ["state", "filename", "progress"],
            "heater_bed": ["temperature", "target"],
            "extruder": ["temperature", "target"],
            "idle_timeout": ["state"]
        })
        await self._trigger_event("idle")
    
    async def _on_status_update(self, status: Dict[str, Any]) -> None:
        """Called when subscribed objects change"""
        # Detect state changes
        event = await self._detect_event(status)
        if event != self.current_event:
            await self._trigger_event(event)
    
    async def _detect_event(self, status: Dict[str, Any]) -> str:
        """Determine current event from printer status"""
        print_stats = status.get("print_stats", {})
        heater_bed = status.get("heater_bed", {})
        extruder = status.get("extruder", {})
        
        state = print_stats.get("state", "standby")
        bed_temp = heater_bed.get("temperature", 0)
        bed_target = heater_bed.get("target", 0)
        
        # Detection logic (same as current Klipper version)
        if bed_target > 0:
            if bed_temp < (bed_target - 2):
                return "heating"
        
        if state == "printing":
            return "printing"
        elif state == "paused":
            return "paused"
        elif state == "complete":
            return "complete"
        
        return "idle"
    
    async def _trigger_event(self, event: str) -> None:
        """Handle event transition"""
        self.previous_event = self.current_event
        self.current_event = event
        
        # Get colors for this event from config
        # Apply to LED groups via G-code
        await self._apply_event_colors(event)
    
    async def _apply_event_colors(self, event: str) -> None:
        """Send LED commands to Klipper"""
        # Build SET_LED commands
        # Send via klippy_apis.run_gcode()
        pass
    
    async def _handle_status(self, web_request: WebRequest) -> Dict[str, Any]:
        """API endpoint: GET /aurora/status"""
        return {
            "current_event": self.current_event,
            "previous_event": self.previous_event,
            # ... more status
        }


def load_component(config: ConfigHelper) -> Aurora:
    return Aurora(config)
```

## What Changes vs Current Implementation

| Feature | Klipper Version | Python Version |
|---------|-----------------|----------------|
| State detection | `delayed_gcode` polling | WebSocket subscription (instant!) |
| AURORA_WAKE needed | Yes, in PRINT_START | No! Auto-detects |
| Effect loops | `delayed_gcode` | Python asyncio |
| Configuration | Variables in .cfg | moonraker.conf section |
| Updates | Manual file copy | `update_manager` git pull |
| User macros | Full macro set | Minimal (just user commands) |

## Migration Path

1. **Keep current Klipper version as v1.x**
   - Simple, works, zero dependencies
   - Good for users who want minimal setup

2. **Build Python version as v2.x**
   - Full Moonraker integration
   - No AURORA_WAKE required
   - Auto-updates via git

3. **Both can coexist**
   - Different installation methods
   - Same color names and concepts
   - User chooses their preference

## Why Not Do Python Now?

1. **Current version works** - heating/printing transitions verified
2. **Learning opportunity** - use v1 to refine the state machine
3. **User feedback** - see what features people actually want
4. **Time investment** - ~20-25 hours for Python rewrite
5. **Klipper version is KISS** - some users prefer no Python
