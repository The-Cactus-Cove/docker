# smart-home

Home Assistant, Mosquitto and Z-Wave JS UI, running on rootless Docker on `squeegee` and deployed through Dockhand.

There are two one-time steps to do on the host before this stack will run. Both are needed because of rootless Docker.

## 1. Mosquitto: disable the host AppArmor profile

Ubuntu ships an AppArmor profile for `/usr/sbin/mosquitto` that only allows reading config from `/etc/mosquitto/`. Rootless Docker can't apply its own AppArmor profile to containers, so the host profile also applies to the mosquitto binary inside the container. Mosquitto then fails with:

```
Error: Unable to open config file '/mosquitto/config/mosquitto.conf'.
```

Disable the profile (this survives reboots):

```bash
sudo ln -s /etc/apparmor.d/mosquitto /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/mosquitto
```

Check that it's gone:

```bash
sudo aa-status | grep -i mosquitto   # should return nothing
```

> An `apparmor` package upgrade can bring the profile back. If Mosquitto stops starting after an upgrade, run these commands again.

## 2. Z-Wave JS: give `squeegee` access to the ZWA-2 stick

Under rootless Docker, the user's `dialout` group doesn't carry into containers, so Z-Wave JS gets `Permission denied` on the serial port. This udev rule makes `squeegee` the owner of the device.

Get the stick's IDs:

```bash
udevadm info -a -n /dev/serial/by-id/usb-Nabu_Casa_ZWA-2_1CDBD4AD2AAC-if00 \
  | grep -E 'ATTRS\{(idVendor|idProduct)\}' | head -2
```

Create the rule, replacing `xxxx` and `yyyy` with the values from the command above:

```bash
echo 'SUBSYSTEM=="tty", ATTRS{idVendor}=="xxxx", ATTRS{idProduct}=="yyyy", OWNER="squeegee", MODE="0660"' \
  | sudo tee /etc/udev/rules.d/99-zwa2.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Check it (the owner should be `squeegee`):

```bash
ls -l "$(readlink -f /dev/serial/by-id/usb-Nabu_Casa_ZWA-2_1CDBD4AD2AAC-if00)"
```

## Notes

- **Map devices to the same path on both sides.** Rootless Docker needs the container-side device path to exist on the host, so `devices:` entries use the same by-id path on the left and right:
  ```yaml
  devices:
    - "/dev/serial/by-id/usb-Nabu_Casa_ZWA-2_1CDBD4AD2AAC-if00:/dev/serial/by-id/usb-Nabu_Casa_ZWA-2_1CDBD4AD2AAC-if00"
  ```
  Set the same path as the serial port in the Z-Wave JS UI settings.
- Home Assistant doesn't need the stick. It connects to Z-Wave JS at `ws://localhost:4000`.
