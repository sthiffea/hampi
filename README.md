# This is how I build my Raspberry Pi 4 for ham use:

## Flash Raspbian Image

Use the full raspbian image
```
Raspberry Pi OS (32-bit) with desktop and recommended software
Image with desktop and recommended software based on Debian Buster
Version:August 2020
Release date:2020-08-20
Kernel version:5.4
Size:2531 MB
2020-08-20-raspios-buster-armhf-full.zip
```
## Configure wifi and enable ssh on first boot
Create file ```wpa_supplicant.conf``` on boot partition.

Content:
```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=CA

network={
 ssid="candy"
 psk="###################"
}
```
Create empy file named ```ssh``` on boot partition



