# Project Context: SAO Social

## Summary
Social interaction game badge based on CH32V003 microcontroller using IR communication.

## Key Decisions Made

### Game Mechanics
- **Mutual encounters**: Both badges must complete full handshake
- **Button reset**: User can clear encounters with long press (hold to confirm)
- **Cooldown bypass**: Double-press skips 30s cooldown for retry

### Badge Types
1. **Regular Badge**: Player badge, collects encounters, shows level
2. **Organizer Badge**: Admin functions - read encounters, set level override, remote LED control

### Hardware
- CH32V003F4U6 (TSSOP-20)
- IR TX (940nm) + IR RX (TSOP38238, 38kHz)
- RGB LED (WS2812B or discrete)
- Tactile button
- Optional piezo buzzer

### Protocol
- NEC-like IR encoding at 38kHz
- 4-step handshake for encounters
- Organizer commands for read/set level

### Levels
| Encounters | Level |
|------------|-------|
| 0-4 | 1 - Newcomer |
| 5-14 | 2 - Social |
| 15-29 | 3 - Networker |
| 30-49 | 4 - Connector |
| 50+ | 5 - Legend |

## Development Phases
1. Basic IR communication
2. Game logic (ID, storage, handshake)
3. User feedback (LED, button, buzzer)
4. Organizer badge features
5. Power optimization

## Files
- `FSD.md` - Full functional specification document
- `CONTEXT.md` - This file, project context summary
