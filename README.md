# syl's radeon ovmf passthrough setup thing (for vr gaming)

this is just my setup for radeon gpu passthrough for vm stuff. i CANNOT guarantee that this will work for you, so do as you will with this. i am not responsible for anything you do wrong.

**this setup is ran under an arch system,** so you will probably have the best luck running this on a system based on arch (or arch itself). *there are no guarantees that it'll work for you*

(plus a lot of this guide is based on the [archwiki entry of ovmf passthrough](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF))

your mileage may vary based on the system you have, as well. personally, this setup is based on these specs:
- **Motherboard:** ASUS TUF Gaming B550-PLUS WiFi II
- **CPU:** Ryzen 7 5700G (8c / 16t)
- **GPU 1:** Radeon Vega 8 Integrated Graphics (part of the CPU)
- **GPU 2:** XFX Speedster SWFT 210 (Radeon RX 6600)
- **RAM:** 64G DDR4 system memory @ 3200mhz. i recommend that you have at least 32GB installed.
- **WLAN:** MediaTek MT7921L (WiFi 6) -- built into motherboard
- **WLAN:** Intel AX210 (WiFi 6E) -- installed in pcie slot
- i also have a [pcie usb 3.0 card](https://www.amazon.com/FebSmart-Self-Powered-Technology-No-Additional-FS-U2-Pro/dp/B071P5C6CS/) that i bought from amazon that i passthrough to my vms, but this is not required. i only passed this through so i can plug in additional vive trackers for VR gameplay.

### system setup

firstly, considering that we're going into virtualization, you want to make sure that you have AMD's **iommu** and **virtualization** enabled in your bios.

my configuration utilizes both my integrated graphics and my discrete graphics -- if you'd like to replicate my setup, **make sure that you have your integrated graphics enabled in your bios.** to find this setting in a bios similar to mine, you will want to go to advanced settings, find NB (northbridge) configuration, and enable something along the lines of "IGFX Multi-Monitor" -- make sure that the primary video device is left on your integrated graphics option.

and now, onto my linux configuration

to make a lot of this easier, i would recommend setting up the **[chaotic-aur](https://aur.chaotic.cx/)** repository for your system. the chaotic-aur is a repository filled with prebuilt packages filled based on user requests, and they keep all of their stuff open source & auditable (which is a HUGE plus)

personally, i'm running my system under the **[linux-tkg-cfs](https://github.com/Frogging-Family/linux-tkg)** kernel (which is also built on & can be installed through the chaotic-aur repository) -- in my (very half-assed & limited) testing, the cpu scheduler included in this kernel seems to maintain the most stability when limited to a smaller amount of cpu cores. this kernel includes all patches required for **vfio** & **pcie acs override patch** (explained later) -- you can also run this setup using the `linux-zen` kernel.

### kernel boot parameters commandline thing

okay so this one is probably going to be the most difficult to explain, but i'll do my best. most of this is configuration copied from workarounds found in bug reports lol

my kernel boot parameters: `rw quiet pcie_acs_override=downstream,multifunction vfio-pci.ids=1912:0015,1002:73ff,1002:ab28,8086:2725,10de:0fc1,10de:0e1b nofb video=vesafb:off,efifb:off vga=normal initcall_blacklist=sysfb_init default_hugepagesz=1G hugepagesz=1G hugepages=32 nohz_full=0-5,8-13 rcu_nocbs=0-5,8-13 iommu=pt preempt=voluntary`

here's my best shot at a breakdown

> `pcie_acs_override=downstream,multifunction`: unfortunately with my motherboard, only the first pcie slot has its own isolated iommu group. every slot past that is all in one combined iommu group, which is an issue considering that i want to pass through my usb card along with my gpu. 
> 
> this kernel parameter, as described by the [archwiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)), attempts to break up as many devices as possible and isolating them in their own iommu groups. 
> 
> to my best understanding, this can also have negative impacts on your system. if a device makes an attempt to contact another device that was *supposed* to be in the same iommu group while this patch is applied, your system can hang and / or completely crash because the destination is now unreachable. in my case, everything has been fine and i've ran into no issues regarding this. *attempt at your own risk*

> `vfio-pci.ids=1912:0015,1002:73ff,1002:ab28,8086:2725`: these are the devices i want to passthrough to my virtual machines **and leave inaccessible on my linux host.** *in my case, i HAVE to include my discrete graphics' ID here, because if it is left out, the vm would either not start because it cannot be removed from the linux graphics driver, or the graphics card would not initialize in the guest because it was not started with the same driver as the guest system.*
> 
> you can find the IDs of your pci devices through the output of `lspci -nn`
> 
> - `1912:0015`: the ID of the usb card i bought from amazon. i leave this inaccessible so only the vm can access my vr headset, and my vr headset audio outputs are not visible on the host system.
> - `1002:73ff` & `1002:ab28`: the IDs of my graphics card and the audio device from the graphics card respectively. **make sure you have both the VGA and audio devices found in your `lspci` output listed, as you'll likely run into issues if you don't.**
> - `8086:2725`: my ax210 wifi card -- i passthrough this wifi card so i'm not bottlenecked by the latency / inconsistencies of the built in libvirt networking (this could be just me, your performance may be better than mine)

> `nofb video=vesafb:off,efifb:off vga=normal initcall_blacklist=sysfb_init`: a collection of parameters that (to my understanding) ensures that the kernel does not modify the boot framebuffer / utilize the graphics in any way, essentially ensuring that the aforementioned "it was not started with the same driver as the guest system" issue does not appear.

> `default_hugepagesz=1G hugepagesz=1G hugepages=32`: 32GB hugepages.
> 
> **to my understanding,** hugepages takes a static amount of memory at boot, and allows processes that explicitly request access to the allocated hugepages (in our use case, qemu) use the statically allocated memory, enabling better guest memory performance

> `nohz_full=6-7,14-15 rcu_nocbs=6-7,14-15`: this parameter instructs the kernel to reduce the amount of processing on these specific cpu threads.
> 
> in my setup case, i dedicate the cpu threads 0-5 and 8-13 (total of 12) to my virtual machines. while we already isolate those cpus through systemd (as later described), there is still processing noise that cannot be managed by systemd. this kernel parameter makes an attempt to minimize said noise as much as it can.

> `iommu=pt`: gives a specific set of instructions to the kernel when managing iommu -- i cannot give a better description myself, but i've heard that this does help boost performance graphics-wise.

> `preempt=voluntary`: i added this parameter in hopes that it would help mounting my discrete graphics after a virtual machine shutdown, but it did not in fact do that. you may have better luck -- i'm always open to suggestions!

### software!

as described in the [archwiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Configuring_libvirt), you'll want to install `qemu-desktop`, `libvirt`, `edk2-ovmf`, `virt-manager` and `dnsmasq`.

**libvirtd is ran system-level**, meaning that you will need to provide superuser / root access to virt-manager every time you want to manage your virtual machines. you can actually work around this by adding your user to the groups `libvirt` and `kvm` (create these groups with `groupadd` if they do not exist)

after installing those packages and giving your user access to libvirt, make sure to enable & start the `libvirtd.service` and `virtlogd.socket` systemd components

oh, and if you want to use networking through libvirtd (as in you're not passing through a wifi / ethernet card), run the following as root:

```bash
# virsh net-autostart default
# virsh net-start default
```

### virtual machine configuration

**now this is where we get started.**

open virt-manager, and go to Edit > Preferences. **make sure "Enable XML editing" is enabled.**

create a new virtual machine, name it whatever you'd like, but make sure to leave "Customize configuration before install" enabled. this is where you're going to edit the XML of the virtual machine.

make sure to passthrough the pcie devices by going to "Add Hardware" > "PCI devices" and add both the "VGA device" and audio device listed for your specific graphics card.

configure the virtual machine to your liking from here. for reference, my windows virtual machine configuration file can be found [here](https://github.com/sylviiu/radeon-ovmf-passthrough/blob/main/vm.xml)

a few important parts i feel like is necessary to mention:

#### hugepages usage:

```xml
  <memoryBacking>
    <hugepages/>
  </memoryBacking>
```

if you set up hugepages, make sure to add this inside your `<domain>` tag. this instructs libvirt / qemu to utilize your hugepages that you set up.

#### cpu pinning:

```xml
  <vcpu placement="static">12</vcpu>
  <cputune>
    <vcpupin vcpu="0" cpuset="2"/>
    <vcpupin vcpu="1" cpuset="10"/>
    <vcpupin vcpu="2" cpuset="3"/>
    <vcpupin vcpu="3" cpuset="11"/>
    <vcpupin vcpu="4" cpuset="4"/>
    <vcpupin vcpu="5" cpuset="12"/>
    <vcpupin vcpu="6" cpuset="5"/>
    <vcpupin vcpu="7" cpuset="13"/>
    <vcpupin vcpu="8" cpuset="6"/>
    <vcpupin vcpu="9" cpuset="14"/>
    <vcpupin vcpu="10" cpuset="7"/>
    <vcpupin vcpu="11" cpuset="15"/>
    <emulatorpin cpuset="0,6"/>
  </cputune>
```

**[I HIGHLY SUGGEST YOU FOLLOW THE INSTRUCTIONS LISTED ON THE ARCHWIKI](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#CPU_pinning); RESULTS VARY SYSTEM BY SYSTEM**

#### qemu `<features>` tag & general cpu configuration

```xml
  <features>
    <acpi/>
    <apic/>
    <hyperv mode="passthrough">
      <relaxed state="on"/>
      <vapic state="on"/>
      <spinlocks state="on" retries="8191"/>
      <vpindex state="on"/>
      <runtime state="on"/>
      <synic state="on"/>
      <stimer state="on"/>
      <reset state="off"/>
      <vendor_id state="on" value="randomid"/>
      <frequencies state="on"/>
    </hyperv>
    <kvm>
      <hidden state="on"/>
    </kvm>
    <vmport state="off"/>
    <smm state="on"/>
    <ioapic driver="kvm"/>
  </features>
  <clock offset="localtime">
    <timer name="rtc" present="no" tickpolicy="catchup"/>
    <timer name="pit" present="no" tickpolicy="delay"/>
    <timer name="hpet" present="no"/>
    <timer name="kvmclock" present="no"/>
    <timer name="hypervclock" present="yes"/>
  </clock>
```

i generally use these tags to help disguise the virtual machine as raw hardware, but...

```xml
  <cpu mode="host-passthrough" check="none" migratable="on">
    <topology sockets="1" dies="1" cores="6" threads="2"/>
    <feature policy="disable" name="rdtscp"/>
    <feature policy="require" name="topoext"/>
    <feature policy="disable" name="hypervisor"/>
    <feature policy="require" name="invtsc"/>
  </cpu>
```

will give you the best shot at hiding your virtualized state from the guest operating system. *with these tags, task manager does not report that the system is virtualized and shows the correct cpu cache sizes*
