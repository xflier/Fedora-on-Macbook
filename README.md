# Fedora on MacBook Pro 11,4 (Mid-2015)

This repository documents hardware-specific fixes and tested settings for running Fedora Linux reliably on a MacBook Pro 11,4 (Mid-2015) with a focus on:
- FaceTime HD camera support
- suspend/resume stability on modern Fedora releases
- Wi-Fi and power management tweaks

---

## Quick start

1. Ensure system is updated: `sudo dnf upgrade --refresh`
2. Install FaceTime HD firmware and driver (Section 1)
3. Apply suspend/wake fixes (Section 2)
4. Reboot and verify:
   - `lsmod | grep facetimehd`
   - `dmesg | grep facetimehd`
   - `systemctl suspend` then resume behavior

---

## 1. FaceTime HD camera

### 1.1 Install firmware

```bash
sudo dnf install git make gcc kernel-devel
rm -rf /tmp/facetimehd-firmware
git clone https://github.com/patjak/facetimehd-firmware.git /tmp/facetimehd-firmware
cd /tmp/facetimehd-firmware
make
sudo make install
```

Firmware path (one of):
- `/usr/lib/firmware/facetimehd`
- `/lib/firmware/facetimehd`

### 1.2 Add sensor calibration `.dat` files

The FaceTime camera requires Apple sensor calibration blobs (not redistributed here).

- Copy `.dat` files such as `1771_01XX.dat`, `1871_01XX.dat`, `1874_01XX.dat`, `9112_01XX.dat`.
- Place them in the firmware folder: `/lib/firmware/facetimehd/`.

### 1.3 Ensure firmware is in initramfs

```bash
sudo tee /etc/dracut.conf.d/facetimehd.conf <<'EOF'
install_items+=" /usr/lib/firmware/facetimehd/firmware.bin "
EOF
sudo dracut -f
```

### 1.4 Install facetimehd driver

```bash
sudo dnf copr enable frgt10/facetimehd-dkms
sudo dnf install facetimehd
sudo dnf reinstall facetimehd --refresh
```

- If the camera is not detected by your app, test with `guvcview`.
- Check `dmesg` for errors: `dmesg | grep -E "facetimehd|firmware"`.

---

## 2. Suspend & wake stability

### 2.1 Use iwd wifi backend (recommended)

```bash
sudo dnf install iwd
sudo tee /etc/NetworkManager/conf.d/wifi-backend.conf <<'EOF'
[device]
wifi.backend=iwd
EOF
sudo systemctl stop wpa_supplicant
sudo systemctl mask wpa_supplicant
sudo systemctl restart NetworkManager
```

### 2.2 Disable USB autosuspend

```bash
sudo grubby --update-kernel=ALL --args="usbcore.autosuspend=-1"
```

### 2.3 CPU online control during sleep

Create helper `/usr/lib/systemd/system-sleep/fix_wake`:

```bash
#!/bin/sh
case "$1" in
  pre)
    /usr/sbin/modprobe -r facetimehd
    for i in 1 2 3 4 5 6 7; do
      echo 0 > /sys/devices/system/cpu/cpu$i/online || true
    done
    ;;
  post)
    for i in 1 2 3 4 5 6 7; do
      echo 1 > /sys/devices/system/cpu/cpu$i/online || true
    done
    /usr/sbin/modprobe facetimehd
    systemctl restart NetworkManager
    ;;
esac
```

```bash
sudo chmod +x /usr/lib/systemd/system-sleep/fix_wake
```

### 2.4 Broadcom calibration data

- Copy `brcmfmac43602-pcie.txt` from this repo to `/lib/firmware/brcm/` if required by your Broadcom Wi-Fi chip.

### 2.5 Optional MAC randomization for iwd

```bash
sudo cp main.conf /etc/iwd/main.conf
sudo systemctl restart iwd
```

---

## 3. Notes & troubleshooting

- FaceTime HD requires non-free firmware (GPL blob) and calibration data from Apple; follow the upstream `patjak/facetimehd-firmware` README.
- Run `sudo journalctl -b | grep facetimehd` for persistent camera log details.
- For suspend issues, apply one change at a time and test.
- Revert `wifi.backend` to `wpa_supplicant` if iwd causes behavior you cannot resolve.

---

## Credits

- FaceTimeHD firmware project: https://github.com/patjak/facetimehd-firmware
- Fedora COPR package: frgt10/facetimehd-dkms
- MacBook Pro 11,4 community troubleshooting notes

