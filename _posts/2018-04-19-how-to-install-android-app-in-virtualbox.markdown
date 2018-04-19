# [note to self] How to install android app on VirtualBox

1. Install `adb` on the host machine
2. Change guest's network adapter to be Host-Only
  There might be some issues with VirtualBox (especially in case of installing a new kernel in the interim). Common solutions are:
- reinstalling `virtualbox-dkms` package (`apt install --reinstall virtualbox-dkms`)
- load VirtualBox kernel modules:
  - `modprobe vboxdrv` to run an instance of a virtual machine
  - `modprobe vboxnetadp` to use a bridged network adapter
  - `modprobe vboxnetflt` to use a Host-Only adapter
  - `modprobe vboxpci` if a virtual machine needs to pass thru a PCI device on the host
3. `adb connect <Guest IP address>` (`alt+f1` to switch to console, `alt+f7` to switch to GUI of the guest)
4. `adb install -r <path to the apk file on the host>`
5. `adb kill-server`
