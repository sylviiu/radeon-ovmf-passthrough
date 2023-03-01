# syl's radeon ovmf passthrough setup thing (for vr gaming)

this is just my setup for radeon gpu passthrough for vm stuff. i CANNOT guarantee that this will work for you, so do as you will with this. i am not responsible for anything you do wrong.

**this setup is ran under an arch system,** so you will probably have the best luck running this on a system based on arch (or arch itself). *there are no guarantees that it'll work for you*

your mileage may vary based on the system you have, as well. personally, this setup is based on these specs:
- **Motherboard:** ASUS TUF Gaming B550-PLUS WiFi II
- **CPU:** Ryzen 7 5700G (8c / 16t)
- **GPU 1:** Radeon Vega 8 Integrated Graphics (part of the CPU)
- **GPU 2:** XFX Speedster SWFT 210 (Radeon RX 6600)
- **RAM:** 64G DDR4 system memory @ 3200mhz. i recommend that you have at least 32GB installed.
- i also have a [pcie usb 3.0 card](https://www.amazon.com/FebSmart-Self-Powered-Technology-No-Additional-FS-U2-Pro/dp/B071P5C6CS/) that i bought from amazon that i passthrough to my vms, but this is not required. i only passed this through so i can plug in additional vive trackers for VR gameplay.

### system setup

firstly, considering that we're going into virtualization, you want to make sure that you have AMD's **iommu** and **virtualization** enabled in your bios.

my configuration utilizes both my integrated graphics and my discrete graphics -- if you'd like to replicate my setup, **make sure that you have your integrated graphics enabled in your bios.** to find this setting in a bios similar to mine, you will want to go to advanced settings, find NB (northbridge) configuration, and enable something along the lines of "IGFX Multi-Monitor" -- make sure that the primary video device is left on your integrated graphics option.

and now, onto my linux configuration

to make a lot of this easier, i would recommend setting up the **[chaotic-aur](https://aur.chaotic.cx/)** repository for your system. the chaotic-aur is a repository filled with prebuilt packages filled based on user requests, and they keep all of their stuff open source & auditable (which is a HUGE plus)

personally, i'm running my system under the **[linux-tkg-cfs](https://github.com/Frogging-Family/linux-tkg)** kernel (which is also built on & can be installed through the chaotic-aur repository) -- in my (very half-assed & limited) testing, the cpu scheduler included in this kernel seems to maintain the most stability when limited to a smaller amount of cpu cores

### kernel config
