# kl-zero-stick

WiFi sync and **live roast graphs** for a Kaffelogic Nano 7 with the **older main board** (the one with a USB-A
port that only takes a USB stick), using a Raspberry Pi Zero W plugged into the roaster in place of the stick.

The Pi Zero does two things at once:

1. **To the roaster it is a USB stick** (`kl-msc`). The roaster reads profiles from it and writes its roast
   logs to it, exactly as it does with a real stick.
2. **To Kaffelogic Studio it looks like a roaster reachable over WiFi** (`kl-sassi`). Studio syncs logs and
   profiles, shows the roast live while it is happening, and can send a profile to the "roaster" (i.e. onto the stick).

No changes are made to the roaster or to Studio.

<p align="center">
  <img src="docs/img/case-on-roaster.jpg" width="45%" alt="The Pi Zero in its printed case, plugged into the roaster's USB-A port">
  <img src="docs/img/case-side.jpg" width="45%" alt="The case from the side: only the metal part of the USB-A plug sticks out">
</p>

> **Unofficial and unsupported.** This project is not affiliated with or endorsed by Kaffelogic. It has been used
> on exactly one roaster (model KN1007B, firmware 7.22.1) for a short time. It writes to the storage your
> roaster uses. Use it at your own risk and keep a backup of your stick.

## Status

| Feature | State |
|---|---|
| Roaster accepts the Pi as a stick, loads profiles, writes logs | works |
| Studio finds the "roaster", syncs logs and profiles over WiFi | works |
| Live log in Studio during a roast | works |
| Studio switching from live view to the final log at the end of a roast | works |
| Sending a profile from Studio to the stick | works, **off by default** (`KL_WRITE=0`) |
| Recording the real date/time of each roast (the roaster has no clock) | works |
| Roaster preferences / commands from Studio | **not possible** on this board (see Limitations) |

A finished roast as Studio shows it after the live view ends (roast and cooling, synced over WiFi):

![Studio: final log after a live roast](docs/img/studio-log.png)

Studio's *View roaster → Wifi Module* tab with the Pi connected. `kl-sassi` answers with fixed values: module
firmware `9.9.9` (newer than any release, so Studio considers it up to date) and a signal of `-50 dBm` (not a real
measurement). Do not use the *Install* links there: they are meant for the official module and have not been tried
with `kl-sassi`.

![Studio: Wifi Module tab](docs/img/studio-wifi-module.png)

## Why this exists

Newer Nano 7 boards have USB-C and can talk to Studio directly (and people tunnel that over WiFi with USB/IP).
The older board has only a USB-A *host* port and understands nothing but a mass-storage stick, so none of that
applies. Things we measured along the way that may help others, even without a Pi:

- **"USB error / check device" is caused by the FAT32 "dirty" flag.** If byte `0x41` of the boot sector is not 0
  (the volume was not cleanly unmounted on a PC), the roaster rejects the stick. `fsck.fat -a` or clearing that
  byte fixes it. `kl-fat-clean` does this before every start.
- The roaster is a very minimal USB host: full-speed, and it only ever sends `REQUEST SENSE`,
  `PREVENT ALLOW MEDIUM REMOVAL`, `TEST UNIT READY`, `READ(10)` and `WRITE(10)` — no `INQUIRY`, no `READ CAPACITY`.
  The Linux kernel mass-storage gadget was not accepted by our roaster; the userspace implementation here is.
- **The roaster writes the log incrementally during the roast** (one line per second, a write roughly every
  0.5 s, and it updates the file size in the directory each time). That is what makes a live graph possible.
- The USB-A port supplies enough power for a Pi Zero W (no external supply needed). A Pi Zero 2 W draws more and
  has not been tried.
- Presenting a USB serial (CDC-ACM) device instead of a stick gets no response from the roaster: it enumerates the
  device but never configures it.

## How it works

```
 roaster (USB host) ── USB ──► Pi Zero W ── WiFi ──► Kaffelogic Studio (PC)
                               │
                               ├─ kl-msc    userspace USB mass storage (FunctionFS) backed by a FAT32 image file,
                               │            logs every SCSI command to /var/log/kl-msc.log
                               └─ kl-sassi  TCP 9056: speaks Studio's "SASSI" protocol, reads the same image
                                            file with its own read-only FAT32 reader while the roaster has it
```

- Reading (directory listings, logs, profiles, the growing live log) never touches the roaster's view of the stick.
- Writing a profile (only if `KL_WRITE=1`): `kl-sassi` stops `kl-msc`, loop-mounts the image, writes the file,
  unmounts cleanly and starts `kl-msc` again. The roaster sees the stick unplugged and re-plugged for a few
  seconds. Writes are refused (`BUSY`) if the roaster wrote anything in the last 90 s, i.e. while roasting.
- Only `*.kpro` under `kaffelogic/roast-profiles` can be written; `*.kpro` and `*.klog` can be deleted (a copy is
  kept in `/var/lib/kl-sassi/trash/`). Everything else Studio sends (including firmware files) is stored in a
  directory *outside* the image, so firmware can never reach the stick through this path.
- `kl-sassi` reports the firmware version it finds in the newest roast log, so Studio does not offer updates.
- The real start time of each roast is recorded in `/var/lib/kl-sassi/roast-dates.json` (the roaster stamps every
  log 2001-01-01).

Protocol notes are in [docs/PROTOCOL.md](docs/PROTOCOL.md).

## Hardware

- Raspberry Pi Zero W (v1.1) and a microSD card
- A short USB-A male → micro-USB data cable, plugged into the Pi's **USB** port (not PWR)
- The original Kaffelogic USB stick (to clone)

### Case

`case/` has a printable two-part case (OpenSCAD, printed in PETG and in use) that plugs straight into the roaster:
the USB-A plug stands perpendicular to the top face, only its metal shell comes through, and the Pi Zero sits
inside next to it. Two M5 bolts with nuts clamp the base plate, the plug and the top together; four M2.5×12 screws
hold the Pi.

- `shell.stl`, `plate.stl`: the files we printed (the shell is exported upside down, top face on the bed).
- `pizero_case.scad`: the source. It assumes a thin, flexible **flat** USB-A → micro-USB cable. **Measure your plugs**
  and set the values in the `USB-A` / `micro-USB` sections; the script checks that the parts fit and prints the
  accepted M5 bolt lengths. Render a part with `openscad -o shell.stl -D 'part="shell"' pizero_case.scad`
  (`part` = `assembly`, `section`, `print`, `shell`, `plate`).

![case, inside](case/inside.png)

## Installation (outline)

These are the steps we followed on Raspberry Pi OS (Debian 13 "trixie"). Adjust paths to taste.

1. Install Raspberry Pi OS, set up WiFi and SSH. Add to `/boot/firmware/config.txt` under `[all]`:
   `dtoverlay=dwc2,dr_mode=peripheral`
2. Clone your Kaffelogic stick into an image file (on any Linux machine, stick **unmounted**):
   `sudo dd if=/dev/sdX of=nano7.img bs=4M` and copy it to `/var/lib/kaffelogic/nano7.img` on the Pi.
   Keep a second copy somewhere safe. (Our stick is a "superfloppy": FAT32 directly on the device, no partition
   table. A freshly formatted image has **not** been tested.)
3. Copy the files from `pi/`:
   - `kl-msc.py`, `kl-sassi.py`, `kl-fat-clean`, `kl-data-mount` → `/usr/local/bin/` (mode 755)
   - `*.service` → `/etc/systemd/system/`
4. `sudo apt install dosfstools` (for `fsck.fat`), then
   `sudo systemctl daemon-reload && sudo systemctl enable --now kl-data kl-msc kl-sassi`
5. Plug the Pi into the roaster. `/var/log/kl-msc.log` should show `UDC state=configured` within a few seconds
   and the roaster should list your profiles.
6. In Studio enable *Options → Enable Wifi module discovery*. Studio looks for the host name `KaffelogicWM`.
   `kl-sassi` answers LLMNR queries for it; if your PC does not use LLMNR (many managed Windows machines do not),
   add a line to the PC's `hosts` file: `<ip-of-the-pi> KaffelogicWM`. Give the Pi a fixed address in your router.
   Studio's *Scan for Wifi module* did not find the Pi in our setup; the host name route did.

### Configuration (environment variables of `kl-sassi.service`)

| Variable | Default | Meaning |
|---|---|---|
| `KL_WRITE` | `0` | `1` allows profile uploads/deletes inside the stick image |
| `KL_IMG` | `/var/lib/kaffelogic/nano7.img` | the stick image |
| `KL_STATE` | `/var/lib/kl-sassi` | state, overlay files, trash, roast dates |
| `KL_LOG` | `/var/lib/kl-sassi/kl-sassi.log` | server log |
| `KL_SERIAL` | `KLZERO0001` | serial number shown in Studio |
| `KL_KEEPALIVE` | `bridge` | `cr` sends a bare `\r` instead of a bridge status packet |
| `KL_PORT` | `9056` | TCP port Studio connects to |

### Optional: read-only root

The Pi loses power every time the roaster is switched off. To protect the SD card we run the OS from a read-only
root (`overlayroot=tmpfs:recurse=0` on the kernel command line, package `overlayroot`); `kl-data.service` then
bind-mounts the few directories that must stay writable. Without the overlay `kl-data` does nothing. If you enable
it, switch swap to zram only and journald to volatile storage first.

## Day-to-day use

The Pi is powered by the roaster, so it boots every time you switch the roaster on. On our Pi Zero W:

| After switching the roaster on | |
|---|---|
| ~30 s | the roaster sees the stick (profiles available) |
| ~1.5 min | Studio can connect; it then syncs logs and profiles on its own |

You can start a roast as soon as the profiles appear on the roaster; Studio does not need to be connected. If it
connects in the middle of a roast, the live view starts from the beginning of that roast. The Pi switches off with
the roaster; logs written to the stick are kept, and Studio picks up anything it missed the next time it connects.

## Testing without Studio

`tools/sassi_client.py` behaves like Studio as far as we understand it:

```
tools/sassi_client.py <ip-of-the-pi> info 9 ls kaffelogic/roast-profiles get kaffelogic/roast-logs/log0001.klog
```

Because the client was written from the same understanding of the protocol as the server, it cannot catch a
shared misunderstanding; real Studio is the real test.

## Limitations

- One Studio connection at a time.
- The roaster's settings ("Roaster preferences" tab) and commands to the roaster cannot work: on this board the
  roaster only reads and writes files, it has no channel to answer questions.
- If Studio reconnects in the middle of a roast, the whole log so far is sent again.
- The stick becomes available about 30 s after power-on and Studio can connect after roughly 1.5 minutes
  (boot time of a Pi Zero W).
- FAT stores modification times in 2-second steps, so after uploading a profile Studio may download it once more.

## Ports

- **[kl-zero-t-dongle](https://github.com/henrygreenslade/kl-zero-t-dongle/tree/main/t-dongle-s3)** by Henry Greenslade:
  the same idea on a LILYGO T-Dongle-S3 (ESP32-S3). It plugs straight into the roaster, is ready within seconds,
  and adds a web page for the files on the card. Read its power caveat before using it.

## Legal

The protocol description was produced by analysing how Kaffelogic Studio communicates, for the sole purpose of
interoperability with a device we own. This repository contains no Kaffelogic code, firmware or files. The
roaster firmware is encrypted and was not decrypted or disassembled. "Kaffelogic" and "Nano 7" are names of
their respective owner.

## License

MIT, see [LICENSE](LICENSE).
