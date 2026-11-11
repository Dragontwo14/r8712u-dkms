# r8712u - Realtek RTL8188SU USB Wi-Fi Driver

A DKMS-compatible adaptation based on the former Linux kernel driver
`drivers/staging/rtl8712` from Linux v6.12.

The driver was tested with a Realtek USB Wi-Fi adapter using USB ID
`0bda:8171`.

## Project status

**Working on the tested system.**

Test environment:
- Distribution: Pop!_OS
- Kernel used for the successful test: `7.2.15-76070215-generic`
- Wi-Fi adapter: Realtek RTL8188SU, USB ID `0bda:8171`
- Firmware: `rtlwifi/rtl8712u.bin`
- Installation method: DKMS

The following checks were successful:
- The module compiled and installed successfully.
- The module loaded automatically after reboot.
- The Wi-Fi interface was detected.
- Nearby Wi-Fi networks were detected and a connection was established.
- Internet connectivity was verified.

These results apply to the tested setup. They do not guarantee compatibility
with other kernels, distributions, or adapter revisions.

## Project structure

```text
r8712u-project/
├── source/                    # Linux v6.12 source (sparse checkout)
├── r8712u-dkms/
│   ├── src/                   # Adapted driver source files
│   ├── Makefile
│   ├── Kbuild
│   └── dkms.conf
├── r8712u-kernel-v6.12.patch  # Changes compared with the original source
├── rtl871x_ioctl_linux.c.save # Separately stored source backup
└── README.md
```

## Changes compared with Linux v6.12

The following eight source files were modified:

| File | Change |
|---|---|
| `mlme_linux.c` | Replaced five `from_timer()` calls with `container_of()` |
| `recv_linux.c` | Updated the timer lookup for packet reordering |
| `rtl8712_led.c` | Updated the LED timer lookup |
| `rtl871x_cmd.c` | Added the `linux/timer.h` header |
| `rtl871x_ioctl_linux.c` | Added the `linux/hex.h` header |
| `rtl871x_pwrctrl.c` | Updated the power-management timer lookup |
| `rtl871x_security.c` | Updated the TKIP timer lookup and added `linux/timer.h` |
| `usb_intf.c` | Added the `linux/hex.h` header |

The timer callbacks use `container_of()` to obtain the containing structure
from the `timer_list` pointer. These changes allow the older driver code to
compile with the kernel headers used in the tested environment.

The complete patch is stored in `r8712u-kernel-v6.12.patch`.

## Requirements

- Linux kernel headers matching the running kernel
- GCC or a compatible C compiler
- `make`
- DKMS
- Git to retrieve the original source
- The firmware file `rtlwifi/rtl8712u.bin`

On the tested system, the firmware was already installed at:

```text
/lib/firmware/rtlwifi/rtl8712u.bin
```

## Retrieving the original source

The Linux v6.12 source was retrieved using a sparse checkout:

```bash
mkdir -p ~/r8712u-project
cd ~/r8712u-project

git clone --depth 1 --branch v6.12 --filter=blob:none --sparse https://github.com/torvalds/linux.git source
cd source
git sparse-checkout set drivers/staging/rtl8712
```

The driver files were then copied into the DKMS project:

```bash
cd ~/r8712u-project
mkdir -p r8712u-dkms/src
cp -a source/drivers/staging/rtl8712/. r8712u-dkms/src/
```

The compatibility changes described above must then be applied to the
corresponding files. The patch file can be used as a reference.

## Building and installing

The following commands describe the DKMS setup used for this project. They
assume that `r8712u-dkms/Makefile`, `Kbuild`, and `dkms.conf` are configured
as described in the project and that the adapted source files are in place.

```bash
cd ~/r8712u-project
sudo cp -a r8712u-dkms /usr/src/r8712u-1.0
sudo dkms add -m r8712u -v 1.0
sudo dkms build -m r8712u -v 1.0
sudo dkms install -m r8712u -v 1.0
```

Check the DKMS status:

```bash
dkms status
```

Load the module for the running kernel:

```bash
sudo modprobe r8712u
```

## Verifying the driver

Check whether the USB adapter is detected:

```bash
lsusb -d 0bda:8171
```

Check whether the module is loaded:

```bash
lsmod | grep r8712u
```

List network devices:

```bash
nmcli device status
```

NetworkManager can be used to manage Wi-Fi connections. The following
commands were used successfully to test internet connectivity:

```bash
ping -c 4 1.1.1.1
ping -c 4 example.com
```

After rebooting, check again that the module is loaded and the Wi-Fi
connection is available.

## Uninstalling

Remove the DKMS installation:

```bash
sudo dkms remove -m r8712u -v 1.0 --all
sudo depmod -a
```

Check whether the module is still loaded:

```bash
lsmod | grep r8712u
```

If it is still loaded, disconnect the Wi-Fi connection first and then unload
the module:

```bash
sudo modprobe -r r8712u
```

## Notes

- The kernel module is named `r8712u`.
- The firmware must be available at the expected path.
- After a kernel update, DKMS attempts to build the module for the new kernel.
  If that build fails, the driver may need further kernel API compatibility
  changes.
- Secure Boot was disabled on the tested system. With Secure Boot enabled,
  module signing may be required.
- During the successful build, BTF generation was skipped because a `vmlinux`
  file was unavailable. This was not a build failure.

## License and origin

The driver is based on the Linux kernel v6.12 source in
`drivers/staging/rtl8712`. The original copyright and license notices in
the source files remain authoritative. The local compatibility changes are
documented in the patch file.
