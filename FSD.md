# Functional Specification Document: SAO Social Interaction Game

## 1. Overview

### 1.1 Project Name
**SAO Social** - A badge-based social interaction game using infrared communication.

### 1.2 Purpose
Create a wearable electronic badge (SAO - Simple Add-On format) that encourages face-to-face social interaction at events. Users collect unique "encounters" by exchanging IR signals with other badge holders.

### 1.3 Target Platform
- **MCU**: CH32V003 (RISC-V, 48MHz, 2KB SRAM, 16KB Flash)
- **Form Factor**: SAO v1.69bis compatible

---

## 2. Hardware Components

### 2.1 Microcontroller
- CH32V003F4U6 (TSSOP-20) or CH32V003J4M6 (SOP-8)
- Operating voltage: 3.3V (from SAO header)
- Clock: Internal 24MHz HSI or 48MHz PLL

### 2.2 IR Transmitter
- Standard IR LED (940nm wavelength)
- Driven via GPIO with current limiting resistor
- Modulation: 38kHz carrier frequency

### 2.3 IR Receiver
- TSOP38238 or equivalent (38kHz demodulated output)
- Active-low output signal
- Connected to interrupt-capable GPIO

### 2.4 Status LED
- Single RGB LED (common cathode) or multiple single-color LEDs
- Used for visual feedback and status indication

### 2.5 Buzzer (Optional)
- Small piezo buzzer for audio feedback
- Directly driven from GPIO (no amplifier needed)
- Used for encounter confirmation and level-up notification

### 2.6 Button
- Tactile switch for user input
- Active-low with internal pull-up
- Used for reset and status display

### 2.7 Power
- 3.3V supplied via SAO header
- Target current consumption: <20mA active, <1mA idle

---

## 3. Game Mechanics

### 3.1 Badge Types

#### 3.1.1 Regular Badge (Player)
- Standard participant badge
- Collects and stores encounters
- Displays level progression

#### 3.1.2 Organizer Badge (Admin)
- Special badge for event organizers
- All regular badge functions, plus:
  - **Read encounters**: Request and display encounter list from any regular badge
  - **Set level override**: Temporarily change a badge's displayed level (for prizes/demos)
  - **Identified by**: Special ID range (0xFFxx) or mode flag

### 3.2 Unique Badge Identity
- Each badge has a unique 16-bit ID (stored in flash)
- ID is factory-programmed or generated on first boot from chip UID
- Regular badges: 0x0001 - 0xFEFF
- Organizer badges: 0xFF00 - 0xFFFE (reserved range)

### 3.3 Encounter System
- When two badges come within IR range (~1m), they exchange IDs
- **Mutual confirmation required**: Both badges must complete full handshake
- Each unique encounter is stored in non-volatile memory
- Maximum storage: 128 unique encounters
- Encounters persist across power cycles until user reset

### 3.4 Encounter Levels
| Encounters | Level | LED Pattern |
|------------|-------|-------------|
| 0-4        | 1 - Newcomer | Slow pulse (1Hz) |
| 5-14       | 2 - Social | Double pulse |
| 15-29      | 3 - Networker | Triple pulse |
| 30-49      | 4 - Connector | Fast pulse (2Hz) |
| 50+        | 5 - Legend | Rainbow/special pattern |

### 3.5 Interaction Feedback
- **New encounter**: LED flashes green 3x rapidly + optional beep
- **Already met**: LED flashes yellow 1x
- **Transmission active**: LED dim blue
- **Error/timeout**: LED flashes red
- **Level up**: Special LED animation + optional melody

### 3.6 Level Override (Organizer Feature)
- Organizer can temporarily set any badge to display a different level
- Override is stored in RAM only (resets on power cycle)
- Used for prizes, demos, or special recognition
- Badge shows special "boosted" indicator when overridden

---

## 4. Communication Protocol

### 4.1 Physical Layer
- Carrier: 38kHz IR modulation
- Encoding: NEC-like protocol (modified)
- Bit timing:
  - Logic 0: 562µs pulse + 562µs space
  - Logic 1: 562µs pulse + 1687µs space
  - Start bit: 9ms pulse + 4.5ms space

### 4.2 Packet Structure
```
| Start | Command (8-bit) | Badge ID (16-bit) | Checksum (8-bit) | Stop |
```

### 4.3 Commands

#### Regular Commands
| Command | Value | Description |
|---------|-------|-------------|
| BEACON  | 0x01  | Periodic presence broadcast |
| HELLO   | 0x02  | Initiate encounter handshake |
| ACK     | 0x03  | Acknowledge encounter |
| NACK    | 0x04  | Reject (already met recently) |

#### Organizer Commands (require organizer badge ID)
| Command | Value | Description |
|---------|-------|-------------|
| READ_REQ    | 0x10  | Request encounter list from target badge |
| READ_RESP   | 0x11  | Response with encounter data (chunked) |
| SET_LEVEL   | 0x12  | Temporarily set target badge's display level |
| LEVEL_ACK   | 0x13  | Confirm level override applied |
| LED_CMD     | 0x20  | Remote LED control (broadcast or targeted) |
| LED_ACK     | 0x21  | Confirm LED command received |

### 4.4 Handshake Sequences

#### Regular Encounter Handshake
1. Badge A sends BEACON periodically (every 2-5s, randomized)
2. Badge B receives BEACON, sends HELLO with own ID
3. Badge A receives HELLO, stores encounter, sends ACK with own ID
4. Badge B receives ACK, stores encounter
5. Both badges show "new encounter" feedback

#### Organizer Read Encounters
1. Organizer sends READ_REQ with target badge ID (or broadcast)
2. Target badge responds with READ_RESP containing:
   - Total encounter count
   - Chunk index / total chunks
   - Up to 8 badge IDs per response packet
3. Organizer sends ACK to request next chunk
4. Repeat until all encounters received

#### Organizer Set Level Override
1. Organizer sends SET_LEVEL with target ID and desired level (1-5)
2. Target badge stores override in RAM, sends LEVEL_ACK
3. Target badge displays overridden level until power cycle

#### Organizer Remote LED Control (Optional)
1. Organizer sends LED_CMD with parameters:
   - Target: specific badge ID or 0x0000 for broadcast
   - Mode: predefined pattern or direct RGB value
   - Duration: how long to override (0 = until next command)
2. Target badge(s) execute LED command, send LED_ACK (if targeted)
3. After duration expires, badge returns to normal level display

**LED_CMD Modes:**
| Mode | Value | Effect |
|------|-------|--------|
| OFF      | 0x00 | Turn LED off |
| SOLID    | 0x01 | Solid color (RGB in payload) |
| FLASH    | 0x02 | Flash color (RGB + rate in payload) |
| RAINBOW  | 0x03 | Rainbow cycle |
| PULSE    | 0x04 | Breathing pulse (RGB in payload) |
| RESTORE  | 0xFF | Return to normal operation |

**Use cases:**
- Synchronized light show at event
- Flash all badges for announcements
- Highlight winner/special guest
- "Find me" beacon for lost badge

### 4.5 Collision Avoidance
- Random delay (100-500ms) before responding to beacons
- Exponential backoff on failed transmissions
- Cooldown period after successful encounter (30s)
- **Cooldown bypass**: Double-press button to skip cooldown and re-engage immediately

### 4.6 Organizer Data Output
When organizer reads encounters from a badge, the data can be output via:
- **LED Morse**: Blink encounter count (slow, for quick check)
- **Serial/UART**: If SAO I2C pins repurposed, output full encounter list as text
- **IR relay**: Organizer re-transmits data to a base station with display

---

## 5. User Interface

### 5.1 LED Indicators
- **Idle**: Slow breathing effect at current level color
- **Searching**: Periodic dim flash
- **Transmitting**: Solid dim
- **Receiving**: Brightening pulse
- **Success**: Rapid green flashes
- **Already met**: Single yellow flash
- **Error**: Red flash

### 5.2 Buzzer Feedback (Optional)
- **New encounter**: Short beep
- **Already met**: Two quick low beeps
- **Level up**: Ascending tone sequence
- **Reset confirmed**: Long descending tone

### 5.3 Button

#### Regular Badge
| Action | Function |
|--------|----------|
| Short press | Display encounter count (LED blinks for level, then count) |
| Double press | Force beacon + skip 30s cooldown (re-engage same badge) |
| Long press (3s) | Reset all encounters (hold to confirm, release to cancel) |

#### Organizer Badge
| Action | Function |
|--------|----------|
| Short press | Cycle admin modes: normal → read → set level → LED control |
| Double press | Execute current mode (send read/set/LED command) |
| Long press (3s) | Reset encounters (same as regular)

---

## 6. Memory Map

### 6.1 Flash Layout (16KB)
```
0x0000 - 0x2FFF: Application code (12KB)
0x3000 - 0x3FFF: Encounter storage (4KB = 256 x 16-bit IDs)
```

### 6.2 Encounter Storage
- First 2 bytes: Encounter count
- Remaining: Array of 16-bit badge IDs
- Simple linear search (acceptable for 128 entries)

---

## 7. Power Management

### 7.1 Operating Modes
- **Active**: Full operation, IR TX/RX enabled
- **Idle**: MCU in low-power mode, wake on IR interrupt
- **Sleep**: Deep sleep, wake on timer (periodic beacon)

### 7.2 Power Budget
| Component | Active | Idle |
|-----------|--------|------|
| CH32V003  | 5mA    | 10µA |
| IR LED    | 20mA   | 0mA  |
| IR RX     | 1mA    | 1mA  |
| Status LED| 10mA   | 1mA  |
| **Total** | 36mA   | ~1mA |

---

## 8. Development Phases

### Phase 1: Basic IR Communication
- [ ] IR transmit with 38kHz modulation
- [ ] IR receive and decode
- [ ] Basic packet encoding/decoding

### Phase 2: Game Logic
- [ ] Badge ID generation/storage
- [ ] Encounter storage in flash
- [ ] Handshake protocol implementation
- [ ] Duplicate detection

### Phase 3: User Feedback
- [ ] LED patterns for all states
- [ ] Level progression display
- [ ] Button handling (short/long/double press)
- [ ] Reset functionality with confirmation
- [ ] Optional buzzer support

### Phase 4: Organizer Badge Features
- [ ] Read encounters from target badge
- [ ] Level override command
- [ ] Admin mode switching via button
- [ ] Remote LED control (optional)

### Phase 5: Power Optimization
- [ ] Sleep mode implementation
- [ ] Wake-on-IR functionality
- [ ] Battery life testing

---

## 9. Bill of Materials (Preliminary)

| Component | Part Number | Quantity | Notes |
|-----------|-------------|----------|-------|
| MCU | CH32V003F4U6 | 1 | TSSOP-20 |
| IR LED | TSAL6200 | 1 | 940nm |
| IR Receiver | TSOP38238 | 1 | 38kHz |
| RGB LED | WS2812B-Mini | 1 | Or discrete RGB |
| Tactile Switch | 6x6mm | 1 | Reset/status button |
| Piezo Buzzer | 5mm | 1 | Optional |
| Resistor 100Ω | 0402 | 1 | IR LED current limit |
| Resistor 10kΩ | 0402 | 1 | Pull-up |
| Capacitor 100nF | 0402 | 2 | Decoupling |
| SAO Header | 2x3 1.27mm | 1 | v1.69bis compatible |

---

## 10. Success Criteria

### Regular Badge
1. Two badges can reliably exchange IDs at 0.5-1m distance
2. Encounters persist across power cycles
3. Battery life >8 hours at typical event usage
4. Visual feedback is clear and intuitive
5. No false encounters from ambient IR
6. Button reset clears all encounters with confirmation

### Organizer Badge
7. Can read encounter list from any regular badge within range
8. Can temporarily set display level on target badge
9. Mode switching via button is intuitive

---

## 11. Future Enhancements (Out of Scope)

- Bluetooth bridge for smartphone app (encounter export)
- Achievement system (special badges for milestones)
- Time-limited event modes (encounters only count during event hours)
- Badge "types" with different point values
- NFC backup for encounter exchange
