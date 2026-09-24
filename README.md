# ROOTCON 20 Badge — flash dump & notes

Krish Paul · me@krishnendu.com

I pulled a full flash dump off my ROOTCON 20 badge (it's an ESP32-S3 running Meshtastic) and picked it apart to see what's actually on there. This folder is the dump plus everything I worked out from it — the hardware, how the flash is carved up, the firmware, the settings I decoded, and notes on changing or restoring the thing.

Heads up: the dump has the badge's **private key sitting in plain text**. Don't share this folder. More on that further down.

---

## The hardware

Nothing exotic — a bog-standard ESP32-S3 with 8 MB of flash.

| | |
|---|---|
| Chip | ESP32-S3 (QFN56), rev v0.2 |
| Features | Wi-Fi, BT 5 LE, dual core + LP core, 240 MHz |
| Flash | 8 MB embedded, GigaDevice (mfr `0xC8`, dev `0x4017`), quad I/O, 3.3 V |
| Crystal | 40 MHz |
| USB | native USB-Serial/JTAG (VID `0x303A`, PID `0x1001`) — comes up as **COM9** here |
| MAC | `e0:72:a1:e4:e1:34` |

## The firmware

It's Meshtastic under the hood, but a custom badge build — the PlatformIO env is `rc20-human-fw` and the hardware model reports as `PRIVATE_HW`. Poking at the strings you can spot the ROOTCON branding and, more interestingly, a hard region lock (`ROOTCON 20 region lock: forcing PH_433`).

| | |
|---|---|
| Base | Meshtastic 2.8.1 (`2.8.1.5b987d4`), `VANILLA` edition |
| Build env | `rc20-human-fw` |
| Framework | Arduino-ESP32 3.x on ESP-IDF, PlatformIO |
| HW model | `PRIVATE_HW` |
| Libs worth noting | `arduino-fsm`, BLE, LittleFS, Preferences, SD, SPI, Wire, RMT/RGB LED, Tone, Adafruit NeoPixel |
| Device state ver | 25 (min app version 30200) |
| Capabilities | BT yes, Wi-Fi no, Ethernet no, PKC yes, can shut down |

The app image checks out — ESP32-S3, entry `0x40374f54`, 7 segments, and both the checksum and appended SHA-256 are valid. So it's an intact, bootable image, not a partial read.

## How the flash is laid out

The partition table lives at `0x8000` and it's just the stock Arduino 8 MB OTA layout:

| Name | Type | Subtype | Offset | Size | Notes |
|---|---|---|---|---|---|
| bootloader | – | – | `0x0` | `0x8000` | second-stage bootloader |
| partition table | – | – | `0x8000` | `0x1000` | |
| nvs | data | nvs | `0x9000` | `0x5000` | rewritten while running |
| otadata | data | otadata | `0xE000` | `0x2000` | points at `app0` (seq 1) |
| app0 | app | ota_0 | `0x10000` | `0x330000` | the live firmware |
| app1 | app | ota_1 | `0x340000` | `0x330000` | empty, all `0xFF` |
| spiffs | data | spiffs | `0x670000` | `0x180000` | it's actually **LittleFS** (block 4096, 384 blocks) |
| coredump | data | coredump | `0x7F0000` | `0x10000` | |

The one gotcha: the partition is *labelled* `spiffs` but the contents are a LittleFS image. Trying to mount it as SPIFFS will just fail — took me a minute to catch that.

## What's in this folder

```
badge_full_8MB.bin          the whole 8 MB image — this is the restore file
partitions/                 the regions above, each carved out to its own file
  bootloader_0x0.bin
  partition_table_0x8000.bin
  nvs_0x9000.bin
  otadata_0xe000.bin
  app0_0x10000.bin          firmware (padded out to the partition size)
  app1_0x340000.bin         empty
  spiffs_0x670000.bin       the LittleFS image
  coredump_0x7f0000.bin
filesystem/prefs/           files I pulled out of LittleFS (raw protobuf)
  config.proto              LocalConfig
  module.proto              LocalModuleConfig
  channels.proto            ChannelFile
  device.proto              DeviceState (owner, node number)
  nodes.proto               NodeDatabase
  transmit_history.dat
settings_decoded/           the protos above, decoded to readable text (.txtpb)
```

SHA-256 of `badge_full_8MB.bin`:
`286652664a9a5055993aae8ec82a1794caf22aca7c03388af00e936fd25ff809`

**On the dump matching the chip:** `esptool verify-flash` complains about a mismatch, which threw me at first — but it's fine. I took a second full dump and diffed the two, and the only differences are 2 sectors of `nvs` and 2 of `spiffs`, both of which the firmware writes to on its own while running. Everything else — bootloader, partition table, otadata, both app slots, coredump — is byte-identical across both dumps. So the image is good.

## The settings I got out of it

**Identity**

- Name: `ROOTCON 20 Badge` / `RC20`
- Node: `!4add9ee7` (`1256038119`), role CLIENT
- Public key: `UD1qu/nmBPGanX5qCEYf8wMtXyc516bOpdWOCb+zdjA=`

**Radio (LoRa)** — region `PH_433`, default preset (`use_preset: true`), hop limit 3, TX on, SX126x RX boosted gain on.

**Channels** — channel 0 is PRIMARY on the default Meshtastic key (`psk = 0x01`, i.e. `AQ==`). Channels 1–7 are all disabled.

**Bluetooth / power / display / position**

- Bluetooth on, fixed PIN `123456`, waits 60 s for a connection
- Light sleep 300 s, min wake 10 s, super-deep-sleep off
- Screen on 600 s
- No GPS. Position broadcast every 3600 s with smart broadcast on (min 100 m / 300 s)
- Node info every 10800 s, NTP `meshtastic.pool.ntp.org`

**Modules**

- MQTT → `mqtt.meshtastic.org`, `meshdev` / `large4cats` (the public defaults), root `msh`, encryption on
- Ambient lighting → RGB (228, 225, 52), current 10
- Detection sensor → min broadcast 45 s, trigger LOGIC_HIGH
- Telemetry → device update interval basically off (`2147483647`)
- Traffic management → position min interval 18000 s

### Getting from the dump to readable text

The chain, start to finish:

1. **Carve the flash up.** Read the partition table at `0x8000` (the table above) and slice `badge_full_8MB.bin` at each offset into the files under `partitions/`.
2. **Get into the filesystem.** Remember `spiffs` is really LittleFS (block 4096, 384 blocks). Open `partitions/spiffs_0x670000.bin` with [`littlefs-python`](https://pypi.org/project/littlefs-python/) and copy out the `/prefs/*.proto` files into `filesystem/prefs/`. What comes out is raw protobuf, not text.
3. **Decode the protobuf.** Each blob is a Meshtastic schema message. Parse it and dump it with `google.protobuf.text_format` to get the `.txtpb` files. The mapping:

   | in `filesystem/prefs/` | message | out to |
   |---|---|---|
   | `config.proto` | `localonly_pb2.LocalConfig` | `settings_decoded/config.txtpb` |
   | `module.proto` | `localonly_pb2.LocalModuleConfig` | `settings_decoded/module.txtpb` |
   | `channels.proto` | `deviceonly_pb2.ChannelFile` | `settings_decoded/channels.txtpb` |
   | `device.proto` | `deviceonly_pb2.DeviceState` | `settings_decoded/device.txtpb` |
   | `nodes.proto` | `deviceonly_pb2.NodeDatabase` | left as raw `nodes.proto` |

   The generated classes come with the `meshtastic` pip package under `meshtastic.protobuf`, so there's nothing to compile.

```python
from google.protobuf import text_format
from meshtastic.protobuf import localonly_pb2, deviceonly_pb2

# config.proto -> config.txtpb
cfg = localonly_pb2.LocalConfig()
cfg.ParseFromString(open("filesystem/prefs/config.proto", "rb").read())
open("settings_decoded/config.txtpb", "w").write(text_format.MessageToString(cfg))
```

One thing that trips people up: the byte fields in the `.txtpb` output (keys, MAC, device id) print as escaped octal like `P=j\273...` because they're raw bytes. The base64 you see in the tables here and in `config_backup_before_led.yaml` is the same bytes, just re-encoded — e.g. the public key `UD1qu/nmBPGanX5qCEYf8wMtXyc516bOpdWOCb+zdjA=` is exactly that `P=j\273...` from `config.txtpb`.

Also in this folder: `config_backup_before_led.yaml`. That one isn't from the decode above — it's a Meshtastic CLI export (`--export-config`) I grabbed off the live badge before messing with the LEDs. It shows the owner set to `Bidhata` / `KP` at the time, and it's got the private key in it too, so treat it like the rest.

## Security notes (the reason this stays private)

- **The private key is right there in the clear.** `config.security.private_key` (Curve25519) is plain text in `badge_full_8MB.bin`, `partitions/spiffs_0x670000.bin`, `filesystem/prefs/config.proto`, `settings_decoded/config.txtpb`, and the yaml backup. Anyone with those files can impersonate this node and read DMs sent to it. Don't hand them out.
- **Primary channel is on the default key**, so anyone nearby on the same preset can read the traffic. That's normal for the public default channel, just worth knowing.
- **Bluetooth PIN is `123456`** — anyone in range can pair and reconfigure.
- **No flash protection.** Flash Encryption and Secure Boot aren't on, which is exactly why dumping it over USB just worked.

## Changing settings

**Easiest — Meshtastic CLI, no reflash.** The badge speaks the normal serial API on COM9. Back up first, then change what you want:

```sh
python -m meshtastic --port COM9 --info
python -m meshtastic --port COM9 --export-config > my_config.yaml    # back up first!
python -m meshtastic --port COM9 --set-owner "Name" --set-owner-short "ME"
python -m meshtastic --port COM9 --set bluetooth.fixed_pin 654321
python -m meshtastic --port COM9 --ch-set psk random --ch-index 0
python -m meshtastic --port COM9 --set ambient_lighting.red 0
python -m meshtastic --port COM9 --configure my_config.yaml          # bulk apply a yaml
```

Note the region is locked to `PH_433` in firmware, so setting `lora.region` won't stick to anything else.

**Offline — edit the filesystem image.** More work, only worth it if you're rebuilding the image anyway:

1. Edit the `settings_decoded/*.txtpb` files.
2. Turn them back into binary with `text_format.Parse` and the same `meshtastic.protobuf` classes.
3. Rebuild the LittleFS image with `littlefs-python` (block 4096, count 384).
4. Flash it: `python -m esptool -p COM9 write-flash 0x670000 new_fs.bin`

**Firmware — patch the app.** Load `partitions/app0_0x10000.bin` in Ghidra with the Xtensa / ESP32-S3 loader. Whatever you change, you have to recompute the image checksum and the appended SHA-256 afterward or the bootloader throws the image out.

## Putting it back

Full restore is one line:

```sh
python -m esptool -p COM9 write-flash 0 badge_full_8MB.bin
```

## Tools

Python 3.13, esptool 5.4.0 (run as `python -m esptool`, it's not on PATH here), littlefs-python for the filesystem, and the meshtastic package for both the protobuf schema and the CLI.

How I took the dump in the first place:

```sh
python -m esptool -p COM9 flash-id
python -m esptool -p COM9 -b 921600 read-flash 0 ALL badge_full_8MB.bin
python -m esptool image-info partitions/app0_0x10000.bin
```

---
*Dumped and written up 2026-09-24 — Krish Paul (me@krishnendu.com).*
