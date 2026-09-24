# ROOTCON 20 Badge: Firmware Extraction & Analysis

**Author:** Krish Paul ([me@krishnendu.com](mailto:me@krishnendu.com))

This folder holds a full flash dump of a ROOTCON 20 badge (ESP32-S3) and the analysis of what is on it. It covers the hardware, how the flash is laid out, the firmware, the decoded settings, and how to change or restore the badge.

> **Keep this folder private.** The dump contains the badge's **private key** in plain text (see [Security notes](#6-security-notes)).

---

## 1. Hardware

| Item | Value |
|---|---|
| Chip | ESP32-S3 (QFN56), revision v0.2 |
| Features | Wi-Fi, BT 5 (LE), dual core + LP core, 240 MHz |
| Flash | 8 MB embedded, GigaDevice (manufacturer `0xC8`, device `0x4017`), quad I/O, 3.3 V |
| Crystal | 40 MHz |
| USB | Native USB-Serial/JTAG (VID `0x303A`, PID `0x1001`), shows up as **COM9** |
| MAC | `e0:72:a1:e4:e1:34` |

## 2. Firmware

| Item | Value |
|---|---|
| Base | **Meshtastic 2.8.1** (`2.8.1.5b987d4`), edition `VANILLA` |
| PlatformIO env | `rc20-human-fw` (custom badge build) |
| Framework | Arduino-ESP32 3.x on ESP-IDF, built with PlatformIO |
| Hardware model | `PRIVATE_HW` |
| Notable libraries | `arduino-fsm`, BLE, LittleFS, Preferences, SD, SPI, Wire, RMT/RGB LED, Tone |
| Strings | `ROOTCON 20 BADGE`, `ROOTCON 20 Badge` |
| Device state version | 25 (min app version 30200) |
| Capabilities | Bluetooth yes, Wi-Fi no, Ethernet no, PKC yes, can shut down |
| App image | ESP32-S3, entry `0x40374f54`, 7 segments, checksum and SHA-256 **valid** |

## 3. Flash layout

The partition table at `0x8000` is the standard Arduino 8 MB OTA layout.

| Name | Type | Subtype | Offset | Size | Notes |
|---|---|---|---|---|---|
| (bootloader) | – | – | `0x0` | `0x8000` | Second-stage bootloader |
| (partition table) | – | – | `0x8000` | `0x1000` | |
| nvs | data | nvs | `0x9000` | `0x5000` | Changes while the badge runs |
| otadata | data | otadata | `0xE000` | `0x2000` | Boots `app0` (seq 1) |
| app0 | app | ota_0 | `0x10000` | `0x330000` | **Active firmware** |
| app1 | app | ota_1 | `0x340000` | `0x330000` | Empty (`0xFF`) |
| spiffs | data | spiffs | `0x670000` | `0x180000` | Actually **LittleFS** (block 4096, 384 blocks) |
| coredump | data | coredump | `0x7F0000` | `0x10000` | |

## 4. Files in this folder

```
badge_full_8MB.bin          Whole 8 MB flash image (restore image)
partitions/                 Each region above split into its own file
  bootloader_0x0.bin
  partition_table_0x8000.bin
  nvs_0x9000.bin
  otadata_0xe000.bin
  app0_0x10000.bin          Firmware (padded to partition size)
  app1_0x340000.bin         Empty
  spiffs_0x670000.bin       LittleFS image
  coredump_0x7f0000.bin
filesystem/prefs/           Files extracted from LittleFS
  config.proto              LocalConfig (protobuf)
  module.proto              LocalModuleConfig
  channels.proto            ChannelFile
  device.proto              DeviceState (owner, node number)
  nodes.proto               NodeDatabase
  transmit_history.dat
settings_decoded/           The .proto files above as readable text (.txtpb)
```

SHA-256 of `badge_full_8MB.bin`:
`286652664a9a5055993aae8ec82a1794caf22aca7c03388af00e936fd25ff809`

### Dump integrity

`esptool verify-flash` against the live chip reports a mismatch. This is expected. A second full dump differs from the first **only** in 2 sectors of `nvs` and 2 sectors of `spiffs`, which the firmware rewrites while it runs. The bootloader, partition table, otadata, both app slots and coredump are identical in both dumps.

## 5. Decoded settings

### Identity
| Setting | Value |
|---|---|
| Long / short name | `ROOTCON 20 Badge` / `RC20` |
| Node ID / number | `!4add9ee7` / `1256038119` |
| Role | CLIENT |
| Public key | `UD1qu/nmBPGanX5qCEYf8wMtXyc516bOpdWOCb+zdjA=` |

### Radio (LoRa)
| Setting | Value |
|---|---|
| Region | `PH_433` |
| Preset | Default (`use_preset: true`) |
| Hop limit | 3 |
| TX enabled | yes |
| SX126x RX boosted gain | yes |

### Channels
| Index | Role | Key |
|---|---|---|
| 0 | PRIMARY | Default key (`psk = 0x01`, "AQ==") |
| 1–7 | disabled | – |

### Bluetooth / power / display / position
| Setting | Value |
|---|---|
| Bluetooth | enabled, fixed PIN `123456` |
| Wait for Bluetooth | 60 s |
| Light sleep | 300 s, minimum wake 10 s, super deep sleep disabled |
| Screen on | 600 s |
| GPS | not present |
| Position broadcast | every 3600 s, smart broadcast on (min 100 m / 300 s) |
| Node info broadcast | every 10800 s |
| NTP | `meshtastic.pool.ntp.org` |

### Modules
| Module | Setting |
|---|---|
| MQTT | `mqtt.meshtastic.org`, user `meshdev` / `large4cats` (public defaults), root `msh`, encryption on |
| Ambient lighting | RGB (228, 225, 52), current 10 |
| Detection sensor | min broadcast 45 s, trigger LOGIC_HIGH |
| Telemetry | device update interval effectively disabled (`2147483647`) |
| Traffic management | position min interval 18000 s |

### How this was decoded

The path from the full flash dump to the readable `settings_decoded/*.txtpb` files:

1. **Split the flash by partition.** Read the partition table at `0x8000` (see [section 3](#3-flash-layout)) and carve `badge_full_8MB.bin` into the per-region files in `partitions/` at their offsets.
2. **Mount the filesystem.** The `spiffs` partition is actually a LittleFS image (block size 4096, 384 blocks). Open `partitions/spiffs_0x670000.bin` with [`littlefs-python`](https://pypi.org/project/littlefs-python/) and copy out the `/prefs/*.proto` files into `filesystem/prefs/`. These are raw protobuf blobs, not text.
3. **Decode the protobufs to text.** Each blob maps to a Meshtastic schema message. Parse the bytes and print them with `google.protobuf.text_format` to produce the `settings_decoded/*.txtpb` files:

   | File in `filesystem/prefs/` | Protobuf message | Decoded to |
   |---|---|---|
   | `config.proto` | `localonly_pb2.LocalConfig` | `settings_decoded/config.txtpb` |
   | `module.proto` | `localonly_pb2.LocalModuleConfig` | `settings_decoded/module.txtpb` |
   | `channels.proto` | `deviceonly_pb2.ChannelFile` | `settings_decoded/channels.txtpb` |
   | `device.proto` | `deviceonly_pb2.DeviceState` | `settings_decoded/device.txtpb` |
   | `nodes.proto` | `deviceonly_pb2.NodeDatabase` | (kept as raw `nodes.proto`) |

   The `meshtastic` Python package ships these generated classes under `meshtastic.protobuf`.

```python
from google.protobuf import text_format
from meshtastic.protobuf import localonly_pb2, deviceonly_pb2

# example: config.proto -> config.txtpb
cfg = localonly_pb2.LocalConfig()
cfg.ParseFromString(open("filesystem/prefs/config.proto", "rb").read())
open("settings_decoded/config.txtpb", "w").write(text_format.MessageToString(cfg))
```

Byte-string fields in the `.txtpb` output (keys, MAC, device id) are shown as escaped octal because they are raw binary. The base64 forms in the tables above and in `config_backup_before_led.yaml` are the same bytes re-encoded — e.g. the public key `UD1qu/nmBPGanX5qCEYf8wMtXyc516bOpdWOCb+zdjA=` decodes to the `P=j\273...` shown in `config.txtpb`.

> **Note:** `config_backup_before_led.yaml` in this folder is a separate artifact — a Meshtastic CLI export (`--export-config`) taken from the live badge, not a product of this decode pipeline. It reflects a state where the owner was set to `Bidhata` / `KP`. It also contains the private key in plain text (`security.privateKey`), so it falls under the same handling rules as the files below.

## 6. Security notes

- **Private key in the dump.** The Curve25519 private key (`config.security.private_key`) sits in plain text in `badge_full_8MB.bin`, `partitions/spiffs_0x670000.bin`, `filesystem/prefs/config.proto` and `settings_decoded/config.txtpb`. Anyone who has these files can impersonate this node and read direct messages sent to it. Do not share them.
- **Default channel key.** The primary channel uses the well-known Meshtastic default key, so anyone nearby on the same preset can read the traffic.
- **Default Bluetooth PIN.** The PIN `123456` lets anyone in range pair and reconfigure the badge.
- **Flash is not protected.** The flash could be read freely over USB, so Flash Encryption and Secure Boot are not enforced.

## 7. How to change settings

### Option A: Meshtastic CLI (recommended, no reflash)

The badge accepts the standard Meshtastic serial API on COM9.

```sh
python -m meshtastic --port COM9 --info
python -m meshtastic --port COM9 --export-config > my_config.yaml    # back up first
python -m meshtastic --port COM9 --set-owner "Name" --set-owner-short "ME"
python -m meshtastic --port COM9 --set bluetooth.fixed_pin 654321
python -m meshtastic --port COM9 --ch-set psk random --ch-index 0
python -m meshtastic --port COM9 --set lora.region PH_433
python -m meshtastic --port COM9 --set ambient_lighting.red 0
python -m meshtastic --port COM9 --configure my_config.yaml          # bulk apply
```

### Option B: Edit the filesystem image offline

1. Edit `settings_decoded/*.txtpb`.
2. Turn the text back into binary with `google.protobuf.text_format.Parse` and the `meshtastic.protobuf` classes (`localonly_pb2.LocalConfig`, `LocalModuleConfig`, `deviceonly_pb2.ChannelFile`, `DeviceState`).
3. Rebuild the LittleFS image with `littlefs-python` (block size 4096, block count 384).
4. Flash it: `python -m esptool -p COM9 write-flash 0x670000 new_fs.bin`

### Option C: Patch the firmware

Load `partitions/app0_0x10000.bin` in Ghidra (Xtensa / ESP32-S3 loader). After patching, recompute the image checksum and the appended SHA-256, or the bootloader will reject the image.

## 8. Restore the original

```sh
python -m esptool -p COM9 write-flash 0 badge_full_8MB.bin
```

## 9. Tools used

| Tool | Version | Notes |
|---|---|---|
| Python | 3.13 | |
| esptool | 5.4.0 | `python -m esptool` (not on PATH) |
| littlefs-python | latest | Filesystem extraction |
| meshtastic | latest | Protobuf schema + CLI |

Commands used to take the dump:
```sh
python -m esptool -p COM9 flash-id
python -m esptool -p COM9 -b 921600 read-flash 0 ALL badge_full_8MB.bin
python -m esptool image-info partitions/app0_0x10000.bin
```

---
*Extracted 2026-09-24 by Krish Paul ([me@krishnendu.com](mailto:me@krishnendu.com)).*
