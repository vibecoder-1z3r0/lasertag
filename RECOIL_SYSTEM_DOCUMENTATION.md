# Recoil Laser Tag System Documentation

> **Sources:**
> - [Recoil Documentation Repository](https://github.com/SkyRocketToys/Recoil_Documentation)
> - [Recoil Hub OpenWRT Repository](https://github.com/SkyRocketToys/Recoil_Hub_OpenWRT_Main)

## Table of Contents

1. [System Overview](#system-overview)
2. [Hardware Architecture](#hardware-architecture)
3. [Infrared (IR) Protocol](#infrared-ir-protocol)
4. [Bluetooth Low Energy (BLE) Protocol](#bluetooth-low-energy-ble-protocol)
5. [Network Hub Architecture](#network-hub-architecture)
6. [Game Application Integration](#game-application-integration)
7. [Data Flow](#data-flow)

---

## System Overview

The Recoil Laser Tag system is a complete wireless gaming platform consisting of four main components:

1. **Recoil Guns** - Hardware devices with IR transceivers and BLE connectivity
2. **Mobile Applications** - Connect to guns via BLE, handle game logic
3. **Network Hub** - OpenWRT-based router coordinating multiplayer gameplay
4. **Game Server** - Unity-based game engine processing events and state

### System Topology

```
┌─────────────┐         BLE          ┌──────────────┐
│  Recoil Gun │◄─────────────────────►│  Mobile App  │
│   (Player)  │                       │   (Player)   │
└──────┬──────┘                       └──────┬───────┘
       │                                     │
       │ IR Shots                           │ WiFi
       │                                     │
       ▼                                     ▼
┌─────────────┐                       ┌──────────────┐
│  Recoil Gun │         WiFi          │ Network Hub  │
│  (Opponent) │◄──────────────────────┤  (Router)    │
└──────┬──────┘                       └──────────────┘
       │                                     │
       │ BLE                                │ TCP/UDP
       │                                     │
       ▼                                     ▼
┌─────────────┐                       ┌──────────────┐
│  Mobile App │         WiFi          │ Game Server  │
│  (Opponent) │◄──────────────────────┤   (Unity)    │
└─────────────┘                       └──────────────┘
```

**Typical Game Flow:**
1. Players connect guns to mobile apps via BLE
2. Apps connect to Network Hub via WiFi
3. Hub assigns unique Client IDs (1-16) and synchronizes game state
4. When Player A shoots Player B:
   - Gun A transmits IR packet containing shooter ID and weapon info
   - Gun B's IR receiver detects the hit
   - Gun B reports hit to App B via BLE telemetry
   - App B sends hit event to Hub via WiFi
   - Hub broadcasts event to all connected apps
   - Game logic processes damage, scores, etc.

---

## Hardware Architecture

### Gun Components

The Recoil Gun is built around the Nordic nRF52832 SoC, providing both Bluetooth Low Energy and a powerful ARM Cortex-M4 processor.

**Main Components:**

| Component | Model/Spec | Function |
|-----------|------------|----------|
| **MCU** | Nordic nRF52832 | ARM Cortex-M4 with BLE 5.0 |
| **IR TX LED (Long Range)** | Vishay TSAL6100 | 940nm, narrow angle |
| **IR TX LED (Short Range)** | Vishay TSAL6100 | 940nm, wide angle |
| **IR Receivers** | Vishay TSOP53338 / Y-lin | 38kHz carrier, 4 sensors |
| **Muzzle LED** | Green LED | Visual firing feedback |
| **Power LED** | White LED | Power indication |
| **Recoil Motor** | DC Motor | Haptic feedback |
| **Battery** | Li-ion/Li-Po | Voltage monitored |

**Gun Models:**
- **Rifle** (GunModel = 1)
- **Pistol** (GunModel = 2)

### IR Receiver Specifications

Source: `Recoil_Protocol_IR.docx` (Recoil_Documentation)

The TSOP53338 IR receivers have specific timing requirements:
- **Carrier frequency:** 38kHz (13.16µs on, 13.16µs off)
- **Minimum burst length:** 6 cycles (158µs)
- **Minimum gap:** 10 cycles (263µs) after burst of 6-35 cycles

**4 Independent Sensors:**
- Sensors 0, 1, 2: Mounted on gun body
- Sensor 3: Clip-on sensor (detachable)

This multi-sensor design enables:
- Directional hit detection
- Increased reliability (redundancy)
- Reduced false negatives from IR noise

### Power Management

The gun monitors battery voltage and adjusts recoil motor timing based on battery level:
- Motor power is directly connected to unregulated VBAT
- Firmware compensates motor run time based on voltage
- Battery voltage reported in telemetry (in millivolts)

**OPT1 vs OPT2 Hardware Variants:**
- **OPT1:** 5V Buck boost design
- **OPT2:** Direct battery drive (cost-saving variant)

Source: `Recoil_Gun_Schematic_REV-4.pdf` (Recoil_Documentation)

---

## Infrared (IR) Protocol

Source: `Recoil_Protocol_IR.docx` (Recoil_Documentation)

### Protocol Overview

The IR protocol uses two encoding methods depending on device type:

| Protocol | Used By | Payload | Encoding |
|----------|---------|---------|----------|
| **MAN20A** | Guns | 20 bits | Manchester |
| **NEC4** | Grenades | 4 bits | NEC (simplified) |

### Encoding Methods

**Manchester Encoding:**
- Bit 0 = "SM" (Space then Mark)
- Bit 1 = "MS" (Mark then Space)
- Mark (M) = 38kHz carrier burst
- Space (S) = no carrier

**NEC Encoding:**
- Bit 0 = "MS" (short space)
- Bit 1 = "MSS" (long space)

### Timing Specifications

**Current Standard (as of June 2017):**
- **Mark/Space duration:** 600µs
- **Carrier frequency:** 38kHz (~15 pulses per mark)
- **Header pulse:** 8 marks, 4 spaces (MMMMMMMMSSSS)

**Historical Note:** Earlier versions used 400µs or 560µs timing, but 600µs provides better range with Y-lin receivers.

### Gun Packet Format (MAN20A)

**20-bit Payload Structure:**

```
Bits: 15 14 13 12 11 10  9  8  7  6  5  4  3  2  1  0
     ┌──────────────────┬─────────────┬────────┬───────┐
     │   Shooter ID (A) │ Weapon (W)  │ Rounds │ Count │
     │     (6 bits)     │  (4 bits)   │  (RRR) │ (CCC) │
     └──────────────────┴─────────────┴────────┴───────┘
```

**Field Descriptions:**

- **A (Bits 15-10):** Shooter ID
  - Valid range: 1-16 for players
  - Default for grenades: 50
  - Allows up to 64 players (6-bit field)

- **W (Bits 9-6):** Weapon ID
  - 0-11: Gun weapons
  - 12-15: Grenades (reserved)

- **RRR (Bits 5-3):** Rounds in shot
  - Used for plasma mode damage calculation
  - Formula: `rounds = (RRR + 1) × 4`
  - Example: RRR=0 → 4 rounds, RRR=7 → 32 rounds

- **CCC (Bits 2-0):** Shot Counter
  - Increments with each shot, wraps around
  - Prevents duplicate hit detection
  - Allows firmware to identify same bullet hitting multiple sensors

**CRC Protection:**
- 4-bit CRC appended to payload (total 24 bits transmitted)

### Grenade Packet Format (NEC4)

**Simplified 4-bit Payload:**

```
Bits: 3  2  1  0
     ┌──────────┐
     │ State (S)│
     │ (4 bits) │
     └──────────┘
```

**Timing:**
- **Header:** 2400µs mark + 2400µs space (MMMMSSSS)
- **Bit 0:** 600µs mark + 600µs space (MS)
- **Bit 1:** 600µs mark + 1200µs space (MSS)
- **Stop bit:** 600µs mark (MSSSSSSS)

### Grenade State Machine

Source: `Recoil_Protocol_IR.docx` lines 143-282

| State Value | State Name | Description | LED | Timeout |
|-------------|------------|-------------|-----|---------|
| 0 | Unarmed | Initial state | Off | - |
| 1 | Cancelled | Button released too early | Off | → Off |
| 2 | Priming | Button held 0.25-1s | Fast blink | - |
| 3 | Primed | Ready to throw | Solid | - |
| 4-13 | Countdown (10-1) | Countdown timer | Slow→Fast | 1s each |
| 14 | Explode | Detonation | Solid | → Off |
| 15 | Waiting | Named kills mode | Solid | → Priming |

**State Logic:**
- Hold button < 0.25s: Cancelled
- Hold button 0.25-1s: Primed (anonymous kill)
- Hold button > 1s: Waiting state (named kills)
- Release during countdown: Cancelled
- Countdown reaches 0: Explode

### Packet Transmission

**Rate Limiting:**
- Maximum rate: 10 packets per second
- Typical pattern: Several packets at 30ms intervals, then 100ms gap
- Prevents IR receiver saturation

**Reliability Features:**
- Packets repeated multiple times (default: 2 repeats)
- Randomized timing between repeats (reduces collision)
- Multiple sensors provide redundancy

### Sensor Independence

Each of the 4 IR sensors processes packets independently:
- Sensor ID included in hit reports
- Same shot detected by multiple sensors is deduplicated using shot counter
- Apps must check shot counter to prevent double-damage from single bullet

---

## Bluetooth Low Energy (BLE) Protocol

Source: `Recoil_Protocol_BLE.docx` (Recoil_Documentation)

### BLE Advertising

**Advertising Parameters:**
- **Period:** 187.5ms
- **Service UUID:** 0x9D10
- **Device Name Format:** `SRG1_<16-char-hex-UUID>`
  - Example: `SRG1_BF7EB8569758B65F`
  - UUID is globally unique hardware identifier

**Bootloader Mode:**
- Device Name: `SRBX_YYYYYY`
  - X = 1 for Rifles, 2 for Pistols
  - YYYYYY = first 6 characters of hex UUID
  - Example: `SRB2_BF7EB8`

### BLE Services

#### 1. Generic Access Service (0x1800)

Standard GATT service required by specification.

**Characteristic: Device Name (0x2A00)**
- Attributes: Read-Write
- Format: ASCII string matching advertising name

#### 2. Device Information Service (0x180A)

**Characteristic: Manufacturer Name String (0x2A29)**
- Attributes: Read
- Recommended for iOS compatibility

#### 3. RecoilGun Service

**Service GUID:** `E6F59D10-8230-4a5c-B22F-C062B1D329E3`

This service contains all gun-specific functionality via 4 characteristics:

---

### Characteristic: ID

**GUID:** `E6F59D11-8230-4a5c-B22F-C062B1D329E3`
**Attributes:** Read
**Size:** 20 bytes

Returns gun identity and firmware information.

**Structure:**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| Version | U16 | 2 | Firmware SVN revision (0 = uncommitted) |
| UUID | U8[8] | 8 | Globally unique hardware identifier |
| GunModel | U8 | 1 | 1=Rifle, 2=Pistol |
| Padding | U8[3] | 3 | Reserved |
| ConfigCRC | U32 | 4 | CRC32 of current configuration |
| BL Version | U16 | 2 | Bootloader SVN revision |

**Important Note:** UUID is a hardware MAC-like identifier, NOT the same as GunID used in telemetry (which is assigned by app).

---

### Characteristic: Telemetry

**GUID:** `E6F59D12-8230-4a5c-B22F-C062B1D329E3`
**Attributes:** Read, Notify
**Size:** 20 bytes

Apps subscribe to this characteristic to receive real-time gun state updates.

**Structure:**

| Offset | Field | Type | Size | Description |
|--------|-------|------|------|-------------|
| 0 | Pkt Cnt | U4 | 0.5 | Packet counter (increments each transmission) |
| 0.5 | Cmd Cnt | U4 | 0.5 | Last received command counter |
| 1 | GunID | U8 | 1 | Gun identifier (1-16 valid, 0 invalid) |
| 2 | Buttons | U8 | 1 | Button state bitmask |
| 3 | Pressed | U4[6] | 3 | Press counters (mod 16) for 6 buttons |
| 6 | Voltage | S16 | 2 | Battery voltage in millivolts |
| 8-9 | IrEvents[0] | Struct | 2 | First IR hit event |
| 10 | IrEvents[0] | U8 | 1 | Event metadata |
| 11-13 | IrEvents[1] | Struct | 3 | Second IR hit event |
| 14 | WeaponAmmo | U8 | 1 | Current ammo count |
| 15 | GunFlags | U8 | 1 | Status flags |
| 16 | Selected Weapon Type | U8 | 1 | Currently active weapon (0-11) |
| 17-19 | Reserved | U8[3] | 3 | Reserved for future use |

**Button Bitmask (Byte 2):**
```
Bit 0 (0x01): Trigger
Bit 1 (0x02): Reload
Bit 2 (0x04): Walkie Talkie
Bit 3 (0x08): Reset
Bit 4 (0x10): Power
Bit 5 (0x20): Recoil Counter
```

**Pressed Counters (Bytes 3-5):**
Six 4-bit counters tracking button press events:
1. Trigger presses
2. Reload presses
3. Walkie Talkie presses
4. Reset presses
5. Power presses
6. Recoil Counter presses

**IrEvents Structure:**

Each IrEvent contains information about a detected IR hit:

```
┌─────────────────────────────────┐
│  IR Payload (U16) - Bytes 0-1   │
│  ┌──────────────────────────┐   │
│  │ Bits 15-10: Shot Counter │   │
│  │ Bits 9-6:   Weapon ID    │   │
│  │ Bits 5-0:   See below    │   │
│  └──────────────────────────┘   │
├─────────────────────────────────┤
│  Event Metadata (U8) - Byte 2   │
│  ┌──────────────────────────┐   │
│  │ Bits 7-4: Event Counter  │   │
│  │ Bits 3-0: Sensor Bitmask │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
```

**Gun Shot (Weapon ID 0-11):**
```
Bits 15-10: Shot counter (C)
Bits 9-6:   Weapon ID (W)
Bits 5-3:   Rounds (R): rounds = (RRR+1) × 4
Bits 2-0:   Shot counter continuation
```

**Grenade (Weapon ID 12-15):**
```
Bits 15-10: Grenade ID (6-bit hash of serial)
Bits 9-6:   Should be 12-15 (grenade marker)
Bits 5-2:   Random counter (J)
Bits 1-0:   State (S) - countdown timer
```

**Sensor Bitmask:**
- Bit 0 (0x1): Sensor 0
- Bit 1 (0x2): Sensor 1
- Bit 2 (0x4): Sensor 2
- Bit 3 (0x8): Sensor 3 (clip-on)
- Value 0: Invalid event

**GunFlags (Byte 15):**
```
Bit 0 (0x01): Reload mode (clip out, cannot fire)
Bit 1 (0x02): Clip-on sensor disconnected
Bits 2-7:     Reserved
```

**Critical Note on Duplicate Detection:**

The shot counter field prevents duplicate damage from:
1. Same bullet hitting multiple sensors on one gun
2. Same bullet being processed multiple times by app

Apps MUST track shot counters to ensure each bullet deals damage only once.

---

### Characteristic: Control

**GUID:** `E6F59D13-8230-4a5c-B22F-C062B1D329E3`
**Attributes:** Read, Write
**Size:** 20 bytes (13 bytes minimum)

Apps write to this characteristic to control gun behavior.

**Structure:**

| Offset | Field | Type | Size | Description |
|--------|-------|------|------|-------------|
| 0 | PktCounter | U4 | 0.5 | Must change for packet to be processed |
| 0.5 | CmdCounter | U4 | 0.5 | Increments to trigger action (prevents duplicates) |
| 1 | IR_ack | U8 | 1 | Sequence of acknowledged IR events |
| 2-3 | Action | U16 | 2 | Action bitmask (see below) |
| 4 | GunID | U8 | 1 | Shooter ID for IR transmission (1-16) |
| 5 | WeaponType | U8 | 1 | Weapon to use for firing (0-11) |
| 6 | WeaponAmmo | U8 | 1 | Ammo count (only if unset reload flag) |
| 7-19 | Reserved | U8[13] | 13 | Optional, reserved for future use |

**Action Bitmask:**
```
0x0000: No action
0x0001: Shoot gun (uses selected weapon type)
0x0002: Set reloading mode (clip out, cannot fire)
0x0004: Unset reloading mode and set ammo (priority over 0x0002)
0x0008: Trigger recoil (500ms duration)
0x0010: Muzzle flash (according to weapon type)
0x0020: Turn power off in 1 second
0x0040: Output stats on UART (debug builds only)
0x0080: Sync (forget outstanding actions and IR events)
0x0100: Reboot to bootloader mode
```

**Important Notes:**

1. **Firmware-level ammo tracking:**
   - Firmware maintains ammo count
   - Gun won't fire when ammo reaches 0
   - App sets ammo count via 0x0004 action
   - Firmware auto-decrements on trigger press

2. **BLE-triggered shots bypass ammo:**
   - 0x0001 action shoots regardless of ammo count
   - Ammo is NOT decremented
   - App responsible for ammo logic when using BLE shooting

3. **PktCounter vs CmdCounter:**
   - PktCounter must differ from previous packet
   - CmdCounter increments only when new action desired
   - Allows retransmission without duplicate execution

---

### Characteristic: Config

**GUID:** `E6F59D14-8230-4a5c-B22F-C062B1D329E3`
**Attributes:** Write
**Size:** Variable (TLV format)

Apps write configuration changes using Tag-Length-Value encoding.

**TLV Structure:**

| Field | Type | Size | Description |
|-------|------|------|-------------|
| Tag | U16 | 2 | Configuration parameter ID |
| Length | U8 | 1 | Value length in bytes |
| Value | U8[Length] | N | Parameter value |

**When to Update Config:**
- App reads ConfigCRC from ID characteristic
- If CRC doesn't match app's expected config, send updates
- Allows dynamic weapon behavior tuning without firmware updates

### Configuration Table

Source: `Recoil_Gun_Firmware_Config_Guide.docx` (Recoil_Documentation)

#### Weapon Definitions (IDs 0-11)

Each weapon type has 9 bytes of configuration:

**Tag IDs:** 0-11 (one per weapon)

| Field | Type | Bits | Description |
|-------|------|------|-------------|
| TriggerMode | U8 | 8 | Firing behavior (see below) |
| RateOfFire | U8 | 8 | Fire rate in 50ms units |
| PowerIR1 | U8 | 8 | Long-range IR LED power (0-255) |
| PowerIR2 | U8 | 8 | Short-range IR LED power (0-255) |
| PowerLED1 | U8 | 8 | Muzzle LED brightness (0-255) |
| PowerLED2 | U8 | 8 | Power LED brightness (0-255, debug) |
| PowerMotor | U8 | 8 | Recoil duration in 5ms units |
| FlashLED1 | U4 | 4 | Muzzle LED flash mode |
| FlashLED2 | U4 | 4 | Power LED flash mode (debug) |
| FlashParam1 | U4 | 4 | Flash parameter (mode-dependent) |
| FlashParam2 | U4 | 4 | Flash parameter (mode-dependent) |

**Trigger Modes:**

| Value | Mode | Behavior |
|-------|------|----------|
| 0 | Plasma | Charge while held, fire on release |
| 1 | Single | Fire once on press |
| 2-253 | N-Burst | Fire N shots while held |
| 254 | Full Auto | Fire continuously while held |
| 255 | Reserved | - |

**Trigger Mode Details:**

*Plasma Mode (0):*
- Hold trigger to charge (accumulates 4 rounds per rate-of-fire period)
- Release trigger to fire single plasma shot worth accumulated rounds
- If ammo depleted while charging, wait for release without adding rounds
- Example: 1s rate, hold 2.5s = instant 4 rounds + 1s later 4 more = 8-round shot

*Single Shot (1):*
- Fire 1 round on trigger press
- No additional shots until trigger released and pressed again

*N-Burst (2-253):*
- Fire 1 round on trigger press
- Continue firing (1 round per rate-of-fire period) while held
- Stop when: trigger released, N shots fired, or ammo depleted

*Full Auto (254):*
- Fire 1 round on trigger press
- Continue firing (1 round per rate-of-fire period) while held
- Stop when: trigger released or ammo depleted

**Rate of Fire:**
- Units: 50ms
- Value 10 = 500ms = 2 shots/second
- Value 20 = 1000ms = 1 shot/second

**IR Power Levels:**
- Scale: 0-255 (interface convenience)
- Internal: Scaled to 0-18 (18 = 100% duty cycle)
- Value 25 → 25×18/255 = 1.76 → 1 (5.5% actual duty)
- Value 255 → 255×18/255 = 18 (100% duty cycle)

**Motor Power:**
- Units: 5ms
- Value adjusted based on battery voltage
- Default: 18 (90ms for full spring load at full charge)

**Flash Modes:**

| Value | Mode | Description |
|-------|------|-------------|
| 0 | None | LED off during fire |
| 1 | Square Wave | N flashes of T ms duration |
| 2 | Glow | Accelerating glow while held, square wave on release |
| 3 | Solid | On while trigger pressed |

*Square Wave Parameters:*
- FlashParam1: Number of flashes
- FlashParam2: Flash duration in 100ms units

*Glow Parameters:*
- FlashParam1: Number of flashes on release
- FlashParam2: Initial glow period (500ms units) + release flash duration (100ms units)
- Glow frequency doubles every rate-of-fire period

Example: FlashParam2=4 with Glow mode
- Initial glow: 2000ms period (0.5 Hz)
- After 1 rate-of-fire: 1000ms period (1 Hz)
- After 2 rate-of-fire: 500ms period (2 Hz)
- On release: Square wave at 400ms period (2.5 Hz)

#### Global Parameters

**ShotConfig (Tag 16):**

2 bytes controlling global shooting behavior.

| Field | Type | Bits | Description |
|-------|------|------|-------------|
| Auto Feedback | U4 | 4 | Automatic feedback bitmask |
| Unused | U4 | 4 | - |
| TriggerMode Override | U8 | 8 | Global trigger mode (255=use weapon) |

*Auto Feedback Bitmask:*
```
Bit 0 (0x1): Auto recoil on shooting (default ON)
Bit 1 (0x2): Auto muzzle flash on shooting (default ON)
Bit 2 (0x4): Auto power LED flash when hit (default OFF, debug only)
```

*TriggerMode Override:*
- Value 0-254: Override weapon-defined trigger mode
- Value 255: Use weapon's trigger mode (default)
- Allows app to change trigger behavior without reconfiguring weapons

**IRConfig (Tag 17):**

3 bytes controlling IR transmission and reception.

| Field | Type | Bits | Description |
|-------|------|------|-------------|
| TX Repeats | U4 | 4 | Packet repeat count (default 2) |
| TX Flags | U4 | 4 | Transmission flags |
| RX Enable | U8 | 8 | Sensor enable bitmask (default 0xF) |
| Clip-On Check Interval | U8 | 8 | Test period in 500ms units |

*TX Repeats:*
- Default: 2 (optimized for 10Hz max rate, 2 simultaneous shooters)
- Higher values increase reliability but can cause packet overlap
- Total transmission time must be less than rate-of-fire period

*TX Flags:*
```
Bit 0 (0x1): Randomize TX times (default ON)
```
- Randomizes delay between repeats
- Reduces collision when multiple players shoot simultaneously

*RX Enable:*
```
Bit 0 (0x01): Enable sensor 0
Bit 1 (0x02): Enable sensor 1
Bit 2 (0x04): Enable sensor 2
Bit 3 (0x08): Enable sensor 3 (clip-on)
```

*Clip-On Check Interval:*
- Period between presence tests (500ms units)
- Default: 6 (3000ms = 3 seconds)
- Absence declared after 3 consecutive failed tests
- Each test disables sensor briefly (may miss shots if too frequent)

#### Configuration Workflow

Source: `Recoil_Gun_Firmware_Config_Guide.docx` lines 48-53

**3-Phase Development Process:**

1. **Firmware Design:**
   - Set tentative default config parameters in firmware

2. **Integration & Testing:**
   - Dynamically alter config via BLE during testing
   - Rapid iteration without firmware rebuild/flash
   - Tune weapon feel, balance, etc.

3. **Manufacturing:**
   - Finalize tested parameters as firmware defaults
   - Dynamic config still available but not required

**Example Configuration Code:**

Source: `Recoil_Gun_Firmware_Config_Guide.docx` lines 193-356

```csharp
// Weapon 0: Single Shot
weapon[0].id = 0;
weapon[0].triggerMode = SINGLE;
weapon[0].rateOfFire = rateOfFire_ms(1000);  // 1 shot/sec
weapon[0].powerIR1 = 255;  // 100% long range
weapon[0].powerIR2 = 25;   // ~5% short range
weapon[0].powerLED1 = 255;
weapon[0].powerMotor = 255;
weapon[0].flashLED1 = SQUARE_WAVE;
weapon[0].flashParam1 = 1;  // 1 flash
weapon[0].flashParam2 = squareWavePeriod_ms(300);

// Weapon 3: Plasma
weapon[3].id = 3;
weapon[3].triggerMode = PLASMA;
weapon[3].rateOfFire = rateOfFire_ms(2000);  // Charge every 2s
weapon[3].powerIR1 = 255;
weapon[3].powerIR2 = 25;
weapon[3].powerLED1 = 255;
weapon[3].flashLED1 = GLOW;
weapon[3].flashParam1 = 15;  // 15 flashes on release
weapon[3].flashParam2 = glowPeriod_ms(2000);
```

---

## Network Hub Architecture

Source: `recoilnetwork.h` (Recoil_Hub_OpenWRT_Main)

The Recoil Hub is a game server running on an OpenWRT-based wireless router (AR9331 chipset), coordinating multiplayer gameplay between up to 16 clients.

### System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     RECOIL HUB                          │
│                 (OpenWRT Router/AP)                     │
│                                                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │         Multi-threaded C Network Server           │ │
│  │                                                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │ │
│  │  │Discovery │  │   Sync   │  │   Ping   │       │ │
│  │  │TCP:50000 │  │TCP:50001 │  │UDP:50002 │       │ │
│  │  └──────────┘  └──────────┘  └──────────┘       │ │
│  │                                                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │ │
│  │  │   Time   │  │  Event   │  │ Upgrade  │       │ │
│  │  │UDP:50003 │  │TCP:50004 │  │TCP:50005 │       │ │
│  │  └──────────┘  └──────────┘  └──────────┘       │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│          WiFi Access Point (10.10.10.x)                 │
└─────────────────────────────────────────────────────────┘
         │           │           │
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌────────┐
    │ Client │  │ Client │  │ Client │
    │  ID=1  │  │  ID=2  │  │  ID=3  │
    └────────┘  └────────┘  └────────┘
```

**Key Specifications:**
- **Max Clients:** 16 simultaneous connections
- **Client ID Range:** 1-16
- **Product ID:** 1
- **Protocol Version:** 1
- **System Version:** v1.0-1

### Protocol Header

All packets use a 4-byte header with bit-packed fields:

```
 31        24 23      18 17      12 11         0
┌────────────┬──────────┬──────────┬────────────┐
│ Product ID │ Protocol │ Protocol │   Payload  │
│  (8 bits)  │  Version │    ID    │   Length   │
│            │ (6 bits) │ (6 bits) │ (12 bits)  │
└────────────┴──────────┴──────────┴────────────┘
```

**Field Definitions:**
- **Product ID:** Always 1 for Recoil
- **Protocol Version:** Always 1 for current spec
- **Protocol ID:** Service type (0-26, see below)
- **Payload Length:** Bytes following header (0-4095)

**Byte Order:**
- Little-endian on clients (Unity/Windows/Android)
- Big-endian on hub (AR9331 MIPS)
- Build flag `ENABLE_BIG_ENDIAN_BUILD` handles conversion

### Network Services

Source: `recoilnetwork.h` lines 210-238

#### Protocol ID Assignments

| ID | Service | Description |
|----|---------|-------------|
| 0 | NetworkDiscovery | Client join/leave, ID assignment |
| 1 | NetworkTime | Time synchronization |
| 2 | NetworkPing | Latency measurement |
| 3 | BS_Password | Base station password reset |
| 4 | BS_SoftwareUpdate | OTA firmware update |
| 5 | BS_Error | Error reporting |
| 6-10 | Gen_Reserved | Reserved |
| 11 | ServerDataSync | Server→Client game state sync |
| 12 | ServerAudio | Server→Client audio |
| 13 | ServerGameSetup | Server→Client game setup |
| 14 | ServerBroadcast | Server→All clients broadcast |
| 15 | ServerTeamcast | Server→Team broadcast |
| 16 | ServerGameMap | Server→Client map data |
| 17-20 | ServerReserved | Reserved |
| 21 | ClientDataSync | Client→Server state updates |
| 22 | ClientAudio | Client→Server voice |
| 23-25 | ClientReserved | Reserved |
| 26 | GameData | Bidirectional game events |

### Port Configuration

**Production Ports (ENABLE_PRODUCTION):**
```
Discovery:      50000 (TCP)
Sync:           50001 (TCP)
Ping:           50002 (UDP)
Time:           50003 (UDP)
Event:          50004 (TCP)
Upgrade:        50005 (TCP)
Ping Return:    50007 (UDP)
Time Return:    50008 (UDP)
```

**Debug/Test Ports:**
```
Discovery:      60000 (TCP)
Sync:           60001 (TCP)
Ping:           60002 (UDP)
Time:           60003 (UDP)
Event:          60004 (TCP)
Upgrade:        60005 (TCP)
Ping Return:    60007 (UDP)
Time Return:    60008 (UDP)
```

---

## Service Details

### 1. Network Discovery Service

**Port:** 50000 (TCP)
**Protocol ID:** 0

Handles client lifecycle: joining, ID assignment, reconnection, and leaving.

#### Packet Types

Source: `recoilnetwork.h` lines 366-388

| Type | Name | Direction | Description |
|------|------|-----------|-------------|
| 1 | JOIN | C→S | Initial connection request |
| 2 | WELCOME | S→C | Connection accepted, ID assigned |
| 3 | REJOIN | C→S | Reconnect with previous ID |
| 4 | NWCLIENTS_REQUEST | C→S | Request list of active clients |
| 5 | NWCLIENTS | S→C | List of active client IDs |
| 6 | NWCLIENT_ADD | S→All | Broadcast: new client joined |
| 7 | NWCLIENT_REMOVE | S→All | Broadcast: client left |
| 8 | GOODBYE | C→S | Graceful disconnect |
| 9 | NWSERVER_VERSION | C→S | Request server version |
| 10 | NWSERVER_VERSION_INFO | S→C | Server version response |

**JOIN Packet (Client→Server):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ Type: 1 (JOIN)                       │
│ ClientId: 0 or previous ID           │
│ Reserved: 0                          │
└──────────────────────────────────────┘
```

**WELCOME Packet (Server→Client):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ Type: 2 (WELCOME)                    │
│ ClientId: Assigned ID (1-16)         │
└──────────────────────────────────────┘
```

**Client State Tracking:**

Source: `recoilnetwork.h` lines 407-413

```c
typedef struct {
    uint8_t  IP[16];        // "xxx.xxx.xxx.xxx\0"
    uint8_t  mac[6];        // MAC address
    bool     connected;     // Connection status
    uint32_t clientId;      // Unique ID (1-16)
} DiscoveryClientInfo_t;
```

**ID Assignment Logic:**
- Server maintains 16 client slots
- On JOIN, server finds lowest available ID (1-16)
- On REJOIN, server attempts to reassign previous ID
- If previous ID unavailable, assigns new ID
- NWCLIENT_ADD broadcast informs all clients of new player

### 2. Network Sync Service

**Port:** 50001 (TCP)
**Protocol ID:** 11 (ServerDataSync) / 21 (ClientDataSync)

Synchronizes game state across all connected clients.

#### Client→Server Sync

Source: `recoilnetwork.h` lines 328-340

**ClientDataSync Packet:**
```
┌──────────────────────────────────────┐
│ Header (4 bytes, Protocol ID=21)     │
├──────────────────────────────────────┤
│ clientid (U32)                       │
│ syncData0 (U32)                      │
│ syncData1 (U32)                      │
│ syncData2 (U32)                      │
│ syncData3 (U32)                      │
│ syncData4 (U32)                      │
│ syncData5 (U32)                      │
│ syncData6 (U32)                      │
│ syncData7 (U32)                      │
└──────────────────────────────────────┘
Total: 40 bytes (4 header + 36 data)
```

Each client sends 8×32-bit words of game state data.

#### Server→All Sync

**ServerDataSync Packet:**
```
┌──────────────────────────────────────┐
│ Header (4 bytes, Protocol ID=11)     │
├──────────────────────────────────────┤
│ NoOfClients (U32)                    │
├──────────────────────────────────────┤
│ Client[0] syncClientData_t (36 bytes)│
│ Client[1] syncClientData_t (36 bytes)│
│ ...                                  │
│ Client[N-1] syncClientData_t         │
└──────────────────────────────────────┘
```

Server aggregates all client states and broadcasts to everyone, enabling peer state awareness.

### 3. Network Ping Service

**Port:** 50002 (UDP)
**Protocol ID:** 2

Measures network latency using echo request/reply.

**Ping Request (Client→Server):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ type: 1 (REQUEST)                    │
│ sequence: U8                         │
│ t_secs: U16 (client time)            │
│ t_nsecs: U32 (client time)           │
└──────────────────────────────────────┘
Total: 12 bytes
```

**Ping Reply (Server→Client):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ type: 2 (REPLY)                      │
│ sequence: U8 (echo from request)     │
│ t_secs: U16 (server time)            │
│ t_nsecs: U32 (server time)           │
└──────────────────────────────────────┘
Total: 12 bytes
```

**Client-side Latency Calculation:**
```
RTT = (receive_time - send_time)
Latency = RTT / 2
```

Server also uses this to monitor client connection health.

### 4. Network Time Service

**Port:** 50003 (UDP)
**Protocol ID:** 1

Provides synchronized game time to all clients.

**Time Request (Client→Server):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ id: Client ID                        │
│ type: 1 (REQUEST)                    │
│ sequence: U8                         │
└──────────────────────────────────────┘
Total: 7 bytes
```

**Time Reply (Server→Client):**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ sequence: U8 (must match request)    │
│ type: 2 (REPLY)                      │
│ t_secs: U16 (seconds)                │
│ t_nsecs: U32 (nanoseconds)           │
└──────────────────────────────────────┘
Total: 12 bytes
```

**Network Time Format:**
- **Seconds:** 16-bit unsigned (0-65535)
- **Nanoseconds:** 32-bit unsigned
- Wraps after ~18.2 hours

Clients use this for synchronized events (countdown timers, timed objectives, etc.).

### 5. Network Event Service

**Port:** 50004 (TCP)
**Protocol ID:** 26 (GameData)

Bidirectional game event communication.

**Maximum Packet Size:** 1024 bytes
- 4-byte header
- 1020-byte payload

**Event Flow:**
1. Client detects local event (gun fired, hit received, etc.)
2. Client sends event to server via TCP
3. Server validates event
4. Server broadcasts event to relevant clients (or all clients)
5. Clients update game state

**Example Event Types:**
- Player fired weapon
- Player was hit
- Player died
- Player respawned
- Player picked up item
- Flag captured
- Objective completed

Events are application-defined; hub acts as reliable message router.

### 6. Upgrade Service

**Port:** 50005 (TCP)
**Protocol ID:** 4

Provides Over-The-Air (OTA) firmware/software updates.

#### Upgrade Packet Types

Source: `recoilnetwork.h` lines 379-387

| Type | Name | Direction | Description |
|------|------|-----------|-------------|
| 11 | NWSERVER_UPGRADE | C→S | Initiate upgrade |
| 12 | NWSERVER_UPGRADE_ACK | S→C | Upgrade accepted |
| 13 | NWSERVER_UPGRADE_FAILED | S→C | Upgrade rejected |
| 14 | NWSERVER_UPGRADE_PAYLOAD | C→S | File chunk transfer |
| 15 | NWSERVER_UPGRADE_PAYLOAD_ACK | S→C | Chunk acknowledged |
| 16 | NWSERVER_UPGRADE_UPLOAD_COMPLETE | C→S | All chunks sent |
| 17 | NWSERVER_UPGRADE_COMPLETE | S→C | Install successful |
| 18 | NWSERVER_UPGRADE_SHUTDOWN | S→C | Rebooting |

#### OTA File Structure

Source: `recoilnetwork.h` lines 520-563

```c
typedef struct {
    uint8_t  client;                 // Client ID
    uint8_t  type;                   // firmware or package
    uint16_t next_block;             // Expected block number
    uint32_t bcount;                 // Total blocks
    uint32_t bsize;                  // Block size
    uint32_t received;               // Bytes received
    file_info_t file;                // File metadata
    compressed compress;             // Compression flag
    file_info_t ufile;              // Uncompressed file info
} OTA_Info_t;

typedef struct {
    char     filename[128];
    uint32_t filesize;
    char     path[128];
    char     md5sum[32];            // MD5 checksum
    algorithm algo;                  // lzma, etc.
} file_info_t;
```

**Update Types:**
- **firmware:** OpenWRT system firmware
- **package:** IPK package (e.g., recoil.ipk)

**Compression Support:**
- LZMA compression
- Server decompresses before installation

**Payload Transfer:**
```
┌──────────────────────────────────────┐
│ Header (4 bytes)                     │
├──────────────────────────────────────┤
│ type: 14 (PAYLOAD)                   │
│ crc_flag: 0 or 1                     │
│ num: Block number                    │
│ crc32: Block CRC32                   │
│ size: Payload bytes in this block    │
├──────────────────────────────────────┤
│ Binary data (size bytes)             │
└──────────────────────────────────────┘
```

**Failure Reasons:**

Source: `recoilnetwork.h` lines 479-490

```
0: None
1: Upgrade already in progress
2: Invalid parameters
3: No upgrade available
4: System error
5: Transfer error
6: MD5 checksum mismatch
7: Installation error
8: Decompression error
```

#### Upgrade Workflow

1. Client sends NWSERVER_UPGRADE with file metadata (name, size, MD5)
2. Server validates and responds ACK or FAILED
3. Client sends file in blocks via PAYLOAD packets
4. Server acknowledges each block
5. On last block, client sends UPLOAD_COMPLETE
6. Server verifies MD5, installs package
7. Server sends COMPLETE on success
8. Server sends SHUTDOWN and reboots

---

## Firmware Security and Signing

### Critical Discovery: Public Private Key

**IMPORTANT:** The private key used for firmware signing is publicly available in the Recoil Documentation repository as `priv.pem`:

```
File: https://github.com/SkyRocketToys/Recoil_Documentation/blob/master/priv.pem
Type: EC (Elliptic Curve) Private Key
Format: PEM encoded
```

### Security Implications

**From a Security Standpoint:**

⚠️ **Critical Vulnerabilities:**
- Anyone with access to this key can sign firmware that guns will accept as legitimate
- No authentication barrier exists for OTA firmware updates
- Malicious actors could potentially create harmful firmware
- The key cannot be revoked without physical hardware modification
- All Recoil guns worldwide trust this single compromised key

**From an Open Source Standpoint:**

✅ **Community Benefits:**
- Community firmware development is fully possible without reverse engineering
- No proprietary signing process required
- Long-term maintenance can continue even after manufacturer EOL
- Educational and research projects have full hardware access
- Hobbyists can customize gun behavior extensively
- Transparent security model (no security through obscurity)

### Gun Firmware Update Process

Source: `Recoil_Gun_Firmware_Upgrade_Guide.docx`

**DFU (Device Firmware Update) Protocol:**

The guns use Nordic Semiconductor's standard DFU protocol over BLE:

1. **Enter DFU Mode:**
   - App sends Control command `0x0100` (Reboot to bootloader)
   - Gun disconnects, reboots
   - Bootloader starts
   - Muzzle LED turns ON (visual indicator)
   - Power LED turns OFF
   - Gun advertises as `SRB1_XXXXXX` or `SRB2_XXXXXX`

2. **Bootloader Verification Checks:**
   - ✅ Image fits in flash memory
   - ✅ Metadata specifies correct hardware/softdevice/bootloader versions
   - ✅ Image signature is valid (verified against `priv.pem` public key)
   - ✅ CRC32 checksum matches received data

3. **Update Process:**
   - If all checks pass: Flash new firmware
   - If any check fails: Keep old firmware, discard new image
   - If old firmware corrupted: Boot to DFU mode, wait for valid image

4. **Recovery:**
   - Bootloader is separate from main firmware
   - Even with corrupted main firmware, bootloader still functions
   - Gun will not become "bricked" - always recoverable via DFU

### Creating Custom Firmware

With the public private key, the community can:

1. **Develop Custom Firmware:**
   ```bash
   # Clone Nordic SDK
   git clone https://github.com/NordicSemiconductor/nRF5-SDK

   # Build custom firmware
   make

   # Sign with public key (priv.pem)
   nrfutil pkg generate --hw-version 52 \
     --sd-req 0x00A9 \
     --application-version 1 \
     --application app.hex \
     --key-file priv.pem \
     firmware.zip
   ```

2. **Deploy via Mobile App:**
   ```dart
   // Using Nordic's DFU library
   final dfuUpdate = DfuUpdate(
     'firmware.zip',
     deviceId: gun.id,
   );

   await dfuUpdate.start();
   ```

3. **Use Reference Implementations:**
   - **iOS:** nRF Toolbox (App Store)
   - **Android:** nRF Connect (Play Store)
   - **Desktop:** nRF Connect Desktop
   - **Source:** https://github.com/NordicSemiconductor

### Recommendations

**For DIY/Community Projects:**
- ✅ Use the provided `priv.pem` for compatibility with existing guns
- ✅ Document any firmware modifications thoroughly
- ✅ Test extensively before deploying to avoid bricking
- ✅ Maintain backward compatibility with stock firmware when possible

**For Commercial/Security-Critical Deployments:**
- ⚠️ Generate new key pair via JTAG/SWD programmer
- ⚠️ Flash new bootloader with different public key
- ⚠️ Implement additional security measures (encrypted BLE, etc.)
- ⚠️ Consider this a known vulnerability of the stock system

---

## Game Application Integration

Source: `UnityGameNetworkServer/` (Recoil_Hub_OpenWRT_Main)

### Game Events

The Unity-based game client implements an event-driven architecture.

**Key Events:**

| Event | Description |
|-------|-------------|
| `ReceivePlayerConnect` | Player joined lobby |
| `ReceivePlayerDisconnect` | Player left/timeout |
| `ReceiveSetGameMode` | Game mode selected |
| `ReceiveStart` | Game countdown started |
| `ReceiveFire` | Player fired weapon |
| `ReceivePlayerWasHit` | Local player hit |
| `ReceiveOtherPlayerWasHit` | Remote player hit |
| `ReceivePlayerDeath` | Player eliminated |
| `ReceiveRespawnPlayer` | Player respawned |
| `ReceiveReloadClip` | Player reloading |
| `ReceiveUpdateGameState` | Score/time update |
| `ReceiveGameOver` | Match ended |

### Player State Synchronization

Each player maintains state synchronized across all clients:

```cpp
class GamePlayerState {
    int playerId;           // 1-16
    int teamId;             // Team assignment
    int health;             // Current HP
    int ammo;               // Current ammo
    int score;              // Player score
    Vector3 position;       // World position (for map)
    bool isDead;            // Alive status
    int kills;              // Kill count
    int deaths;             // Death count
};
```

State updates sent via Sync service, events via Event service.

### Game Modes

Typical game modes supported:
- **Free-for-All:** Every player for themselves
- **Team Deathmatch:** Team vs team elimination
- **Capture the Flag:** Team objective
- **King of the Hill:** Control point holding
- **Infection:** Tag-style elimination

---

## Data Flow

### Complete Hit Detection Flow

```
┌──────────────┐
│  Player A    │
│  Gun Trigger │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────────┐
│ Gun A Firmware                           │
│ 1. Check ammo > 0                        │
│ 2. Decrement ammo                        │
│ 3. Generate IR packet:                   │
│    - Shooter ID: A's GunID               │
│    - Weapon ID: Current weapon           │
│    - Shot Counter: ++counter             │
│    - Rounds: Based on weapon config      │
│ 4. Transmit IR (repeat 2x)               │
│ 5. Trigger recoil motor                  │
│ 6. Flash muzzle LED                      │
└──────────────┬───────────────────────────┘
               │
               │ Infrared (38kHz, 940nm)
               │
               ▼
┌──────────────────────────────────────────┐
│ Gun B IR Receivers (4 sensors)           │
│ 1. Sensors 0,1,3 detect IR packet        │
│ 2. Decode Manchester bits                │
│ 3. Verify CRC                            │
│ 4. Extract: ShooterID=A, WeaponID=W,     │
│            ShotCounter=C, Rounds=R       │
│ 5. Check shot counter not duplicate      │
│ 6. Store in IrEvents[0], IrEvents[1]     │
└──────────────┬───────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────┐
│ Gun B Firmware Telemetry (BLE Notify)    │
│ Packet includes:                         │
│ - GunID: B                               │
│ - Buttons: Current state                 │
│ - Voltage: Battery mV                    │
│ - IrEvents[0]:                           │
│   - Payload: A's packet                  │
│   - Sensor: 0x9 (sensors 0,3)            │
│ - IrEvents[1]:                           │
│   - Payload: A's packet                  │
│   - Sensor: 0x2 (sensor 1)               │
└──────────────┬───────────────────────────┘
               │
               │ BLE Notify
               │
               ▼
┌──────────────────────────────────────────┐
│ App B (Mobile)                           │
│ 1. Receive telemetry notification        │
│ 2. Parse IrEvents[]                      │
│ 3. Deduplicate using shot counter        │
│    (3 sensors = 1 hit, counter C)        │
│ 4. Calculate damage from rounds (R)      │
│ 5. Update local player health            │
│ 6. Create hit event packet               │
└──────────────┬───────────────────────────┘
               │
               │ WiFi (TCP 50004)
               │
               ▼
┌──────────────────────────────────────────┐
│ Network Hub (Router)                     │
│ 1. Receive hit event from App B          │
│ 2. Validate: ClientID=B, ShooterID=A     │
│ 3. Broadcast event to all clients        │
└──────────────┬───────────────────────────┘
               │
               ├──────────────┬─────────────┐
               │              │             │
               ▼              ▼             ▼
         ┌─────────┐    ┌─────────┐   ┌─────────┐
         │  App A  │    │  App B  │   │  App C  │
         └─────────┘    └─────────┘   └─────────┘
              │              │             │
              ▼              ▼             ▼
       Update UI:      Update UI:    Update UI:
       +1 kill         -HP, hit      See B hit
       +score          flash/sound   by A
```

### Detailed Packet Examples

**IR Packet from Gun A:**
```
ShooterID (GunID): 3
WeaponID: 5 (Assault Rifle)
Shot Counter: 42
Rounds: 1 (single shot)

Binary encoding (20 bits):
Bits 15-10: 000011 (shooter=3)
Bits 9-6:   0101   (weapon=5)
Bits 5-3:   000    (rounds=(0+1)×4=4)
Bits 2-0:   010    (counter=42 mod 8 = 2)

Full packet: 0x0D42 (hex)
Plus 4-bit CRC: 0x0D426 (24 bits total)

Manchester encoded:
Header: MMMMMMMMSSSS (8M, 4S)
Data bits: (each bit is SM or MS)
CRC bits: (4 bits)
Total transmission: ~14.4ms at 600µs/bit
```

**BLE Telemetry from Gun B:**
```
Offset 00: Pkt=5, Cmd=2
Offset 01: GunID=7
Offset 02: Buttons=0x00 (none pressed)
Offset 03-05: Pressed counters=...
Offset 06-07: Voltage=3850 (3.85V)
Offset 08-10: IrEvents[0]
  - Payload (U16): 0x0D42
    - Shooter: 3
    - Weapon: 5
    - Rounds: 4
    - Counter: 42
  - Metadata (U8): 0x39
    - Event counter: 3
    - Sensors: 0x9 (sensors 0 and 3)
Offset 11-13: IrEvents[1]
  - Payload: 0x0D42 (same shot)
  - Metadata: 0x42
    - Event counter: 4
    - Sensors: 0x2 (sensor 1)
Offset 14: WeaponAmmo=25
Offset 15: GunFlags=0x00
Offset 16: Selected Weapon=2
```

**App B Hit Event to Hub:**
```
Header:
  Product ID: 1
  Protocol Version: 1
  Protocol ID: 26 (GameData)
  Payload Length: 16

Payload:
  EventType: PlayerHit
  VictimID: 7 (B's client ID)
  ShooterID: 3 (A's client ID)
  WeaponID: 5
  Damage: 4 (from rounds field)
  Timestamp: 12345 (network time)
  Reserved: 0
```

**Hub Broadcast to All Clients:**
```
Same packet sent to all 16 client connections
Apps A, C, D, etc. all receive:
  - Player 7 (B) was hit
  - By player 3 (A)
  - With weapon 5
  - For 4 damage
```

---

## Appendix: Network Protocol Summary

### Connection Lifecycle

```
1. Client connects to WiFi AP (Hub)
   - Obtains IP via DHCP (10.10.10.x)

2. Client connects to Discovery (TCP 50000)
   - Sends JOIN packet
   - Receives WELCOME with Client ID

3. Client connects to Sync (TCP 50001)
   - Sends periodic state updates
   - Receives aggregated state from all clients

4. Client connects to Event (TCP 50004)
   - Bidirectional event stream established

5. Client starts Ping (UDP 50002)
   - Periodic echo requests
   - Monitors latency

6. Client starts Time (UDP 50003)
   - Periodic time sync requests
   - Maintains synchronized clock

7. Game in progress
   - Events flow through Event service
   - State synced via Sync service

8. Client disconnects
   - Sends GOODBYE to Discovery
   - Hub broadcasts NWCLIENT_REMOVE to others
```

### Error Handling

Source: `recoilnetwork.h` lines 241-252

**Network Errors:**
```
SYNC_INTERNAL_ERROR: Internal sync service error
SYNC_ALREADY_CONNECTED: Client already has sync connection
EVENT_ALREADY_CONNECTED: Client already has event connection
DISCOVERY_NOT_CONNECTED: Must connect to discovery first
DISCOVERY_ALREADY_CONNECTED: Client already registered
UPGRADE_VERSION_ERROR: Incompatible firmware version
UPGRADE_ALREADY_CONNECTED: Upgrade in progress
UPGRADE_PAYLOAD_ERROR: Corrupt payload block
PING_NO_FREE_CONNECTIONS: Server full (16 clients max)
```

**Error Packet Format:**
```
┌──────────────────────────────────────┐
│ Header (4 bytes, Protocol ID=5)      │
├──────────────────────────────────────┤
│ id: Client ID (U8)                   │
│ error: NetworkErrors enum (U8)       │
│ reserved: 0 (U16)                    │
└──────────────────────────────────────┘
```

---

## References

1. **Recoil Documentation Repository**
   - URL: https://github.com/SkyRocketToys/Recoil_Documentation
   - Files:
     - `Recoil_Protocol_IR.docx` - Infrared protocol specification
     - `Recoil_Protocol_BLE.docx` - Bluetooth protocol specification
     - `Recoil_Gun_Firmware_Config_Guide.docx` - Configuration parameters
     - `Recoil_Gun_Firmware_Upgrade_Guide.docx` - OTA update procedures
     - `Recoil_Gun_Schematic_REV-4.pdf` - Hardware schematics

2. **Recoil Hub OpenWRT Repository**
   - URL: https://github.com/SkyRocketToys/Recoil_Hub_OpenWRT_Main
   - Files:
     - `custom-feed/recoil/src/recoilnetwork.h` - Network service definitions
     - `custom-feed/recoil/src/discovery.c` - Discovery service implementation
     - `custom-feed/recoil/src/sync.c` - Sync service implementation
     - `custom-feed/recoil/src/event.c` - Event service implementation
     - `custom-feed/recoil/src/ping.c` - Ping service implementation
     - `custom-feed/recoil/src/time.c` - Time service implementation
     - `custom-feed/recoil/src/upgrade.c` - OTA upgrade implementation
     - `custom-feed/recoil/src/UnityGameNetworkServer/` - Unity integration layer

---

## License

The Recoil system and documentation are released under the MIT License.

Copyright (c) 2017 Hotgen Ltd / Sky Rocket Toys

See repository LICENSE files for full text.

---

*Document Version: 1.0*
*Last Updated: 2026-01-21*
*Based on Recoil firmware v1.3 and Hub v1.0-1*
