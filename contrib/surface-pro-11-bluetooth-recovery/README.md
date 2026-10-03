# Surface Pro 11 (Intel) — recovering Bluetooth that comes up dead at boot

On this machine the Bluetooth controller (Intel BE201, `btintel_pcie`) sometimes
comes up dead. A mailbox interrupt arrives during the firmware download before
the driver is waiting for it, the driver's one retry fails the same way, and the
adapter is left DOWN with the address 00:00:00:00:00:00:

    Bluetooth: hci0: Received hw exception interrupt
    Bluetooth: hci0: Unsupported cnvi 0x00000000
    Bluetooth: hci0: Controller in error state

On Ubuntu's 7.0.0-34 that was 9 boots in 28 here. Reloading the module brings
the controller back. This service does that at boot, and only that: it has
nothing to do with suspend.

**This is a workaround, not a fix, and you may not need it.** The race is in
Ubuntu's kernel, not mainline: v7.0 built with Ubuntu's own configuration came
up clean on 41 boots of 41, as did the 7.2 and 7.3 kernels built here. It is
reported as [Launchpad bug 2161900](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/2161900).
Look for "Controller in error state" in your own boot log before installing it.

## Where the files go

    /usr/local/sbin/sp11-bt-recover
    /etc/systemd/system/sp11-bt-recover.service

    chmod +x /usr/local/sbin/sp11-bt-recover
    systemctl daemon-reload
    systemctl enable sp11-bt-recover.service

It uses `hciconfig`, which some distributions ship in a separate package of
deprecated BlueZ tools. `DEV` at the top of the script is the controller's PCI
address, `0000:00:14.7` here; `lspci -D | grep -i bluetooth` shows yours.

## Why it waits

The probe returns at once and the firmware setup runs after it, with the address
at zero throughout. A failing setup takes about 8 s to give up. Unloading the
module in the middle of it frees the interrupts under it, and the next probe
then fails with -62 and leaves no adapter at all. An earlier version of this
script reloaded on the first zero address and did exactly that on every boot it
fired.

So it treats a zero address as dead only after 15 s, and reloads at once when no
driver is bound. If the first reload gives no address it tries once more, and
if neither worked it exits non-zero, so the unit shows in `systemctl --failed`.
Keep the wait if you adapt it.

## Provenance

Surface Pro for Business 11th Edition with Intel, Core Ultra 7 268V, Ubuntu
26.04, kernel 7.0.0-34. Ten warm reboots on 2026-09-29, with the wait then at
10 s: seven healthy boots, where it did nothing, and three dead ones, all
recovered, one of them on the second reload.
