# Surface Pro 11 (Intel) — recovering Bluetooth after a failed suspend

On this machine `btintel_pcie` sometimes misses the controller's alive interrupt
when entering D3. The driver's fallback then consults a cache that only the
interrupt handler updates, so the check always fails, all retries are exhausted,
and `btintel_pcie_suspend()` returns `-EBUSY`. One device returning `-EBUSY`
aborts the whole system suspend: the machine simply does not sleep.

    Bluetooth: hci0: Timeout (200 ms) on alive interrupt for D2 entry, retry count 0
    Bluetooth: hci0: Timeout (200 ms) on alive interrupt for D2 entry, retry count 1
    Bluetooth: hci0: Timeout (200 ms) on alive interrupt for D2 entry, retry count 2
    btintel_pcie 0000:00:14.7: PM: failed to suspend async: error -16
    PM: Some devices failed to suspend, or early wake event detected

**This is a workaround, not a fix.** Two kernel patches addressing it are on
linux-bluetooth as of 2026-09-03, fixing different halves of the same failure.
Once either lands, delete this.

The failure is intermittent: measured here at roughly one suspend in six, and
eight consecutive clean suspends prove nothing.

## Where the files go

    /usr/local/bin/sp11-bt-recover
    /etc/systemd/system/sp11-bt-recover.service

    chmod +x /usr/local/bin/sp11-bt-recover
    systemctl daemon-reload
    systemctl enable sp11-bt-recover.service

## What it does, and two things worth knowing

It watches for the controller wedging and rebinds it, rather than unloading the
module from a sleep hook, which is the usual advice and which loses the adapter
for the rest of the session.

- It polls rather than sleeping a fixed interval. An earlier version slept 20
  seconds and was wrong on both sides: too long when the controller was ready
  early, too short when it was not.
- The unit is `Type=simple`, not `Type=oneshot`. As a oneshot it was considered
  finished the moment it started, and systemd tore it down before it had done
  anything.
