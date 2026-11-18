# Donate a drive to CommaView

CommaView needs real drive data to check that its overlays line up correctly across
different openpilot and sunnypilot builds. What it does **not** need is anything about
you, your car's identity, or where you drove.

This repository has one tool, `sanitize_route_for_contribution.py`. It strips a drive
down to just the parts CommaView reads and shows you a receipt of exactly what's left
before you send anything. It runs on your comma device, uploads nothing, and needs no
comma account or login. It writes a file, and you decide whether to send it.

## What you need

- A comma device running openpilot or sunnypilot (or a fork of either).
- A computer on the same Wi-Fi network as the device, with an SSH client. macOS, Linux
  and Windows 10/11 all have one built in (`ssh` and `scp` in a terminal).
- SSH turned on for the device. If you haven't done that before, see
  [Setting up SSH](#setting-up-ssh) below.

## Step 1: connect to the device

Find the device's IP address in its settings (**Settings → Network**, or the Wi-Fi
network's details), then from your computer:

```bash
ssh comma@<device-ip>
```

Everything in steps 2 to 4 runs in that SSH session, on the device.

## Step 2: get the tool

```bash
cd /data/media/0
curl -LO https://raw.githubusercontent.com/CommaView/donate-a-drive/main/sanitize_route_for_contribution.py
```

## Step 3: find a drive

Your drives live in `/data/media/0/realdata`. Each drive is split into one-minute
segments named like `0000002a--abcdef0123--5`: the drive's name, then the minute.

```bash
ls /data/media/0/realdata | tail -20
```

The most recent drives are at the bottom.

## Step 4: run the tool

```bash
python3 sanitize_route_for_contribution.py /data/media/0/realdata \
  --route-name 0000002a--abcdef0123 \
  --segments 3-6 \
  -o /data/media/0 \
  --tar
```

Use your own drive's name in place of `0000002a--abcdef0123`, and the minutes you want
in place of `3-6`. You'll get a folder and a `.tar.gz` in `/data/media/0`, and a
receipt printed on screen.

## Step 5: read the receipt

The receipt lists every kind of data the tool kept, every kind it threw away, how many
messages each had, and the total size. Read it. If something in the "KEPT" list bothers
you, don't send the bundle. The same receipt is saved as `receipt.txt` in the bundle.

## Step 6: copy it to your computer and send it

Leave the SSH session (`exit`), then on your computer:

```bash
scp 'comma@<device-ip>:/data/media/0/commaview-donation-*.tar.gz' .
```

Send that `.tar.gz` to the person at CommaView who asked you for a drive.

To tidy up the device afterwards, remove the tool and the bundle:

```bash
ssh comma@<device-ip> 'rm -r /data/media/0/sanitize_route_for_contribution.py /data/media/0/commaview-donation-*'
```

---

## The options you might want

| Flag | What it does |
| --- | --- |
| `--segments 3-6` | Pick specific minutes of the drive (`3,4,7` works too). Default is the whole drive. |
| `--route-name <name>` | Pick the drive when the folder holds more than one (it always does in `realdata`). |
| `--include-video road,wide,qcamera` | Add camera video. **Off by default.** Read the warning below first. |
| `--include-driver-monitoring` | Add the driver-attention data. **Off by default.** |
| `--allow-endpoints` | Allow the first and last minute of the drive. **Blocked by default.** |
| `--tar` | Also write a `.tar.gz` to send. |
| `-o <folder>` | Where to write the bundle. Default is the current folder. |

### Why the first and last segments are blocked

A drive usually starts where you live and ends where you were going. Those minutes are
the ones that identify you, even without GPS, because the car's motion is enough to
match a driveway. The tool refuses the first and last segment of the drive that are
still on the device unless you pass `--allow-endpoints`.

The device deletes old segments when it runs short of space. If the start of a drive has
already been deleted, the first segment left isn't really where the drive began, and
the one the tool refuses is from the middle. The real start is gone either way, so
nothing is exposed, but it's why picking minutes from the middle of a long drive is the
safest choice.

### Why driver monitoring is off by default

`driverStateV2` and `driverMonitoringState` come from the camera pointed at your face:
head position, eye state, whether you looked away. That's data about you, not about
the car, so it's excluded unless you turn it on.

The interior camera **video** (`dcamera.hevc`) is never included. There's no flag for
it, and the tool checks that it isn't in the bundle.

---

## What is and isn't removed

**Removed.** The tool works from an allowlist. It keeps a short, fixed list of data
CommaView reads and deletes everything else rather than trying to scrub it. These are
never written into the bundle:
- GPS and every other location message
- the navigation destination and route
- raw GNSS
- all log and crash text
- the device's startup record
- the driver-facing camera and everything derived from it
- your device's dongle ID

Because new kinds of data are excluded unless they're added to the list, a future
openpilot release that adds a new message can't quietly leak through.

Within the data it keeps, the tool also clears these:
- **Your car's VIN:** the last six characters, the serial number that identifies your
  individual car, become `X`.
- **The device's mobile network details:** the carrier's name and the cell tower it was
  connected to, which would give a rough location.
- **Any picture embedded in the camera status messages.**
- **Your time zone:** the bundle's timestamp is in UTC.
- **File timestamps that would date the drive.**
- **Your user name in the archive.**

The bundle records which openpilot or sunnypilot build the drive came from (git remote,
branch, commit, version, device model), because that's the point of the bundle. It
never records your dongle ID, device serial or any account identifier.

**Not removed.** The driving data itself is real, and it is yours: steering, speed,
acceleration, the model's predicted path, radar targets and the car's calibration. That
is a detailed record of how one car was driven for those minutes. Someone could rebuild
the shape of the path you drove from it, though not where on the map it was.

If you choose `--include-video`, understand this clearly: **the road and wide camera
video shows exactly where you drove: the streets, the buildings, the license plates of
cars around you, possibly your own street. That can't be anonymized, and this tool
doesn't try.** If you include video, you are sharing your location, whatever the rest
of the bundle says. Only include video of a stretch of road you're comfortable showing
to someone.

If you're unsure, send it without video first. That's the default for a reason.

---

## Running it on a computer instead

If you've copied a drive to a computer that has an openpilot checkout, the tool runs
there too:

```bash
OPENPILOT_ROOT=/path/to/openpilot python3 sanitize_route_for_contribution.py /path/to/route -o .
```

**Use the openpilot or sunnypilot checkout that matches the build that recorded the
drive.** A sunnypilot drive read with a plain openpilot checkout silently loses its
sunnypilot-only data. On the device this is automatic.

---

## Setting up SSH

openpilot lets you log in to the device over SSH with the SSH keys on your GitHub
account. You need to do this once.

1. **Make an SSH key on your computer** if you don't have one. In a terminal (macOS,
   Linux, or Windows PowerShell): `ssh-keygen -t ed25519`, then press Enter at each
   question.
2. **Add the public key to GitHub:** copy the contents of `~/.ssh/id_ed25519.pub` into
   GitHub → Settings → SSH and GPG keys → New SSH key.
3. **On the device:** Settings → Developer (Settings → Network → Advanced on older
   builds). Turn on **Enable SSH**, then under **SSH Keys** enter your GitHub username.
   The device downloads your public keys from GitHub.

## License

[MIT](LICENSE)
