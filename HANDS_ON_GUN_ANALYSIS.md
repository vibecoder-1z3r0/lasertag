# Hands-On Recoil Gun Analysis Guide

Deep technical dive into analyzing and connecting to Recoil guns via BLE and IR.

## Table of Contents
1. [BLE Connection Basics](#ble-connection-basics)
2. [Can Multiple Devices Connect?](#can-multiple-devices-connect)
3. [Connecting with Mobile Apps](#connecting-with-mobile-apps)
4. [Connecting with PC/Linux](#connecting-with-pclinux)
5. [Reading Real-Time Gun Data](#reading-real-time-gun-data)
6. [IR Signal Analysis](#ir-signal-analysis)
7. [Complete Example: Monitoring a Live Game](#complete-example-monitoring-a-live-game)

---

## BLE Connection Basics

### How BLE Works

Bluetooth Low Energy uses a **Central-Peripheral** model:

```
┌─────────────┐                    ┌─────────────┐
│  Central    │                    │ Peripheral  │
│  (Phone)    │◄──────────────────►│   (Gun)     │
│             │   One Connection   │             │
└─────────────┘                    └─────────────┘
```

**Key Concepts:**

1. **Advertising:** Gun broadcasts "I'm here!" packets every 187.5ms
2. **Scanning:** Phone listens for advertising packets
3. **Connection:** Phone initiates connection to gun
4. **Services/Characteristics:** Structured data exchange after connection

### Recoil Gun BLE Profile

**Advertising Packet:**
```
Device Name: SRG1_BF7EB8569758B65F
             ^^^^ ^^^^^^^^^^^^^^^^
             |    |
             |    +-- 16-char hex UUID (unique gun ID)
             +------- "SRG1" = Recoil Gun model 1 (Rifle)
                      "SRG2" = Recoil Gun model 2 (Pistol)

Service UUID: 0x9D10
Connectable: Yes
Interval: 187.5ms
```

**In DFU Mode (bootloader):**
```
Device Name: SRB1_BF7EB8
             ^^^^ ^^^^^^
             |    |
             |    +-- First 6 chars of UUID
             +------- "SRB1" = Recoil Bootloader model 1
                      "SRB2" = Recoil Bootloader model 2

Muzzle LED: ON (solid green)
Power LED: OFF
```

---

## Can Multiple Devices Connect?

### Short Answer: **NO** (with standard BLE)

**BLE Peripheral Role Limitation:**

The nRF52832 chip in Recoil guns, when acting as a **BLE Peripheral**, supports:
- ❌ **Only 1 central connection at a time**
- ✅ Multiple advertising packets (many can scan, only one can connect)

```
┌─────────┐
│ Phone A │──Connected──► Gun ◄──❌ Rejected──┌─────────┐
└─────────┘                                    │ Phone B │
                                               └─────────┘
```

**What happens if Phone B tries to connect while Phone A is connected:**
- Phone B's connection request is **rejected/ignored**
- Phone A remains connected
- Phone B sees connection timeout error

### Advanced: Multi-Connection Possible?

**YES, but requires custom firmware:**

The nRF52832 **can** support multiple connections if programmed as both:
- **Peripheral** (connected to phones)
- **Central** (connecting to other devices)

```c
// Custom firmware with multiple connections
#define NRF_SDH_BLE_PERIPHERAL_LINK_COUNT 3  // Allow 3 phones

// Nordic SoftDevice S132 supports:
// - Up to 8 simultaneous connections (peripheral + central combined)
// - Up to 20 connections total (if configured)
```

**Stock Firmware:**
- Limited to **1 connection** (standard peripheral-only config)
- This is a firmware choice, not hardware limitation

**Custom Firmware Could Support:**
```
Phone A ──►
            ├──► Gun (with modified firmware)
Phone B ──►
            └──► Can handle 3+ simultaneous connections
Phone C ──►
```

**Use Cases for Multi-Connection:**
- **Spectator Mode:** Phone A plays, Phone B watches stats
- **Tournament Monitoring:** Referee phone + player phone
- **Data Logging:** One phone plays, another logs telemetry

---

## Connecting with Mobile Apps

### Option 1: nRF Connect (Nordic's Official App)

**Best for:** Understanding BLE, debugging, testing

**Download:**
- iOS: https://apps.apple.com/app/nrf-connect/id1054362403
- Android: https://play.google.com/store/apps/details?id=no.nordicsemi.android.mcp

**Step-by-Step:**

1. **Open nRF Connect**
2. **Scan for devices**
   - Tap "SCAN" button
   - Look for `SRG1_` or `SRG2_` devices
   - You'll see signal strength (RSSI) and UUID

3. **Connect to gun**
   - Tap "CONNECT" next to your gun
   - Connection takes ~1-2 seconds
   - You'll see "Connected" status

4. **Explore Services**
   ```
   Generic Access (0x1800)
   └── Device Name (0x2A00) - Read
       Value: "SRG1_BF7EB8569758B65F"

   Device Information (0x180A)
   └── Manufacturer Name (0x2A29) - Read
       Value: "Sky Rocket Toys" (or similar)

   RecoilGun Service (E6F59D10-8230-4a5c-B22F-C062B1D329E3)
   ├── ID (E6F59D11-...) - Read
   │   Value: [20 bytes hex data]
   │
   ├── Telemetry (E6F59D12-...) - Read, Notify
   │   Value: [20 bytes, updates in real-time]
   │
   ├── Control (E6F59D13-...) - Read, Write
   │   Value: [20 bytes command data]
   │
   └── Config (E6F59D14-...) - Write
       Value: [Variable length TLV data]
   ```

5. **Read ID Characteristic**
   - Tap "ID" characteristic
   - Tap "Read" (down arrow icon)
   - You'll see hex bytes:
   ```
   Hex: 64 00 BF 7E B8 56 97 58 B6 5F 01 00 00 00 A4 3C 21 87 0A 00
        ^^-^^ ^^-^^-^^-^^-^^-^^-^^-^^ ^^    ^^-^^-^^ ^^-^^-^^-^^
        |     |                        |     |        |
        |     UUID (8 bytes)          Model  CRC32    Bootloader
        Firmware Version (U16)
   ```

   **Decode:**
   - Bytes 0-1: `0x0064` = Firmware version 100
   - Bytes 2-9: `BF7EB8569758B65F` = Hardware UUID
   - Byte 10: `0x01` = Rifle (0x02 = Pistol)
   - Bytes 14-17: `0x873C21A4` = Config CRC32
   - Bytes 18-19: `0x000A` = Bootloader version 10

6. **Subscribe to Telemetry**
   - Tap "Telemetry" characteristic
   - Tap "Subscribe" (three down arrows icon)
   - **Press trigger on gun** - you'll see value update!

   ```
   Initial: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00

   After trigger press:
            10 00 03 01 10 00 00 0E A2 00 00 00 00 00 00 00 00 02 00 00
            ^^-^^ ^^-^^ ^^    ^^-^^
            |     |     |     Battery voltage
            |     |     Buttons (0x01 = trigger pressed)
            |     GunID (0x03 = player 3)
            Pkt counter
   ```

7. **Send Commands via Control**
   - Tap "Control" characteristic
   - Tap "Write" (up arrow)
   - Enter hex values:

   **Example: Trigger Recoil**
   ```
   Hex: 10 00 00 08 00 03 00 00 00 00 00 00 00
        ^^-^^ ^^-^^ ^^    ^^-^^
        |     |     |     GunID
        |     |     IR_ack
        |     Action (0x0008 = Recoil)
        PktCnt/CmdCnt

   Steps:
   1. PktCounter (nibble 0): 1
   2. CmdCounter (nibble 1): 0
   3. IR_ack: 0
   4. Action: 0x0008
   5. Rest: zeros

   Tap "SEND"
   Gun motor should activate!
   ```

   **Example: Fire Gun**
   ```
   Hex: 21 00 00 01 00 03 05 00 00 00 00 00 00
        ^^-^^ ^^-^^ ^^-^^ ^^
        |     |     |     GunID (3)
        |     |     Action (0x0001 = Shoot)
        |     IR_ack
        PktCnt=2, CmdCnt=1

   Gun will:
   - Transmit IR packet
   - Flash muzzle LED
   - Trigger recoil
   ```

### Option 2: LightBlue (iOS) / BLE Scanner (Android)

**LightBlue (iOS):**
- More user-friendly than nRF Connect
- Great for quick testing
- Download: https://apps.apple.com/app/lightblue/id557428110

**BLE Scanner (Android):**
- Similar to LightBlue
- Good alternative to nRF Connect
- Download: https://play.google.com/store/apps/details?id=com.macdom.ble.blescanner

**Interface is similar to nRF Connect:**
1. Scan
2. Connect
3. Tap service to expand
4. Tap characteristic to read/write/notify

---

## Connecting with PC/Linux

### Option 1: nRF Connect Desktop

**Best for:** Full-featured GUI, logging, scripting

**Install:**
```bash
# Download from:
# https://www.nordicsemi.com/Products/Development-tools/nRF-Connect-for-desktop

# Linux installation
wget https://nsscprodmedia.blob.core.windows.net/prod/software-and-other-downloads/desktop-software/nrf-connect-for-desktop/4.4.1/nrfconnect-4.4.1-x86_64.appimage
chmod +x nrfconnect-4.4.1-x86_64.appimage
./nrfconnect-4.4.1-x86_64.appimage
```

**Usage:**
1. Launch nRF Connect
2. Install "Bluetooth Low Energy" app from app store
3. Select BLE adapter (built-in Bluetooth or USB dongle)
4. Click "Start Scan"
5. Find gun, click "Connect"
6. Same interface as mobile app

### Option 2: Python with Bleak

**Best for:** Automation, logging, custom apps

**Install:**
```bash
pip install bleak
```

**Complete Gun Monitor Script:**

```python
#!/usr/bin/env python3
"""
recoil_gun_monitor.py
Real-time monitoring of Recoil gun telemetry
"""

import asyncio
import struct
from bleak import BleakScanner, BleakClient

# BLE UUIDs
RECOIL_SERVICE = "E6F59D10-8230-4a5c-B22F-C062B1D329E3"
ID_CHAR        = "E6F59D11-8230-4a5c-B22F-C062B1D329E3"
TELEMETRY_CHAR = "E6F59D12-8230-4a5c-B22F-C062B1D329E3"
CONTROL_CHAR   = "E6F59D13-8230-4a5c-B22F-C062B1D329E3"

def parse_id_characteristic(data):
    """Parse 20-byte ID characteristic"""
    version, uuid, gun_model, _, _, _, config_crc, bl_version = struct.unpack(
        '<H8sBBBBIH', data
    )

    return {
        'firmware_version': version,
        'uuid': uuid.hex().upper(),
        'gun_model': 'Rifle' if gun_model == 1 else 'Pistol',
        'config_crc': f'0x{config_crc:08X}',
        'bootloader_version': bl_version
    }

def parse_telemetry(data):
    """Parse 20-byte telemetry characteristic"""
    pkt_cmd = data[0]
    pkt_cnt = pkt_cmd & 0x0F
    cmd_cnt = (pkt_cmd >> 4) & 0x0F

    gun_id = data[1]
    buttons = data[2]
    voltage = struct.unpack('<h', data[6:8])[0]  # Signed 16-bit

    # Parse button states
    button_names = []
    if buttons & 0x01: button_names.append('TRIGGER')
    if buttons & 0x02: button_names.append('RELOAD')
    if buttons & 0x04: button_names.append('WALKIE')
    if buttons & 0x08: button_names.append('RESET')
    if buttons & 0x10: button_names.append('POWER')
    if buttons & 0x20: button_names.append('RECOIL_CNT')

    # Parse IR events
    ir_events = []
    for i in [8, 11]:  # Two IR event structures
        if i + 2 < len(data):
            payload = struct.unpack('<H', data[i:i+2])[0]
            metadata = data[i+2]

            if payload != 0:  # Valid event
                shooter_id = (payload >> 10) & 0x3F
                weapon_id = (payload >> 6) & 0x0F

                if weapon_id <= 11:  # Gun shot
                    rounds = ((payload >> 3) & 0x07)
                    shot_cnt = payload & 0x07
                    rounds_actual = (rounds + 1) * 4

                    ir_events.append({
                        'type': 'gun_shot',
                        'shooter_id': shooter_id,
                        'weapon_id': weapon_id,
                        'rounds': rounds_actual,
                        'shot_counter': shot_cnt,
                        'sensors': metadata & 0x0F,
                        'event_counter': (metadata >> 4) & 0x0F
                    })
                else:  # Grenade
                    ir_events.append({
                        'type': 'grenade',
                        'grenade_id': shooter_id,
                        'state': payload & 0x0F
                    })

    weapon_ammo = data[14]
    gun_flags = data[15]
    selected_weapon = data[16]

    return {
        'packet_counter': pkt_cnt,
        'command_counter': cmd_cnt,
        'gun_id': gun_id,
        'buttons': button_names,
        'voltage_mv': voltage,
        'voltage_v': voltage / 1000.0,
        'ir_events': ir_events,
        'ammo': weapon_ammo,
        'reload_mode': bool(gun_flags & 0x01),
        'clipon_disconnected': bool(gun_flags & 0x02),
        'selected_weapon': selected_weapon
    }

def telemetry_callback(sender, data):
    """Called when telemetry notification received"""
    telemetry = parse_telemetry(data)

    print("\n" + "="*60)
    print(f"Gun ID: {telemetry['gun_id']}")
    print(f"Battery: {telemetry['voltage_v']:.2f}V ({telemetry['voltage_mv']}mV)")
    print(f"Ammo: {telemetry['ammo']}")
    print(f"Weapon: {telemetry['selected_weapon']}")
    print(f"Buttons: {', '.join(telemetry['buttons']) if telemetry['buttons'] else 'None'}")
    print(f"Reload Mode: {telemetry['reload_mode']}")

    if telemetry['ir_events']:
        print("\nIR EVENTS:")
        for event in telemetry['ir_events']:
            if event['type'] == 'gun_shot':
                sensors = []
                for i in range(4):
                    if event['sensors'] & (1 << i):
                        sensors.append(f"S{i}")

                print(f"  🎯 HIT by Player {event['shooter_id']}")
                print(f"     Weapon: {event['weapon_id']}")
                print(f"     Damage: {event['rounds']} rounds")
                print(f"     Shot #: {event['shot_counter']}")
                print(f"     Sensors: {', '.join(sensors)}")
            else:
                print(f"  💣 Grenade {event['grenade_id']} - State {event['state']}")

    print("="*60)

async def find_gun():
    """Scan for Recoil guns"""
    print("Scanning for Recoil guns...")

    devices = await BleakScanner.discover(timeout=5.0)

    guns = []
    for device in devices:
        if device.name and device.name.startswith('SRG'):
            guns.append(device)
            print(f"Found: {device.name} ({device.address}) RSSI: {device.rssi}dBm")

    return guns

async def monitor_gun(address):
    """Connect to gun and monitor telemetry"""

    async with BleakClient(address) as client:
        print(f"\n✅ Connected to {address}")

        # Read ID characteristic
        print("\nReading gun info...")
        id_data = await client.read_gatt_char(ID_CHAR)
        gun_info = parse_id_characteristic(id_data)

        print(f"Gun Model: {gun_info['gun_model']}")
        print(f"UUID: {gun_info['uuid']}")
        print(f"Firmware: v{gun_info['firmware_version']}")
        print(f"Bootloader: v{gun_info['bootloader_version']}")
        print(f"Config CRC: {gun_info['config_crc']}")

        # Subscribe to telemetry
        print("\nSubscribing to telemetry...")
        await client.start_notify(TELEMETRY_CHAR, telemetry_callback)

        print("\n📡 Monitoring gun... (Press Ctrl+C to stop)")
        print("Try pressing buttons or shooting the gun with another gun!")

        # Keep running
        try:
            while True:
                await asyncio.sleep(1)
        except KeyboardInterrupt:
            print("\n\nStopping...")

        await client.stop_notify(TELEMETRY_CHAR)

async def send_command(address, action):
    """Send a control command to the gun"""

    async with BleakClient(address) as client:
        print(f"Connected to {address}")

        # Build control packet
        control_data = bytearray(20)
        control_data[0] = 0x10  # PktCnt=1, CmdCnt=0
        control_data[1] = 0x00  # IR_ack
        control_data[2] = action & 0xFF
        control_data[3] = (action >> 8) & 0xFF
        control_data[4] = 0x01  # GunID (doesn't matter for non-shooting commands)

        print(f"Sending action: 0x{action:04X}")
        await client.write_gatt_char(CONTROL_CHAR, bytes(control_data))
        print("✅ Command sent!")

async def main():
    import sys

    if len(sys.argv) < 2:
        print("Usage:")
        print("  python recoil_gun_monitor.py scan          # Scan for guns")
        print("  python recoil_gun_monitor.py monitor <MAC> # Monitor telemetry")
        print("  python recoil_gun_monitor.py recoil <MAC>  # Trigger recoil")
        print("  python recoil_gun_monitor.py shoot <MAC>   # Fire gun")
        return

    command = sys.argv[1]

    if command == 'scan':
        guns = await find_gun()
        if not guns:
            print("No guns found!")

    elif command == 'monitor':
        if len(sys.argv) < 3:
            print("Error: Provide gun MAC address")
            print("Example: python recoil_gun_monitor.py monitor AA:BB:CC:DD:EE:FF")
            return

        await monitor_gun(sys.argv[2])

    elif command == 'recoil':
        if len(sys.argv) < 3:
            print("Error: Provide gun MAC address")
            return

        await send_command(sys.argv[2], 0x0008)  # Recoil action

    elif command == 'shoot':
        if len(sys.argv) < 3:
            print("Error: Provide gun MAC address")
            return

        await send_command(sys.argv[2], 0x0001)  # Shoot action

    else:
        print(f"Unknown command: {command}")

if __name__ == '__main__':
    asyncio.run(main())
```

**Usage:**

```bash
# 1. Scan for guns
python recoil_gun_monitor.py scan
# Output:
# Found: SRG1_BF7EB8569758B65F (AA:BB:CC:DD:EE:FF) RSSI: -45dBm

# 2. Monitor gun in real-time
python recoil_gun_monitor.py monitor AA:BB:CC:DD:EE:FF

# Output will show:
# - Battery voltage
# - Button presses
# - IR hits detected
# - Current ammo
# - All telemetry updates

# 3. Send commands
python recoil_gun_monitor.py recoil AA:BB:CC:DD:EE:FF   # Trigger motor
python recoil_gun_monitor.py shoot AA:BB:CC:DD:EE:FF    # Fire IR
```

### Option 3: Linux Command Line (bluetoothctl + gatttool)

**Quick and dirty BLE interaction:**

```bash
# 1. Scan for devices
bluetoothctl scan on

# Output:
# [NEW] Device AA:BB:CC:DD:EE:FF SRG1_BF7EB8569758B65F

# 2. Connect
bluetoothctl connect AA:BB:CC:DD:EE:FF

# 3. List services (in another terminal)
gatttool -b AA:BB:CC:DD:EE:FF --primary

# 4. Read ID characteristic
gatttool -b AA:BB:CC:DD:EE:FF --char-read --uuid=E6F59D11-8230-4a5c-B22F-C062B1D329E3

# 5. Subscribe to telemetry
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req --uuid=E6F59D12-8230-4a5c-B22F-C062B1D329E3 --value=0100 --listen

# 6. Send recoil command
gatttool -b AA:BB:CC:DD:EE:FF --char-write-req --uuid=E6F59D13-8230-4a5c-B22F-C062B1D329E3 --value=1000000800030000000000000000
```

---

## Reading Real-Time Gun Data

### What You Can Monitor

**1. Button Events**
```python
# From telemetry:
buttons = data[2]

if buttons & 0x01:
    print("TRIGGER PRESSED!")
    # You can count trigger pulls
    # Measure reaction time
    # Detect rapid fire attempts

if buttons & 0x02:
    print("RELOAD PRESSED!")
    # Track reload frequency
    # Measure reload time
```

**2. Battery Monitoring**
```python
voltage_mv = struct.unpack('<h', data[6:8])[0]
voltage_v = voltage_mv / 1000.0

# Typical values:
# 4.2V = Fully charged
# 3.7V = ~50% charge
# 3.4V = Low battery warning
# 3.0V = Critical, gun may shut down

if voltage_v < 3.4:
    print(f"⚠️ LOW BATTERY: {voltage_v:.2f}V")
```

**3. Hit Detection**
```python
# IR event in telemetry
shooter_id = (ir_payload >> 10) & 0x3F
weapon_id = (ir_payload >> 6) & 0x0F
rounds = ((ir_payload >> 3) & 0x07 + 1) * 4
shot_counter = ir_payload & 0x07
sensors = ir_metadata & 0x0F

print(f"🎯 HIT!")
print(f"   Shooter: Player {shooter_id}")
print(f"   Weapon: {weapon_id}")
print(f"   Damage: {rounds} rounds")
print(f"   Sensors hit: {bin(sensors)}")

# You can:
# - Calculate damage
# - Track who shot who
# - Detect hit direction from sensor bitmask
# - Prevent duplicate hits using shot_counter
```

**4. Weapon Selection**
```python
selected_weapon = data[16]
print(f"Current weapon: {selected_weapon} (0-11)")

# Track weapon usage statistics:
# - Which weapons are most popular?
# - How often do players switch?
# - Performance per weapon
```

**5. Ammo Tracking**
```python
ammo = data[14]
reload_mode = data[15] & 0x01

if reload_mode:
    print("🔄 RELOADING (clip out)")
else:
    print(f"🔫 Ammo: {ammo} rounds")

# Track:
# - Accuracy (shots fired vs hits)
# - Ammo consumption rate
# - Reload frequency
```

### Live Dashboard Example

```python
#!/usr/bin/env python3
"""
Real-time gun dashboard with curses UI
"""

import asyncio
import curses
from bleak import BleakClient
from collections import deque
import time

class GunDashboard:
    def __init__(self, stdscr):
        self.stdscr = stdscr
        self.battery_history = deque(maxlen=50)
        self.hit_log = deque(maxlen=10)
        self.shot_count = 0
        self.hit_count = 0

        curses.curs_set(0)
        curses.init_pair(1, curses.COLOR_GREEN, curses.COLOR_BLACK)
        curses.init_pair(2, curses.COLOR_RED, curses.COLOR_BLACK)
        curses.init_pair(3, curses.COLOR_YELLOW, curses.COLOR_BLACK)

    def update(self, telemetry):
        self.stdscr.clear()

        # Header
        self.stdscr.addstr(0, 0, "=" * 70, curses.A_BOLD)
        self.stdscr.addstr(1, 25, "RECOIL GUN MONITOR", curses.A_BOLD)
        self.stdscr.addstr(2, 0, "=" * 70, curses.A_BOLD)

        # Gun status
        y = 4
        self.stdscr.addstr(y, 2, f"Gun ID: {telemetry['gun_id']}")
        self.stdscr.addstr(y, 20, f"Weapon: {telemetry['selected_weapon']}")
        self.stdscr.addstr(y, 40, f"Ammo: {telemetry['ammo']}")

        # Battery with color
        y += 2
        voltage = telemetry['voltage_v']
        self.battery_history.append(voltage)

        if voltage > 3.7:
            color = curses.color_pair(1)  # Green
        elif voltage > 3.4:
            color = curses.color_pair(3)  # Yellow
        else:
            color = curses.color_pair(2)  # Red

        self.stdscr.addstr(y, 2, f"Battery: {voltage:.2f}V ", color)

        # Battery graph
        graph = "["
        for v in self.battery_history:
            if v > 3.9:
                graph += "█"
            elif v > 3.7:
                graph += "▓"
            elif v > 3.5:
                graph += "▒"
            else:
                graph += "░"
        graph += "]"
        self.stdscr.addstr(y, 20, graph)

        # Buttons
        y += 2
        buttons_str = ", ".join(telemetry['buttons']) if telemetry['buttons'] else "None"
        self.stdscr.addstr(y, 2, f"Buttons: {buttons_str}")

        # Stats
        y += 2
        accuracy = (self.hit_count / self.shot_count * 100) if self.shot_count > 0 else 0
        self.stdscr.addstr(y, 2, f"Shots Fired: {self.shot_count}")
        self.stdscr.addstr(y, 25, f"Hits Taken: {self.hit_count}")
        self.stdscr.addstr(y, 45, f"Accuracy: {accuracy:.1f}%")

        # Hit log
        y += 2
        self.stdscr.addstr(y, 2, "Recent Hits:", curses.A_BOLD)
        y += 1
        for hit in self.hit_log:
            self.stdscr.addstr(y, 4, hit)
            y += 1

        # IR events
        if telemetry['ir_events']:
            for event in telemetry['ir_events']:
                self.hit_count += 1
                hit_str = f"[{time.strftime('%H:%M:%S')}] Player {event['shooter_id']} - {event['rounds']} dmg"
                self.hit_log.append(hit_str)

        self.stdscr.refresh()

async def run_dashboard(stdscr, mac_address):
    dashboard = GunDashboard(stdscr)

    def callback(sender, data):
        telemetry = parse_telemetry(data)  # From previous example
        dashboard.update(telemetry)

    async with BleakClient(mac_address) as client:
        await client.start_notify(TELEMETRY_CHAR, callback)

        while True:
            await asyncio.sleep(0.1)

# Run with:
# curses.wrapper(lambda stdscr: asyncio.run(run_dashboard(stdscr, "AA:BB:CC:DD:EE:FF")))
```

---

## IR Signal Analysis

### Understanding IR Signals

**Physical Properties:**
- **Wavelength:** 940nm (near-infrared, invisible to human eye)
- **Carrier:** 38kHz square wave
- **Encoding:** Manchester (MAN20A for guns, NEC4 for grenades)
- **Range:** Long-range LED ~50ft, short-range ~15ft
- **Timing:** 600µs mark/space duration

### Viewing IR with Camera

**Your phone camera can see IR!**

```
1. Open phone camera app
2. Point gun at camera
3. Press trigger
4. You'll see IR LEDs flash purple/white on screen

This works because:
- Camera sensors detect 940nm light
- Display shows it as visible light
- Human eyes cannot see 940nm
```

**Better View:**
- Use front camera (usually more sensitive to IR)
- Dim the lights
- IR appears as bright white/purple flash

### Hardware IR Receiver

**Option 1: TSOP38238 Module ($2)**

```
┌─────────────┐
│  TSOP38238  │
│   IR Sensor │
│             │
│  Out  Vcc   │
│   │   │     │
│   │   │     │
└───┼───┼─────┘
    │   │
    │   └──── 5V or 3.3V
    │
    └──────── Digital signal to microcontroller
```

**Wiring to Arduino:**
```
TSOP38238 Pin    Arduino Pin
─────────────    ───────────
GND              GND
Vcc              5V
Out              Pin 2 (interrupt)
```

**Arduino Code:**
```cpp
/*
 * Recoil IR Receiver
 * Decodes MAN20A packets from Recoil guns
 */

#define IR_PIN 2

volatile unsigned long lastTime = 0;
volatile unsigned int pulseCount = 0;
volatile unsigned long pulses[100];

void setup() {
  Serial.begin(115200);
  pinMode(IR_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(IR_PIN), irChange, CHANGE);

  Serial.println("Recoil IR Receiver Ready");
  Serial.println("Point gun at sensor and fire!");
}

void irChange() {
  unsigned long now = micros();
  unsigned long duration = now - lastTime;
  lastTime = now;

  if (pulseCount < 100) {
    pulses[pulseCount++] = duration;
  }
}

void loop() {
  if (pulseCount > 40) {  // Enough data for a packet
    // Decode Manchester
    uint32_t packet = 0;
    int bitCount = 0;

    for (int i = 4; i < pulseCount && bitCount < 24; i += 2) {
      unsigned long t1 = pulses[i];
      unsigned long t2 = pulses[i+1];

      // Manchester decoding
      // SM (short-long) = 0
      // MS (long-short) = 1

      if (t1 < 400 && t2 > 700) {  // SM
        packet = (packet << 1) | 0;
        bitCount++;
      } else if (t1 > 700 && t2 < 400) {  // MS
        packet = (packet << 1) | 1;
        bitCount++;
      }
    }

    if (bitCount >= 20) {
      // Extract fields (20 data bits + 4 CRC)
      uint8_t shooter_id = (packet >> 14) & 0x3F;
      uint8_t weapon_id = (packet >> 10) & 0x0F;
      uint8_t rounds_enc = (packet >> 7) & 0x07;
      uint8_t shot_cnt = (packet >> 4) & 0x07;
      uint8_t crc = packet & 0x0F;

      uint8_t rounds = (rounds_enc + 1) * 4;

      Serial.println("\n═══ IR PACKET DECODED ═══");
      Serial.print("Shooter ID: ");
      Serial.println(shooter_id);
      Serial.print("Weapon ID: ");
      Serial.println(weapon_id);
      Serial.print("Rounds: ");
      Serial.println(rounds);
      Serial.print("Shot Counter: ");
      Serial.println(shot_cnt);
      Serial.print("CRC: 0x");
      Serial.println(crc, HEX);
      Serial.println("═════════════════════════\n");
    }

    // Reset
    pulseCount = 0;
    noInterrupts();
    pulseCount = 0;
    interrupts();
  }

  delay(100);
}
```

**Option 2: Oscilloscope**

If you have an oscilloscope:

```
1. Connect TSOP38238 output to scope probe
2. Set trigger to falling edge
3. Trigger level: 2.5V
4. Time base: 500µs/div
5. Fire gun at sensor

You'll see:
- 38kHz carrier bursts (marks)
- Gaps between bursts (spaces)
- Manchester encoding pattern
```

**Typical waveform:**
```
Header: MMMMMMMMSSSSSSSSSSS (8 marks, 4 spaces)
Data:   MSSMMSMSSMMS... (Manchester encoded bits)
```

**Option 3: Python + USB IR Receiver**

Use a USB IR receiver (like ones for TV remotes):

```python
import evdev

# Find IR receiver device
devices = [evdev.InputDevice(path) for path in evdev.list_devices()]
ir_device = None

for device in devices:
    if 'IR' in device.name:
        ir_device = device
        break

if ir_device:
    print(f"Using: {ir_device.name}")

    for event in ir_device.read_loop():
        if event.type == evdev.ecodes.EV_MSC:
            # Decode IR scancode
            print(f"IR Code: 0x{event.value:08X}")
```

### Passive IR Monitoring Setup

**Complete monitoring station:**

```
┌──────────────────────────────────────┐
│   Raspberry Pi Zero W                │
│                                      │
│   ┌────────────┐   ┌─────────────┐  │
│   │ TSOP38238  │   │  BLE Radio  │  │
│   │ IR Sensor  │   │  (built-in) │  │
│   └─────┬──────┘   └──────┬──────┘  │
│         │                 │          │
│      GPIO 17           BLE Stack     │
│         │                 │          │
└─────────┼─────────────────┼──────────┘
          │                 │
          ▼                 ▼
    Receives IR       Connects to guns
    from all guns     and monitors hits
          │                 │
          └────────┬────────┘
                   ▼
          Python analysis script
          Correlates IR and BLE data
                   │
                   ▼
          Real-time web dashboard
```

**Software:**
```python
#!/usr/bin/env python3
"""
Combined IR + BLE monitoring
Raspberry Pi with TSOP38238 on GPIO17
"""

import asyncio
import RPi.GPIO as GPIO
from bleak import BleakScanner, BleakClient
from datetime import datetime

IR_PIN = 17

class GameMonitor:
    def __init__(self):
        self.ir_events = []
        self.ble_guns = {}

        # Setup GPIO for IR
        GPIO.setmode(GPIO.BCM)
        GPIO.setup(IR_PIN, GPIO.IN)
        GPIO.add_event_detect(IR_PIN, GPIO.BOTH, callback=self.ir_callback)

    def ir_callback(self, channel):
        # Decode IR packet (same as Arduino example)
        timestamp = datetime.now()
        # ... decoding logic ...

        print(f"[{timestamp}] IR: Shooter {shooter_id} → Target")
        self.ir_events.append({
            'time': timestamp,
            'shooter': shooter_id,
            'weapon': weapon_id
        })

    async def monitor_ble_gun(self, address):
        def telemetry_callback(sender, data):
            telemetry = parse_telemetry(data)

            if telemetry['ir_events']:
                for event in telemetry['ir_events']:
                    print(f"[BLE] Gun {telemetry['gun_id']} HIT by {event['shooter_id']}")

                    # Correlate with IR events
                    # Find matching IR event within 100ms
                    # Confirm hit was registered

        async with BleakClient(address) as client:
            await client.start_notify(TELEMETRY_CHAR, telemetry_callback)

            while True:
                await asyncio.sleep(1)

    async def run(self):
        # Scan for all guns
        print("Scanning for guns...")
        devices = await BleakScanner.discover()

        guns = [d for d in devices if d.name and d.name.startswith('SRG')]

        print(f"Found {len(guns)} guns")

        # Monitor all guns simultaneously
        tasks = [self.monitor_ble_gun(gun.address) for gun in guns]
        await asyncio.gather(*tasks)

# Run monitoring station
monitor = GameMonitor()
asyncio.run(monitor.run())
```

---

## Complete Example: Monitoring a Live Game

### Scenario: 4-Player Free-for-All

**Setup:**
```
4 Recoil guns (Gun IDs: 1, 2, 3, 4)
1 Raspberry Pi monitor (BLE + IR receiver)
1 Laptop running analysis dashboard
```

**Step 1: Configure Guns**

```python
# setup_game.py
import asyncio
from bleak import BleakScanner, BleakClient

async def setup_gun(address, gun_id):
    async with BleakClient(address) as client:
        # Set gun ID via Control characteristic
        control = bytearray(20)
        control[0] = 0x10  # PktCnt/CmdCnt
        control[4] = gun_id  # GunID
        control[6] = 30  # Ammo = 30 rounds
        control[2] = 0x04  # Action: Set ammo, unset reload
        control[3] = 0x00

        await client.write_gatt_char(CONTROL_CHAR, bytes(control))
        print(f"✅ Gun {gun_id} configured (ammo: 30)")

async def main():
    devices = await BleakScanner.discover()
    guns = [d for d in devices if d.name and d.name.startswith('SRG')][:4]

    for i, gun in enumerate(guns, 1):
        await setup_gun(gun.address, i)

asyncio.run(main())
```

**Step 2: Start Monitoring**

```python
# game_monitor.py
import asyncio
from bleak import BleakClient
import json
from datetime import datetime
import aiohttp

class GameState:
    def __init__(self):
        self.players = {
            1: {'health': 100, 'kills': 0, 'deaths': 0, 'shots': 0},
            2: {'health': 100, 'kills': 0, 'deaths': 0, 'shots': 0},
            3: {'health': 100, 'kills': 0, 'deaths': 0, 'shots': 0},
            4: {'health': 100, 'kills': 0, 'deaths': 0, 'shots': 0},
        }
        self.event_log = []

    def process_hit(self, victim_id, shooter_id, damage):
        self.players[victim_id]['health'] -= damage
        self.players[shooter_id]['kills'] += 1

        event = {
            'time': datetime.now().isoformat(),
            'type': 'hit',
            'victim': victim_id,
            'shooter': shooter_id,
            'damage': damage,
            'health_remaining': self.players[victim_id]['health']
        }

        self.event_log.append(event)

        if self.players[victim_id]['health'] <= 0:
            self.players[victim_id]['deaths'] += 1
            self.players[victim_id]['health'] = 100  # Respawn

            event = {
                'time': datetime.now().isoformat(),
                'type': 'death',
                'player': victim_id,
                'killer': shooter_id
            }
            self.event_log.append(event)

        return event

    async def send_to_dashboard(self, websocket_url):
        async with aiohttp.ClientSession() as session:
            async with session.ws_connect(websocket_url) as ws:
                while True:
                    await ws.send_json({
                        'players': self.players,
                        'events': self.event_log[-10:]
                    })
                    await asyncio.sleep(0.1)

game = GameState()

async def monitor_gun(address, gun_id):
    def callback(sender, data):
        telemetry = parse_telemetry(data)

        # Process IR hits
        for event in telemetry['ir_events']:
            if event['type'] == 'gun_shot':
                hit_event = game.process_hit(
                    victim_id=gun_id,
                    shooter_id=event['shooter_id'],
                    damage=event['rounds']
                )

                print(f"💥 Player {gun_id} HIT by Player {event['shooter_id']} - {event['rounds']} damage")
                print(f"   Health: {game.players[gun_id]['health']}/100")

    async with BleakClient(address) as client:
        await client.start_notify(TELEMETRY_CHAR, callback)

        while True:
            await asyncio.sleep(1)

async def main():
    # Find all 4 guns
    devices = await BleakScanner.discover()
    guns = [d for d in devices if d.name and d.name.startswith('SRG')][:4]

    # Monitor all simultaneously
    tasks = [monitor_gun(gun.address, i+1) for i, gun in enumerate(guns)]

    # Also send to dashboard
    tasks.append(game.send_to_dashboard('ws://localhost:8080/game'))

    await asyncio.gather(*tasks)

asyncio.run(main())
```

**Step 3: Web Dashboard**

```html
<!-- dashboard.html -->
<!DOCTYPE html>
<html>
<head>
    <title>Recoil Game Monitor</title>
    <style>
        body { font-family: monospace; background: #000; color: #0f0; }
        .player { border: 2px solid #0f0; margin: 10px; padding: 10px; }
        .health-bar { background: #300; height: 20px; position: relative; }
        .health-fill { background: #0f0; height: 100%; transition: width 0.3s; }
        .events { border: 2px solid #0f0; padding: 10px; height: 300px; overflow-y: auto; }
    </style>
</head>
<body>
    <h1>🎮 RECOIL LASER TAG - LIVE GAME 🎮</h1>

    <div id="players"></div>
    <div class="events">
        <h2>Event Log</h2>
        <div id="events"></div>
    </div>

    <script>
        const ws = new WebSocket('ws://localhost:8080/game');

        ws.onmessage = (event) => {
            const data = JSON.parse(event.data);

            // Update player stats
            let html = '';
            for (let [id, player] of Object.entries(data.players)) {
                html += `
                    <div class="player">
                        <h2>Player ${id}</h2>
                        <div>Health: ${player.health}/100</div>
                        <div class="health-bar">
                            <div class="health-fill" style="width: ${player.health}%"></div>
                        </div>
                        <div>Kills: ${player.kills} | Deaths: ${player.deaths}</div>
                    </div>
                `;
            }
            document.getElementById('players').innerHTML = html;

            // Update events
            let eventsHtml = '';
            for (let evt of data.events.reverse()) {
                if (evt.type === 'hit') {
                    eventsHtml += `<div>💥 Player ${evt.victim} HIT by Player ${evt.shooter} (-${evt.damage} HP)</div>`;
                } else if (evt.type === 'death') {
                    eventsHtml += `<div>☠️ Player ${evt.player} ELIMINATED by Player ${evt.killer}</div>`;
                }
            }
            document.getElementById('events').innerHTML = eventsHtml;
        };
    </script>
</body>
</html>
```

---

## Summary

### BLE Multi-Connection Answer

**Standard firmware:**
- ❌ **Only 1 device can connect at a time**
- First connected device "owns" the gun
- Second device gets rejected

**Custom firmware:**
- ✅ **Can support 3-8 simultaneous connections**
- Requires modifying firmware with Nordic SDK
- Set `NRF_SDH_BLE_PERIPHERAL_LINK_COUNT` to desired value

### Best Tools for Gun Analysis

| Tool | Platform | Best For | Cost |
|------|----------|----------|------|
| **nRF Connect** | iOS/Android | Quick testing | Free |
| **Python + Bleak** | Any | Automation, logging | Free |
| **TSOP38238 + Arduino** | Hardware | IR analysis | $5 |
| **Oscilloscope** | Hardware | Deep protocol analysis | $50-500 |

### Quick Start Checklist

✅ **To monitor a gun right now:**
1. Download nRF Connect app
2. Scan for `SRG1_` or `SRG2_` device
3. Connect
4. Subscribe to Telemetry characteristic
5. Press trigger on gun, watch values update!

✅ **To see IR transmissions:**
1. Open phone camera app
2. Point gun at camera
3. Press trigger
4. See IR flash on screen!

✅ **To build custom monitoring:**
1. Install Python + Bleak
2. Use provided `recoil_gun_monitor.py` script
3. Run: `python recoil_gun_monitor.py scan`
4. Connect and monitor!

---

*Want to dive deeper into any specific area? Let me know!*
