.. SPDX-License-Identifier: GPL-2.0

==============================
Rodin OEM charge-pause backend
==============================

This backend is ported from EVONIX by NEESCHAL. It uses the existing Xiaomi
navigation charge-pause policy on MediaTek MT6899 (Rodin). It does not set
input suspend or replace the vendor charger modules. Compatible vendor
charging interfaces are necessary even when the Android system is AOSP.

Build structure
===============

``CONFIG_EVONIX_OEM_BYPASS=y`` builds both objects into the kernel Image::

    drivers/misc/evonix_oem_bypass.c    request ownership and observation
    fs/sysfs/evonix_supply.c           guarded OEM attribute callbacks
    include/linux/evonix_oem_bypass.h  private built-in accessor declarations

The option requires built-in POWER_SUPPLY, OF and SYSFS. It is enabled in
capybara_defconfig, but defaults off for other configurations. There are no
new exported vendor-module symbols or private vendor structure offsets.
The backend waits for the battery supply and attaches its sysfs group. It
does not require a replacement vendor_boot image or recovery.

Control interface
=================

All attributes reside under ``/sys/class/power_supply/battery/``:

============================ =============================================
Attribute                    Meaning
============================ =============================================
bypass_charging              Read/write boolean owned request
bypass_charge                Alias of bypass_charging
bypass_charging_supported    smart_chg and charge-pump state are readable
bypass_charging_active       Sustained battery-neutral observation
bypass_charging_diagnostics  API version, backend, ownership and last error
============================ =============================================

Writable controls are mode 0644; status attributes are read-only. A production
ROM must authorize its privileged controller through ownership/SELinux rather
than expose writable controls to every application.

The apply worker submits only ``3\n`` (enable navigation with SOC threshold
zero) or ``2\n`` (disable navigation) to battery/smart_chg. These are packed
OEM commands, not charging-current values. It confirms the navigation bit
with a readback, preserves other smart-charging slots and returns actual
command errors. A failed enable/readback attempts rollback.

An existing unowned navigation limit returns EBUSY on enable; the stock getter
cannot recover that owner's original threshold. Disable releases only an
EVONIX-owned request. Reading status does not rewrite charging controls.
The accessor holds power-supply and kernfs active references while calling
the OEM attributes, including protection against concurrent sysfs removal.

Observed status
===============

The active flag requires an owned navigation pause, USB online, input suspend
off, charge-pump state stopped, primary VBUS between 4400 and 6000 mV and BMS
battery current within +/-100000 microamps for three seconds. Missing or
failing measurements do not report active. Read gaps over two seconds restart
confirmation; a privileged controller should observe approximately once per
second while requested and connected. Inactive bypass skips extra USB/current
measurements.

The OEM controller pauses battery charging and retains its normal adapter
power path. Thermal/electrical decisions remain with the vendor drivers.
The connected battery may supplement adapter power under load. The observed
active flag is not proof of battery isolation or an exact zero-current promise.
The legacy battery status string alone is not a reliable confirmation.

Persistence
===========

The kernel starts off and does not store a battery-percentage threshold.
Userspace owns saved preferences, optional thresholds and boot restoration.
Closing a Settings screen or application does not release an accepted kernel
request. Integrate a privileged controller separately; this patch does not
add a ROM Settings switch, Android service or SELinux policy.

Validation on the target ROM
============================

Before distributing, boot the built kernel with the target's vendor modules
and verify support, enable/readback, measured current and charge counter,
disable/resume, cable reconnect and reboot restoration with a real charger.
Keep adapter power and input suspension readings in the trace. A successful
kernel build is not a device test on Shinkai.
