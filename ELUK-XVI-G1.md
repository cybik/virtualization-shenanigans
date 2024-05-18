# Eluktronics Prometheus 16 Gen.1

## What the---

Yeah that computer is literally out of print. It wasn't good and I chose wrong, but hey, the hardware itself is palatable:

* DDR4 support
* 2 NVMe slots
* Up to 165hz LCD
* LED controls (I implemented the driver in Linux [it sucks {shut up} ])
* AMD iGPU
* nVidia RTX 3080 Mobile dGPU

## You did what with it?

I had the notion of trying to asspull a PCIe passthrough. It kinda worked?

* PCIe passthrough of the Laptop's dGPU for virtualization
* Blacklisting nVidia drivers for good measure

## Gimme the details

aight here we go.

### Distro

As of this writing, I pulled this off with PikaOS 3, base kernel. They have the ACS patchset, it helps with the passthrough.

### Bootloader config

This goes into `/boot/refind_linux.com`
```
"Bootable emu acs"              "amd_pstate=active nowatchdog initrd=\booster.img-%v-pikaos amd_prefcore=enable root=UUID=<YOUR_ROOT_PARTITION_UUID> quiet splash module_blacklist=nvidia modprobe.blacklist=nvidia amd_iommu=on iommu=pt pcie_acs_override=downstream,multifunction vfio-pci.ids=10de:249c,10de:228b ---"
```

This:
* Enables as much AMD bullshit as possible
* Yeets the nVidia drivers (`modprobe.blacklist` and `module_blacklist`)

### Special QEMU bits

#### Fuckoff Battery State

The nVidia driver in Windows will be a bit of a party-crasher because if it detects it's on a laptop, it checks for a Battery and/or the state of AC power.

To remedy this: [Gentoo Wiki about a battery.dat file to prepare](https://wiki.gentoo.org/wiki/Nvidia_GPU_passthrough_with_QEMU_on_Lenovo_ThinkPad_P53#Add_fake_battery)

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
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <feature policy='disable' name='hypervisor'/>
    <feature policy='require' name='topoext'/> <!-- for AMD only -->
  </cpu>
  <clock offset='localtime'>
    <timer name='rtc' present='no' tickpolicy='catchup'/>
    <timer name='pit' present='no' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='kvmclock' present='no'/>
    <timer name='hypervclock' present='yes'/>
  </clock>
  [...]
  <qemu:commandline>
    <qemu:arg value='-acpitable'/>
    <qemu:arg value='file=/opt/qemu/battery.dat'/>
  </qemu:commandline>
  <qemu:override>
    <qemu:device alias='hostdev0'>
      <qemu:frontend>
        <qemu:property name='x-pci-sub-vendor-id' type='unsigned' value='5421'/>
        <qemu:property name='x-pci-sub-device-id' type='unsigned' value='4919'/>
      </qemu:frontend>
    </qemu:device>
  </qemu:override>
[...]
```
Generally speaking:
* The `<features>` section enables as many "accelerators" as possible but also masks the emulation as much as possible
* The `<clock>` section I just copied from the net. might be helping.
* The `<qemu:commandline>` section is to tell the nVidia driver to stop being itself
* The `<qemu:override>` is VERY important because it sets the right "SUBSYSTEM" device identifiers used in Windows driver detection bollocks

#### Okay but what the fuck are those values

Long story short: the `x-pci-sub-vendor-id` and `x-pci-sub-device-id` can be extracted from `lspci -nnk -s 01:00` 
(in my system, the nVidia stuff is hooked up to PCIe bus `01:00`), and represent the ODM's subsystem identifier, 
not the actal devices'.

In the case of ***my*** Prometheus XVI, the Vendor ID was `152d`, and the Device ID was `1337` (heh). Converted
to decimal, these would become `5421` and `4919`.

### Virtual Monitor

In order to tell the GPU to f*ck off and just have a fake screen that forces a decent-sized framebuffer for Steam Streaming:

[https://github.com/itsmikethetech/Virtual-Display-Driver](https://github.com/itsmikethetech/Virtual-Display-Driver)

This driver will fake an actual monitor and force the system to have a fair resolution on boot. Very practical for streaming
setups if you don't want the smoke.

I won't help here because everyone's reqs are different. I just played with the option.txt to force a specific resolution and
refresh rate that I felt would be good for my own setup.

## Aight, beyond that?

Few things to keep in mind.
* Use VirtIO as much as possible
* If passthroughing locally, look into `Looking Glass`. If not, yer on your own for now, I can't bring myself to give a shit.

