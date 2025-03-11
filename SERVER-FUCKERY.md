# Server Fuckery

## Now what

Based off of the [laptop one](ELUK-XVI-G1.md), I'm making an offshoot for shit that's applicable to "server"-ish uses

## Why?

The GPU blacklisting on dual-GPU setups is bullshit.

## Gimme the details

aight here we go.

### Distro

As of this writing, I pulled this off with PikaOS 4, base kernel. They have the ACS patchset, it helps with the passthrough.

### Modprobe config

This goes into the `/etc/modprobe.d/nvidia.conf` config file, right at the top:

```
# PCI passthrough setup
softdep nvidia pre: vfio-pci
softdep nvidia* pre: vfio-pci
```

This forces the module evaluation to happen BEFORE nVidia even has a chance to load. That lets vfio-pci grab the device before nVidia's bollocks have a chance to, giving vfio full passthrough control.

### Bootloader config

This goes into `/boot/refind_linux.com`

```
"Bootable emu acs"              "amd_pstate=active nowatchdog initrd=\booster.img-%v-pikaos amd_prefcore=enable root=UUID=<YOUR_ROOT_PARTITION_UUID> quiet splash amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction vfio-pci.ids=10de:2206,10de:1aef ---"
```

This:
* Specifically applies the pci.ids for my 3080 to the PCIe passthrough
* Doesn't yeet nVidia because I'm going to need it for nvenc transcode using the GTX 1070 Ti and RTX 3060 Ti (when I replace the 1070 Ti with it)

### Booster config

This goes into the `/etc/booster.yaml` config file:

```
[...]
modules_force_load: usbhid,hid_generic,loop,usb_storage,vfio_pci,vfio,vfio_iommu_type1
modules: usbhid,hid_generic,loop,usb_storage,vfio_pci,vfio,vfio_iommu_type1
[...]
```

Specifically, the `vfio_pci,vfio,vfio_iommu_type1` part that forces vfio load. This forces vfio to load on boot through initramfs, and forces the modules to be packed INTO initramfs.

### QEMU config

There are a few special bits

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
[...]
  <features>
    <acpi/>
    <apic/>
    <hyperv mode='custom'>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vpindex state='on'/>
      <synic state='on'/>
      <stimer state='on'>
        <direct state='on'/>
      </stimer>
      <reset state='on'/>
      <vendor_id state='on' value='1234567890ab'/>
      <frequencies state='on'/>
      <reenlightenment state='on'/>
      <tlbflush state='on'/>
      <ipi state='on'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    <vmport state='off'/>
    <smm state='on'/>
    <ioapic driver='kvm'/>
  </features>
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="6" threads="2"/>
    <cache mode="passthrough"/>
    <feature policy="require" name="topoext"/>
  </cpu>
  <clock offset='localtime'>
    <timer name='rtc' present='no' tickpolicy='catchup'/>
    <timer name='pit' present='no' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='kvmclock' present='no'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
  [...]
  <qemu:override>
    <qemu:device alias='hostdev0'>
      <qemu:frontend>
        <qemu:property name="x-pci-sub-vendor-id" type="unsigned" value="14402"/>
        <qemu:property name="x-pci-sub-device-id" type="unsigned" value="14487"/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
</domain>
```
Generally speaking:
* The `<features>` section enables as many "accelerators" as possible but also masks the emulation as much as possible
* The `<clock>` section I just copied from the net. might be helping.
* The `<qemu:override>` is VERY important because it sets the right "SUBSYSTEM" device identifiers used in Windows driver detection bollocks
* ***UPDATE***: The `<cpu>` section was edited back to re-enable the hypervisor AND to have a reasonable topology, instead of letting QEMU decide
  * This may have been a problem, as QEMU was deducing it should show the CPU as multiple *sockets*, possibly enabling weird cross-socket stuff in Windows

#### Okay but what the fuck are those values

Long story short: the `x-pci-sub-vendor-id` and `x-pci-sub-device-id` are identifiers that *need* to be known,
represent the card make's subsystem identifier(s); they can be extracted using the following command in Linux:
```
lspci -nnk -s 10:00
```

In the case of ***my*** RTX 3080:
* In my system, the nVidia card is hooked up to PCIe bus `10:00`, matching the command's last option
* The Vendor ID was `3842`, and the Device ID was `3897` 
  * Converted to decimal, these become `14402` and `14487`

### Virtual Monitor

In order to tell the GPU to f*ck off and just have a fake screen that forces a decent-sized framebuffer for [Steam, Sunshine, etc] Streaming tech:

[https://github.com/itsmikethetech/Virtual-Display-Driver](https://github.com/itsmikethetech/Virtual-Display-Driver)

This driver will fake an actual monitor and force the system to have a fair resolution on boot. Very practical for streaming
setups if you don't want the smoke.

I won't help here because everyone's reqs are different. I just played with the option.txt to force a specific resolution and
refresh rate that I felt would be good for my own setup.

## Aight, beyond that?

Few things to keep in mind.
* Use VirtIO as much as possible
* If passthroughing locally, look into `Looking Glass`. If not, yer on your own for now, I can't bring myself to give a shit.

