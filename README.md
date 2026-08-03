# bc250-acpi-fix

SSDT overrides exposing processor C-states and P-states on the AMD BC-250.

The stock firmware does not expose `_CST` or `_PSS` objects, so Linux has no
cpuidle states beyond POLL/C1 and no cpufreq interface at all. SSDT-CST adds
three C-states, SSDT-PST adds eight P-states from 800 MHz to 3.2 GHz.

## Installation

The tables are injected through the initrd using the kernel's
[ACPI table upgrade mechanism](https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html).
This requires `CONFIG_ACPI_TABLE_UPGRADE`, which major distribution kernels
enable.

Prebuilt `.aml` files are included, so you can skip ahead to building the CPIO
archive. To compile them yourself:

<pre>
iasl -p SSDT-CST SSDT-CST.dsl
iasl -p SSDT-PST SSDT-PST.dsl
</pre>

Build a CPIO archive:

<pre>
mkdir -p kernel/firmware/acpi
cp SSDT-CST.aml SSDT-PST.aml kernel/firmware/acpi/
find kernel | cpio -H newc --create > acpi_override
</pre>

### GRUB

Copy the archive to `/boot` and prepend it to the initrd. Note `tee -a`, not
`tee`:

<pre>
sudo cp acpi_override /boot
echo 'GRUB_EARLY_INITRD_LINUX_CUSTOM="acpi_override"' | sudo tee -a /etc/default/grub
sudo update-grub
</pre>

### systemd-boot

Copy the archive to the ESP and add it as the first `initrd` line in your entry
under `/boot/loader/entries/`. The uncompressed archive must come before the
real initramfs:

<pre>
sudo cp acpi_override /boot/acpi_override
</pre>

<pre>
title   Linux
linux   /vmlinuz-linux
initrd  /acpi_override
initrd  /initramfs-linux.img
options root=...
</pre>

Paths are relative to the ESP. Note that some distributions regenerate entries
on kernel updates, which may drop the `initrd` line. Check after upgrading.

## Verifying

Both tables should appear in the boot log:

<pre>sudo dmesg | grep -iE 'ACPI.*(SSDT|Table Upgrade)'</pre>

<pre>
ACPI: SSDT ACPI table found in initrd [kernel/firmware/acpi/SSDT-CST.aml][0x30e]
ACPI: SSDT ACPI table found in initrd [kernel/firmware/acpi/SSDT-PST.aml][0x39e]
ACPI: Table Upgrade: install [SSDT-  HACK-  P_CST3]
ACPI: Table Upgrade: install [SSDT-  HACK- PSTATES]
</pre>

Check that the P-states are present:

<pre>cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_frequencies</pre>

<pre>3200000 2550000 2325000 1960000 1820000 1600000 1271000 800000</pre>

Then check the C-states:

<pre>cpupower idle-info</pre>

At idle, cores should sit mostly in C2 and drop to around 800 MHz, boosting
above 3.2 GHz under load.

Linux does not report the boost clock (~3.5 GHz), so use `cpupower monitor`
to get the actual hardware counters.

<pre>sudo cpupower monitor</pre>
