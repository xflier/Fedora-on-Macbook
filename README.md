# Fedora on MacBook Pro 11,4 (Mid-2015)

This repository documents hardware-specific fixes and tips to run Fedora Linux reliably on the MacBook Pro 11,4 (Mid-2015). The laptop can run Fedora well, but a few workarounds are required for the FaceTime HD camera and for suspend/wake stability.

## Contents

- Overview
- FaceTime HD camera (firmware & driver)
- Suspend / wake fixes (network & CPU tweaks)
- Notes & troubleshooting
- Credits

---

## Overview

Primary issues covered here (on Fedora 43):

- FaceTime HD camera not recognized (requires non-free firmware + driver)
- Long delays or hangs when waking from suspend (network and power/state workarounds)

This document collects links and commands from upstream projects and community posts. Use with caution and review each command before running.

## 1) FaceTime HD camera

The FaceTime camera in this model uses a PCIe device that needs firmware and the `facetimehd` driver.

Step A — Install firmware

Clone and build the firmware package, then install it:

```bash
git clone https://github.com/patjak/facetimehd-firmware.git
cd facetimehd-firmware
make
sudo make install
```

The firmware files should end up in `/usr/lib/firmware/facetimehd` or `/lib/firmware/facetimehd`.

Step B — Sensor calibration files

You must provide the sensor `.dat` calibration files (extracted from Apple firmware). Common names include:

- `1771_01XX.dat`
- `1871_01XX.dat`
- `1874_01XX.dat`
- `9112_01XX.dat`

Place them in the firmware directory (e.g. `/lib/firmware/facetimehd/`). See the `facetimehd` firmware project for extraction instructions.

Ensure the firmware is included in initramfs so it is available early:

```bash
sudo tee /etc/dracut.conf.d/facetimehd.conf <<'EOF'
install_items+=" /usr/lib/firmware/facetimehd/firmware.bin "
EOF
sudo dracut -f
```

Step C — Install driver

Enable the COPR and install the DKMS package (project maintained on COPR):

```bash
sudo dnf copr enable frgt10/facetimehd-dkms
sudo dnf install facetimehd
```

If the packaged camera app does not detect the device, try an alternate viewer such as `guvcview`.

## 2) Suspend & Wake stability

Some combinations of drivers and network backends can cause long delays when resuming from suspend. The most common mitigations are switching NetworkManager's WiFi backend to `iwd`, disabling USB autosuspend, and adding a small system-sleep helper to manage CPU online states.

Switch NetworkManager to `iwd` (faster reconnection)

```bash
# install iwd
sudo dnf install iwd

# create or update /etc/NetworkManager/conf.d/wifi-backend.conf
sudo tee /etc/NetworkManager/conf.d/wifi-backend.conf <<'EOF'
[device]
wifi.backend=iwd
EOF

# stop and mask wpa_supplicant, then restart NetworkManager
sudo systemctl stop wpa_supplicant
sudo systemctl mask wpa_supplicant
sudo systemctl restart NetworkManager
```

Disable USB autosuspend (may help devices that hang resume):

```bash
sudo grubby --update-kernel=ALL --args="usbcore.autosuspend=-1"
```

Add CPU online management during suspend/resume

Create `/usr/lib/systemd/system-sleep/fix_wake` with the following content:

```bash
#!/bin/sh
case "$1" in
    pre)
        for i in 1 2 3 4 5 6 7; do
            echo 0 > /sys/devices/system/cpu/cpu$i/online || true
        done
        ;;
    post)
        for i in 1 2 3 4 5 6 7; do
            echo 1 > /sys/devices/system/cpu/cpu$i/online || true
        done
        ;;
esac
```

Make it executable:

```bash
sudo chmod +x /usr/lib/systemd/system-sleep/fix_wake
```

MAC address randomization

If you want to enable the repository's suggested NetworkManager MAC randomization config, copy `main.conf` to `/etc/iwd/` and restart iwd:

```bash
sudo cp main.conf /etc/iwd/
sudo systemctl restart iwd
```

## Notes & troubleshooting

- Extraction of the `.dat` calibration files requires access to Apple firmware; follow the `facetimehd-firmware` project instructions.
- If the camera still fails to initialize, check `dmesg` for `facetimehd` or firmware messages.
- For suspend issues, test changes incrementally (switching `iwd` first is low-risk).

## Credits

- FaceTimeHD firmware project: https://github.com/patjak/facetimehd-firmware
- Fedora COPR package: frgt10 (facetimehd-dkms)

