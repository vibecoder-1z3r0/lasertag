# Recoil Gun Firmware Development & Factory Reset Guide

## Table of Contents
1. [Current Situation](#current-situation)
2. [Writing Custom Firmware](#writing-custom-firmware)
3. [Factory Reset Options](#factory-reset-options)
4. [Firmware Backup & Restore](#firmware-backup--restore)
5. [Development Workflow](#development-workflow)

---

## Current Situation

### What's Available

✅ **Available:**
- Hardware schematics (`Recoil_Gun_Schematic_REV-4.pdf`)
- BLE protocol specification (`Recoil_Protocol_BLE.docx`)
- IR protocol specification (`Recoil_Protocol_IR.docx`)
- Configuration guide (`Recoil_Gun_Firmware_Config_Guide.docx`)
- Firmware signing private key (`priv.pem`)
- DFU (Device Firmware Update) process documentation

❌ **NOT Available:**
- Factory firmware binaries (.hex, .bin, .zip files)
- Firmware source code (.c, .cpp files)
- Pre-compiled firmware images
- Default configuration dumps

### Why No Factory Firmware?

**Most likely reasons:**

1. **Legal/IP Protection** - Source code may contain proprietary algorithms from Hotgen/Sky Rocket Toys
2. **Incomplete Open Source** - Hardware specs released, but firmware kept closed
3. **Repository Separation** - Firmware might be in a private repository
4. **Lost/Abandoned** - Company may have ceased operations, firmware lost

**However:** You have the private signing key, so you can create firmware that guns will accept!

---

## Writing Custom Firmware

### Method 1: Reverse Engineering (Hard Way)

Since no source is available, you'd need to:

1. **Extract existing firmware from a gun:**
   ```bash
   # Using nRF Connect Desktop or J-Link programmer
   nrfjprog --readcode firmware_dump.hex
   ```

2. **Disassemble it:**
   ```bash
   arm-none-eabi-objdump -D -b binary -m arm firmware_dump.hex > disassembly.txt
   ```

3. **Analyze and recreate:**
   - Study the disassembly
   - Identify BLE characteristic handlers
   - Reverse engineer IR timing code
   - Recreate logic in C

**Estimated effort:** 100-500 hours depending on complexity

### Method 2: Build from Scratch (Recommended)

Use Nordic's SDK and the protocol documentation to build new firmware.

#### Prerequisites

```bash
# Install ARM GCC toolchain
sudo apt-get install gcc-arm-none-eabi

# Clone Nordic SDK v12.3 (compatible with nRF52832)
wget https://developer.nordicsemi.com/nRF5_SDK/nRF5_SDK_v12.x.x/nRF5_SDK_12.3.0_d7731ad.zip
unzip nRF5_SDK_12.3.0_d7731ad.zip

# Install nrfutil for packaging/signing
pip install nrfutil

# Install Nordic DFU tools
# https://github.com/NordicSemiconductor/pc-nrfutil
```

#### Firmware Structure

```c
// main.c - Minimal Recoil Gun Firmware

#include <stdint.h>
#include <string.h>
#include "nordic_common.h"
#include "nrf.h"
#include "ble.h"
#include "ble_srv_common.h"
#include "nrf_sdh.h"
#include "nrf_sdh_ble.h"
#include "app_timer.h"

// BLE Service GUIDs (from protocol documentation)
#define RECOIL_SERVICE_UUID     0xE6F59D10-8230-4a5c-B22F-C062B1D329E3
#define ID_CHAR_UUID           0xE6F59D11-8230-4a5c-B22F-C062B1D329E3
#define TELEMETRY_CHAR_UUID    0xE6F59D12-8230-4a5c-B22F-C062B1D329E3
#define CONTROL_CHAR_UUID      0xE6F59D13-8230-4a5c-B22F-C062B1D329E3
#define CONFIG_CHAR_UUID       0xE6F59D14-8230-4a5c-B22F-C062B1D329E3

// Hardware configuration (from schematic)
#define IR_TX_LONG_PIN    22   // Long-range IR LED
#define IR_TX_SHORT_PIN   23   // Short-range IR LED
#define IR_RX_PIN_0       24   // IR sensor 0
#define IR_RX_PIN_1       25   // IR sensor 1
#define IR_RX_PIN_2       26   // IR sensor 2
#define IR_RX_PIN_3       27   // Clip-on sensor
#define MUZZLE_LED_PIN    28   // Green muzzle LED
#define POWER_LED_PIN     29   // White power LED
#define MOTOR_PIN         30   // Recoil motor
#define TRIGGER_PIN       2    // Trigger button
#define RELOAD_PIN        3    // Reload button

// Firmware version
#define FW_VERSION        100  // SVN revision number
#define BL_VERSION        10   // Bootloader version

// Gun configuration
typedef struct {
    uint8_t trigger_mode;
    uint8_t rate_of_fire;
    uint8_t power_ir1;
    uint8_t power_ir2;
    uint8_t power_led1;
    uint8_t power_motor;
    uint8_t flash_mode;
} weapon_config_t;

weapon_config_t weapons[12];  // 12 weapon types
uint8_t current_weapon = 0;
uint8_t gun_id = 0;
uint8_t ammo_count = 0;
uint8_t shot_counter = 0;

// ID Characteristic (20 bytes)
typedef struct {
    uint16_t version;       // Firmware SVN revision
    uint8_t uuid[8];        // Hardware UUID
    uint8_t gun_model;      // 1=Rifle, 2=Pistol
    uint8_t padding[3];
    uint32_t config_crc;    // CRC32 of weapon configs
    uint16_t bl_version;    // Bootloader version
} __attribute__((packed)) id_char_t;

// Telemetry Characteristic (20 bytes)
typedef struct {
    uint8_t pkt_cnt : 4;
    uint8_t cmd_cnt : 4;
    uint8_t gun_id;
    uint8_t buttons;
    uint8_t pressed[3];      // 6×4-bit counters
    int16_t voltage;
    uint16_t ir_payload[2];
    uint8_t ir_metadata[2];
    uint8_t weapon_ammo;
    uint8_t gun_flags;
    uint8_t selected_weapon;
    uint8_t reserved[3];
} __attribute__((packed)) telemetry_char_t;

telemetry_char_t telemetry;

// Initialize BLE stack
void ble_stack_init(void) {
    ret_code_t err_code;

    // Initialize SoftDevice
    err_code = nrf_sdh_enable_request();
    APP_ERROR_CHECK(err_code);

    // Configure BLE parameters
    ble_cfg_t ble_cfg;
    memset(&ble_cfg, 0, sizeof(ble_cfg));
    ble_cfg.gap_cfg.role_count_cfg.periph_role_count = 1;

    err_code = sd_ble_cfg_set(BLE_GAP_CFG_ROLE_COUNT, &ble_cfg, ram_start);
    APP_ERROR_CHECK(err_code);

    // Enable BLE stack
    err_code = nrf_sdh_ble_enable(&ram_start);
    APP_ERROR_CHECK(err_code);
}

// Initialize advertising
void advertising_init(void) {
    ble_advertising_init_t init;
    memset(&init, 0, sizeof(init));

    init.advdata.name_type = BLE_ADVDATA_FULL_NAME;
    init.advdata.include_appearance = false;
    init.advdata.flags = BLE_GAP_ADV_FLAGS_LE_ONLY_GENERAL_DISC_MODE;

    ble_uuid_t service_uuid;
    service_uuid.type = BLE_UUID_TYPE_VENDOR_BEGIN;
    service_uuid.uuid = 0x9D10;
    init.advdata.uuids_complete.uuid_count = 1;
    init.advdata.uuids_complete.p_uuids = &service_uuid;

    init.config.ble_adv_fast_enabled = true;
    init.config.ble_adv_fast_interval = MSEC_TO_UNITS(187.5, UNIT_0_625_MS);
    init.config.ble_adv_fast_timeout = 0;  // No timeout

    ble_advertising_init(&m_advertising, &init);
}

// IR transmission using PWM for 38kHz carrier
void ir_transmit_packet(uint8_t shooter_id, uint8_t weapon_id, uint8_t rounds) {
    // Build 20-bit MAN20A packet
    uint32_t packet = 0;
    packet |= (shooter_id & 0x3F) << 10;   // Bits 15-10
    packet |= (weapon_id & 0x0F) << 6;     // Bits 9-6
    packet |= (rounds & 0x07) << 3;        // Bits 5-3
    packet |= (shot_counter & 0x07);       // Bits 2-0

    // Calculate 4-bit CRC
    uint8_t crc = calculate_crc4(packet);
    packet = (packet << 4) | crc;

    // Manchester encode and transmit
    transmit_header();  // 8 marks, 4 spaces
    for (int i = 19; i >= 0; i--) {
        if (packet & (1 << i)) {
            transmit_manchester_1();  // MS (mark then space)
        } else {
            transmit_manchester_0();  // SM (space then mark)
        }
    }

    shot_counter++;
}

// Manchester bit transmission (600µs timing)
void transmit_manchester_0(void) {
    // SM = Space then Mark
    ir_led_off();
    nrf_delay_us(600);
    ir_led_on_38khz();  // 38kHz carrier
    nrf_delay_us(600);
}

void transmit_manchester_1(void) {
    // MS = Mark then Space
    ir_led_on_38khz();
    nrf_delay_us(600);
    ir_led_off();
    nrf_delay_us(600);
}

// 38kHz carrier generation using PWM
void ir_led_on_38khz(void) {
    // 38kHz = 26.3µs period
    // 13.16µs on, 13.16µs off
    NRF_PWM0->ENABLE = 1;
    NRF_PWM0->PRESCALER = PWM_PRESCALER_PRESCALER_DIV_1;
    NRF_PWM0->COUNTERTOP = 421;  // 16MHz / 421 ≈ 38kHz
    NRF_PWM0->SEQ[0].PTR = (uint32_t)&pwm_values;
    NRF_PWM0->SEQ[0].CNT = 1;
    NRF_PWM0->TASKS_SEQSTART[0] = 1;
}

// Trigger button handler
void trigger_pressed(void) {
    if (ammo_count == 0) {
        // Out of ammo, do nothing
        return;
    }

    weapon_config_t *weapon = &weapons[current_weapon];

    // Fire shot
    ir_transmit_packet(gun_id, current_weapon, 0);  // 0 rounds = 4 actual rounds
    ammo_count--;

    // Feedback
    if (weapon->flash_mode != 0) {
        nrf_gpio_pin_set(MUZZLE_LED_PIN);
    }

    if (weapon->power_motor > 0) {
        nrf_gpio_pin_set(MOTOR_PIN);
        nrf_delay_ms(weapon->power_motor * 5);
        nrf_gpio_pin_clear(MOTOR_PIN);
    }
}

// IR receiver handler (interrupt-driven)
void ir_receiver_handler(nrf_drv_gpiote_pin_t pin, nrf_gpiote_polarity_t action) {
    // Decode IR packet
    uint32_t packet = decode_ir_packet();

    // Extract fields
    uint8_t shooter_id = (packet >> 10) & 0x3F;
    uint8_t weapon_id = (packet >> 6) & 0x0F;
    uint8_t rounds = (packet >> 3) & 0x07;
    uint8_t counter = packet & 0x07;

    // Store in telemetry for app to read
    telemetry.ir_payload[0] = packet & 0xFFFF;
    telemetry.ir_metadata[0] = (event_counter << 4) | (1 << pin);

    // Notify connected app
    ble_telemetry_notify();
}

// Main application entry
int main(void) {
    // Initialize hardware
    nrf_gpio_cfg_output(IR_TX_LONG_PIN);
    nrf_gpio_cfg_output(IR_TX_SHORT_PIN);
    nrf_gpio_cfg_output(MUZZLE_LED_PIN);
    nrf_gpio_cfg_output(MOTOR_PIN);
    nrf_gpio_cfg_input(TRIGGER_PIN, NRF_GPIO_PIN_PULLUP);

    // Initialize BLE
    ble_stack_init();
    gap_params_init();
    services_init();
    advertising_init();

    // Load default weapon configs
    init_default_weapons();

    // Calculate config CRC
    uint32_t crc = crc32_compute((uint8_t*)weapons, sizeof(weapons), NULL);

    // Start advertising as "SRG1_<UUID>"
    advertising_start();

    // Main loop
    while (1) {
        // Process BLE events
        sd_app_evt_wait();
    }
}
```

#### Build Process

```bash
# 1. Configure Makefile
cd nRF5_SDK/examples/ble_peripheral/
cp -r ble_app_template recoil_gun
cd recoil_gun/pca10040/s132/armgcc

# 2. Edit Makefile
# - Set SDK_ROOT path
# - Add source files
# - Set linker script

# 3. Build
make

# 4. Sign firmware with priv.pem
nrfutil pkg generate \
  --hw-version 52 \
  --sd-req 0x00A9 \
  --application-version 100 \
  --application _build/nrf52832_xxaa.hex \
  --key-file ~/Recoil_Documentation/priv.pem \
  recoil_firmware_v100.zip

# 5. Flash via DFU
# Using nRF Connect app on phone, or:
nrfutil dfu serial -pkg recoil_firmware_v100.zip -p /dev/ttyACM0
```

---

## Factory Reset Options

Since no factory firmware exists, here are your options:

### Option 1: Configuration-Only Reset

Reset to known-good weapon configs without changing firmware:

```dart
// Flutter app code
Future<void> factoryResetConfig(BluetoothDevice gun) async {
  // Default weapon configurations
  final defaultWeapons = [
    // Weapon 0: Assault Rifle
    WeaponConfig(
      triggerMode: TriggerMode.fullAuto,
      rateOfFire: 100,  // 50ms × 100 = 5 shots/sec
      powerIR1: 255,
      powerIR2: 25,
      powerLED1: 255,
      powerMotor: 18,
      flashMode: FlashMode.squareWave,
    ),
    // Weapon 1: Pistol
    WeaponConfig(
      triggerMode: TriggerMode.single,
      rateOfFire: 200,  // 1 shot per second
      powerIR1: 200,
      powerIR2: 50,
      powerLED1: 200,
      powerMotor: 10,
      flashMode: FlashMode.squareWave,
    ),
    // ... weapons 2-11
  ];

  // Write each weapon config via BLE Config characteristic
  for (int i = 0; i < 12; i++) {
    await writeWeaponConfig(gun, i, defaultWeapons[i]);
  }

  // Reset global parameters
  await writeShotConfig(gun, autoFeedback: 0x3);
  await writeIRConfig(gun, txRepeats: 2, rxEnable: 0xF);
}
```

**Pros:**
- ✅ No firmware flashing needed
- ✅ Quick (takes ~2 seconds)
- ✅ Safe (can't brick gun)

**Cons:**
- ❌ Doesn't fix firmware bugs
- ❌ Only resets configuration, not code

### Option 2: Firmware Backup & Restore

Create your own "factory" firmware by backing up current firmware:

```bash
# 1. Connect J-Link programmer to gun SWD pins
# (See schematic for SWD pinout)

# 2. Read entire flash
nrfjprog --readcode factory_backup.hex

# 3. Also read UICR (User Information Configuration Registers)
nrfjprog --readuicr factory_uicr.hex

# 4. Store safely
cp factory_backup.hex ~/recoil_backups/gun_serial_BF7EB8_backup_$(date +%Y%m%d).hex
```

**To restore:**
```bash
# 1. Erase gun
nrfjprog --eraseall

# 2. Flash backed up firmware
nrfjprog --program factory_backup.hex --verify

# 3. Flash UICR
nrfjprog --program factory_uicr.hex --verify

# 4. Reset
nrfjprog --reset
```

**Pros:**
- ✅ Perfect restore to original state
- ✅ Preserves factory firmware

**Cons:**
- ❌ Requires hardware programmer (~$10-80)
- ❌ Requires opening gun case
- ❌ Must do BEFORE modifying firmware

### Option 3: Community Firmware Repository

**Suggested approach:**

1. **Extract firmware from unmodified gun** (using Method 2 above)
2. **Upload to GitHub** as "Recoil Factory Firmware Archive"
3. **Community maintains** different versions:
   - `factory_v1.0.hex` - Original firmware
   - `factory_v1.3.hex` - Last known official version
   - `community_v2.0.zip` - Enhanced community firmware
   - `minimal_v1.0.zip` - Bare-bones open source

4. **Anyone can restore** by downloading and flashing

**Example repository structure:**
```
recoil-firmware-archive/
├── README.md
├── factory/
│   ├── v1.0/
│   │   ├── rifle_firmware.hex
│   │   ├── pistol_firmware.hex
│   │   └── checksums.txt
│   └── v1.3/
│       └── ...
├── community/
│   ├── minimal/
│   │   ├── src/
│   │   └── build/
│   └── enhanced/
│       └── ...
└── tools/
    ├── flash.sh
    └── sign.sh
```

### Option 4: DFU Bootloader Reset

Nordic's bootloader has a "factory reset" feature:

```c
// In bootloader code, triggered by holding button during power-on
if (factory_reset_requested()) {
    // Erase application firmware
    sd_flash_page_erase(APPLICATION_START_PAGE);

    // Boot to DFU mode
    // User must then flash new firmware
}
```

**To trigger (if implemented in bootloader):**
1. Power off gun
2. Hold trigger + reload buttons
3. Power on gun
4. Bootloader erases app firmware
5. Muzzle LED blinks (DFU mode)
6. Flash firmware via phone

**Check if available:**
```bash
# Read bootloader settings
nrfjprog --readcode bootloader.hex --family NRF52
# Search for button-combo handling code
```

---

## Firmware Backup & Restore

### Complete Backup Procedure

```bash
#!/bin/bash
# backup_gun.sh - Complete gun firmware backup

SERIAL="BF7EB8"  # First 6 chars of gun UUID
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="backups/gun_${SERIAL}_${DATE}"

mkdir -p "$BACKUP_DIR"

echo "Connecting to gun via J-Link..."
nrfjprog --family NRF52

echo "Reading application firmware..."
nrfjprog --readcode "$BACKUP_DIR/application.hex"

echo "Reading bootloader..."
nrfjprog --memrd 0x0007F000 --w 32 --n 4096 > "$BACKUP_DIR/bootloader.hex"

echo "Reading SoftDevice..."
nrfjprog --memrd 0x00001000 --w 32 --n 98304 > "$BACKUP_DIR/softdevice.hex"

echo "Reading UICR..."
nrfjprog --readuicr "$BACKUP_DIR/uicr.hex"

echo "Reading device info..."
nrfjprog --memrd 0x10000060 --w 32 --n 8 > "$BACKUP_DIR/device_id.txt"

echo "Creating checksums..."
md5sum "$BACKUP_DIR"/*.hex > "$BACKUP_DIR/checksums.md5"

echo "Backup complete: $BACKUP_DIR"
echo "Gun can now be safely modified!"
```

### Complete Restore Procedure

```bash
#!/bin/bash
# restore_gun.sh - Complete gun firmware restore

BACKUP_DIR=$1

if [ -z "$BACKUP_DIR" ]; then
    echo "Usage: ./restore_gun.sh <backup_directory>"
    exit 1
fi

echo "Verifying checksums..."
cd "$BACKUP_DIR"
md5sum -c checksums.md5 || exit 1

echo "Erasing gun..."
nrfjprog --eraseall

echo "Flashing SoftDevice..."
nrfjprog --program softdevice.hex --verify

echo "Flashing application..."
nrfjprog --program application.hex --verify

echo "Flashing bootloader..."
nrfjprog --program bootloader.hex --verify

echo "Flashing UICR..."
nrfjprog --program uicr.hex --verify

echo "Resetting gun..."
nrfjprog --reset

echo "Restore complete!"
echo "Gun should now boot with backed up firmware."
```

---

## Development Workflow

### Recommended Setup

```
┌─────────────────┐
│  Development    │
│  Gun (DUT)      │◄─── J-Link Programmer ($80)
└────────┬────────┘
         │ BLE
         ▼
┌─────────────────┐
│  Test Phone     │
│  (nRF Connect)  │
└─────────────────┘
         │
         ▼
┌─────────────────┐
│  Production     │
│  Gun (backup)   │◄─── Keep unmodified for reference
└─────────────────┘
```

### Development Cycle

1. **Backup** production gun firmware
2. **Develop** firmware on PC
3. **Flash** via J-Link to development gun
4. **Test** with nRF Connect app
5. **Iterate** (repeat 2-4)
6. **Sign** with priv.pem
7. **Deploy** via DFU to all guns

### Testing Checklist

Before deploying firmware:

- [ ] Gun advertises with correct name
- [ ] ID characteristic readable
- [ ] Telemetry updates on button press
- [ ] Control commands work (shoot, reload, recoil)
- [ ] Config updates persist across reboot
- [ ] IR transmission works (verified with oscilloscope or camera)
- [ ] IR reception works (shoot gun with another gun)
- [ ] Battery voltage reported correctly
- [ ] DFU mode accessible (can reflash)
- [ ] Power consumption acceptable (<20mA idle)

---

## Hardware Programmer Setup

### Option 1: Official J-Link EDU ($80)

```bash
# Install J-Link software
wget --post-data "accept_license_agreement=accepted" \
  https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.deb
sudo dpkg -i JLink_Linux_x86_64.deb

# Connect to gun SWD pins:
# VCC → 3.3V
# GND → GND
# SWDIO → SWDIO (Pin from schematic)
# SWCLK → SWCLK (Pin from schematic)

# Test connection
JLinkExe
> connect
> device nrf52832_xxaa
> si swd
> speed 4000
> halt
> r  # Should show registers
```

### Option 2: ST-Link V2 Clone ($5-10)

```bash
# Install OpenOCD
sudo apt-get install openocd

# Create config file: nrf52.cfg
source [find interface/stlink.cfg]
transport select hla_swd
source [find target/nrf52.cfg]

# Flash firmware
openocd -f nrf52.cfg \
  -c "init" \
  -c "reset halt" \
  -c "flash write_image erase application.hex" \
  -c "reset" \
  -c "exit"
```

### Option 3: Raspberry Pi GPIO (Free!)

```bash
# Use OpenOCD with Raspberry Pi
# Connect Pi GPIO pins to gun SWD pins
# BCM 25 → SWDIO
# BCM 11 → SWCLK

# Create Pi config: raspi-swd.cfg
interface bcm2835gpio
bcm2835gpio_peripheral_base 0x20000000
bcm2835gpio_speed_coeffs 113714 28
bcm2835gpio_swd_nums 25 11
transport select swd

source [find target/nrf52.cfg]

# Flash
openocd -f raspi-swd.cfg \
  -c "init" \
  -c "targets" \
  -c "reset halt" \
  -c "flash write_image erase firmware.hex" \
  -c "reset" \
  -c "exit"
```

---

## Summary

### Quick Reference

| Task | Method | Difficulty | Cost |
|------|--------|------------|------|
| **Config Reset** | BLE Config char | Easy | $0 |
| **Firmware Backup** | J-Link/nrfjprog | Medium | $5-80 |
| **Firmware Restore** | J-Link/nrfjprog | Medium | $5-80 |
| **Write Custom FW** | Nordic SDK + C | Hard | $0-80 |
| **Flash via DFU** | nRF Connect app | Easy | $0 |
| **Flash via SWD** | Programmer | Medium | $5-80 |

### Recommended First Steps

1. ✅ **Buy a J-Link EDU** ($80) or ST-Link clone ($10)
2. ✅ **Backup existing firmware** from an unmodified gun
3. ✅ **Upload backup to GitHub** (help the community!)
4. ✅ **Study Nordic SDK examples** (ble_app_hrs, ble_app_uart)
5. ✅ **Build minimal firmware** (just BLE, no IR yet)
6. ✅ **Test on development gun** (verify BLE works)
7. ✅ **Add IR functionality** (study protocol docs)
8. ✅ **Sign with priv.pem** (enable DFU updates)
9. ✅ **Deploy to production** (after extensive testing)

### Resources

- **Nordic SDK:** https://www.nordicsemi.com/Software-and-tools/Software/nRF5-SDK
- **DFU Tools:** https://github.com/NordicSemiconductor/pc-nrfutil
- **J-Link:** https://www.segger.com/products/debug-probes/j-link/
- **OpenOCD:** http://openocd.org/
- **nRF52 Reference:** https://infocenter.nordicsemi.com/topic/struct_nrf52/struct/nrf52832.html

---

## Contributing

If you successfully:
- Extract factory firmware
- Build custom firmware
- Create open-source firmware

**Please share with the community!** Open a pull request or create a new repository.

This documentation will be updated as community firmware development progresses.

---

*Last Updated: 2026-01-21*
*Contributors: Community*
