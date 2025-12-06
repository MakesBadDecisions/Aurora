# Aurora Lights - Architecture Decisions

## Decision Log

### ADR-001: Pure Klipper vs Python Implementation

**Date:** December 2, 2025  
**Status:** Under Consideration  
**Context:** Evaluating implementation approaches for Aurora Lights LED effects system.

---

## The Discovery: How Mainsail/KlipperScreen Stay Updated

### Problem We Solved (Painfully)
Klipper's `delayed_gcode` dies during blocking operations (G28, probing, temperature waits). Our dispatcher couldn't detect state changes during `PRINT_START` because Klipper was busy.

### How We Fixed It (Current Approach)
Added `AURORA_WAKE` macro to `PRINT_START` that:
1. Does inline state detection (checks temps, print_stats)
2. Triggers the appropriate event handler directly
3. Restarts the dispatcher loop

**Trade-off:** Requires 2 lines in user's `PRINT_START` macro.

### How Mainsail/KlipperScreen Do It
They **don't use Klipper macros at all** for state monitoring.

**Architecture:**
```
┌─────────────────┐     WebSocket      ┌─────────────────┐
│   Moonraker     │◄──────────────────►│  KlipperScreen  │
│                 │  notify_status_    │  (Python app)   │
│  - Subscribes   │      update        │                 │
│    to Klipper   │                    │  - Receives     │
│    objects      │                    │    real-time    │
│                 │                    │    updates      │
└────────┬────────┘                    └─────────────────┘
         │
         │ Unix Socket
         ▼
┌─────────────────┐
│     Klipper     │
│  (klippy host)  │
└─────────────────┘
```

**Key API Endpoints:**
- `printer.objects.subscribe` - Subscribe to printer object updates
- `notify_status_update` - Real-time notification when objects change

**Objects They Subscribe To:**
- `print_stats` - state, filename, progress
- `heater_bed` - temperature, target
- `extruder` - temperature, target
- `idle_timeout` - state

**Result:** Instant updates, no polling, survives blocking operations.

---

## Implementation Options

### Option A: Current Pure Klipper (What We Have)

**Pros:**
- ✅ Zero external dependencies
- ✅ Works on any Klipper install
- ✅ No Python knowledge required for users
- ✅ Single `[include Aurora.cfg]` setup
- ✅ Already working!

**Cons:**
- ❌ Requires `AURORA_WAKE` in `PRINT_START`
- ❌ Jinja2 templating is painful for complex logic
- ❌ Limited to what Klipper macros can do
- ❌ No persistent storage (colors reset on restart)

**Best For:** Simple setups, users who don't want complexity

---

### Option B: Python Klipper Extra (Like led_effect)

**Architecture:**
```
┌─────────────────┐
│     Klipper     │
│                 │
│  ┌───────────┐  │
│  │ aurora.py │  │  ◄── Python module in klippy/extras/
│  │           │  │
│  │ - Direct  │  │
│  │   LED     │  │
│  │   control │  │
│  └───────────┘  │
└─────────────────┘
```

**Pros:**
- ✅ Direct Klipper integration
- ✅ Real Python, not Jinja2 hell
- ✅ Access to Klipper's event system
- ✅ Can register own printer objects

**Cons:**
- ❌ Requires Klipper restart to update
- ❌ No moonraker update manager support (unofficial extra)
- ❌ Must be placed in `klippy/extras/` directory
- ❌ More complex installation

**Best For:** Deep Klipper integration, performance-critical effects

---

### Option C: Moonraker Component (Like spoolman, power)

**Architecture:**
```
┌─────────────────┐     WebSocket      ┌─────────────────┐
│   Moonraker     │◄──────────────────►│    Mainsail     │
│                 │                    │                 │
│  ┌───────────┐  │                    └─────────────────┘
│  │aurora.py  │  │
│  │           │  │
│  │- Subscribe│  │     G-code commands
│  │  to print │  │─────────────────────►┌─────────────┐
│  │  stats    │  │                      │   Klipper   │
│  │- Send LED │  │                      │             │
│  │  commands │  │                      │ SET_LED ... │
│  └───────────┘  │                      └─────────────┘
└─────────────────┘
```

**Pros:**
- ✅ Full moonraker update manager support
- ✅ Real Python with async/await
- ✅ WebSocket subscription to Klipper state
- ✅ Can expose own API endpoints
- ✅ Git-based updates like KAMP

**Cons:**
- ❌ More complex than pure Klipper
- ❌ Requires moonraker.conf changes
- ❌ LED commands go through G-code (slight latency)

**Best For:** Full-featured system with auto-updates, API access

---

### Option D: Standalone Python Daemon

**Architecture:**
```
┌─────────────────┐     WebSocket      ┌─────────────────┐
│   Moonraker     │◄──────────────────►│  aurora-daemon  │
│                 │  notify_status_    │  (systemd svc)  │
│                 │      update        │                 │
└─────────────────┘                    │  - Subscribe    │
                                       │  - Send G-code  │
                                       └─────────────────┘
```

**Pros:**
- ✅ Complete independence
- ✅ Can be installed/updated separately
- ✅ Full Python power

**Cons:**
- ❌ Another service to manage
- ❌ More failure points
- ❌ Overkill for LED effects

**Best For:** Complex multi-printer setups

---

## Rewrite Analysis: Klipper → Python

### What Would Change

| Component | Current (Klipper) | Python Version |
|-----------|-------------------|----------------|
| State machine | `_AURORA_STATE` gcode_macro | Python class |
| Dispatcher | `delayed_gcode` loop | Moonraker event subscription |
| Effects | Jinja2 templates | Python functions |
| LED control | `SET_LED` via macro | `printer.gcode.script` API |
| Colors | Dict in gcode_macro | Python dict or JSON file |
| Config | Variables in .cfg | moonraker.conf section |
| Installation | `[include Aurora.cfg]` | `install.sh` + moonraker config |

### What We Could Keep

| Component | Reusable? | Notes |
|-----------|-----------|-------|
| Color definitions | ✅ Yes | Convert dict to Python/JSON |
| Event logic | ✅ Yes | Same state machine, different syntax |
| Effect algorithms | ✅ Yes | Translate Jinja2 → Python |
| Group concept | ✅ Yes | Same A1/A2/B1/B2 structure |
| User config pattern | ✅ Yes | Similar moonraker.conf section |

### Estimated Effort

**Pure Klipper → Moonraker Component:**
- Core state machine: 2-3 hours (straightforward translation)
- Moonraker integration: 4-6 hours (learning curve)
- Event subscription: 2-3 hours
- Effect reimplementation: 3-4 hours
- Installation script: 2-3 hours
- Testing: 4-6 hours
- **Total: ~20-25 hours**

---

## Hybrid Strategy (Recommended)

### Phase 1: Ship Pure Klipper (Current)
- ✅ Already working
- ✅ Validates the concept
- ✅ Gathers user feedback
- ✅ Low barrier to entry

### Phase 2: Build Foundation for Python
While using v1.0, structure code to make Python migration easier:
- Keep effect logic simple and translatable
- Document state machine clearly
- Use consistent naming conventions
- **Prepare `moonraker.conf` section format now**

### Phase 3: Python Moonraker Component (Future)
When ready:
- Full WebSocket subscription (no AURORA_WAKE needed)
- Git-based updates via moonraker
- API endpoints for frontends
- Persistent configuration

---

## Key Insight: The `AURORA_WAKE` Approach is Fine

The "proper" WebSocket approach is elegant, but for LED effects:

1. **We only need state detection at key moments**
   - Print start (when AURORA_WAKE runs)
   - Print end (dispatcher catches this)
   - User commands (explicit)

2. **We're not a real-time monitor**
   - We're not displaying live temps like Mainsail
   - We're just changing LED colors at state transitions
   - A few hundred milliseconds of latency is invisible

3. **The dispatcher handles ongoing monitoring**
   - Once awakened, it polls at update_rate
   - Perfectly adequate for LED effects

**Verdict:** `AURORA_WAKE` in `PRINT_START` is an acceptable trade-off for zero external dependencies.

---

## References

- [Moonraker WebSocket API](https://moonraker.readthedocs.io/en/latest/external_api/printer/#subscribe-to-printer-object-status-updates)
- [Moonraker JSON-RPC Notifications](https://moonraker.readthedocs.io/en/latest/external_api/jsonrpc_notifications/)
- [KlipperScreen WebSocket Implementation](https://github.com/KlipperScreen/KlipperScreen/blob/master/ks_includes/KlippyWebsocket.py)
- [Moonraker Component Development](https://moonraker.readthedocs.io/en/latest/components/)
