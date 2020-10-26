Install this image:

Raspberry Pi OS (32-bit) with desktop and recommended software
Image with desktop and recommended software based on Debian Buster
Version:August 2020
Release date:2020-08-20
Kernel version:5.4
Size:2531 MB
2020-08-20-raspios-buster-armhf-full.zip

Add this file to boot partition:
wpa_supplicant.conf

ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=CA

network={
 ssid="candy"
 psk="###################"
}
