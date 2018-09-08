---
layout: post
title: 'Lenovo ThinkPad X1 6en: Enabling S3 Sleep for Linux after Firmware Update'
---

Today a new firmware update enabled the long-missing S3 support for 6en Lenovo
ThinkPad X1. After getting the new update via:

```
sudo fwupdmgr refresh
sudo fwupdmgr get-updates
```

You should see:

```
20KHCTO1WW System Firmware has firmware updates:
GUID:                    a4b51dca-8f97-4310-8821-3330f83c9135
GUID:                    230c8b18-8d9b-53ec-838b-6cfc0383493a
ID:                      com.lenovo.ThinkPadN23ET.firmware
Update Version:          0.1.30
Update Name:             ThinkPad X1 Carbon 6th
Update Summary:          Lenovo ThinkPad X1 Carbon 6th System Firmware
Update Remote ID:        lvfs
Update Checksum:         SHA1(1a528d1b227e500bcaedbd4c7026a477c5f4a5ca)
Update Location:         https://fwupd.org/downloads/7bd315afb8ff3a610474b752265e7703e6bf1d5e-Lenovo-ThinkPad-X1Carbon6th-SystemFirmware-1.30.cab
Update Description:      Lenovo ThinkPad X1 Carbon 6th System Firmware
                         
                         CHANGES IN THIS RELEASE
                         
                         Version 1.30
                         
                         [Important updates]
                          • Nothing.
                         
                         [New functions or enhancements]
                          • Support Optimized Sleep State for Linux in ThinkPad Setup - Config - Power.
                          • (Note) "Linux"option is optimized for Linux OS, Windows user must select
                          • "Windows 10" option
                         
                         [Problem fixes]
                          • Nothing.
```

After installing the update via:

```
sudo fwupdmgr update
```

S3 will still not be enabled. To enable it fully you must enter the BIOS on
boot and change:

![alt text]({{ site.github.url }}/_img/windows10.jpg)

to

![alt text]({{ site.github.url }}/_img/linux.jpg)

Then

```
dmesg | grep S3
```

should show

```
[    0.236226] ACPI: (supports S0 S3 S4 S5)
```

Christian
