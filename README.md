# This is how I build my Raspberry Pi 4 for ham use:

## Flash Raspbian Image

- Use the full raspbian image
    ```
    Raspberry Pi OS (32-bit) with desktop and recommended software
    Image with desktop and recommended software based on Debian Buster
    Version:August 2020
    Release date:2020-08-20
    Kernel version:5.4
    Size:2531 MB
    2020-08-20-raspios-buster-armhf-full.zip
    ```
---
## Configure wifi and enable ssh on first boot
- Create file ```wpa_supplicant.conf``` on boot partition.\
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
- Create empy file named ```ssh``` on boot partition
---
## SSH into pi, enable VNC
- ```ssh pi@raspberrypi```\
  (Default hostname, could resolv as raspberrypi.local or raspberrypi.\<localdomain\>)
- Password: ```raspberry```
- ```sudo apt update && sudo apt upgrade -y```
- ```sudo apt clean```
- ```sudo apt autoremove```
- ```sudo reboot```
- ```sudo raspi-config```
    - Change hostname to hampi
    - Enable VNC (Under Interfacing Options)
    - Exit
- ```sudo vi /boot/config.txt```
    ```
    disable_overscan=1

    hdmi_group=2
    hdmi_mode=47
    ```
- Reboot
---
## Connect VNC and configure a few things
- User RealVNC client to connect to hampi.\<localdomain\>
- Go through Setup Wizard
    - Canada
    - Canadian English
    - Toronto
    - Check both check boxes
    - Change User password
    - Select Wifi Network
    - Restart
- Preference, Screen configuration, use full screen resolution
- In RealVNC menu, under Licensing... configure cloud connection
---
## Git
- ```git config --global user.name "Simon"```\
```git config --global user.email "stiffo@gmail.com"```
- git clone https://github.com/sthiffea/hampi.git
    - Use ```git commit -m "reasons"``` to commit changes
    - Use ```git add <filename>``` to add new files
    - Use git push to upload to github
---
## Stuff
- Set background to the image in hampi
- ```sudo apt install pavucontrol```
- reboot
---
## Hotspot
### Taken from https://www.raspberryconnect.com/projects/65-raspberrypi-hotspot-accesspoints/157-raspberry-pi-auto-wifi-hotspot-switch-internet
- ```sudo apt-get install -y hostapd dnsmasq```
- Disable autostart of services:
    ```
    sudo systemctl unmask hostapd
    sudo systemctl disable hostapd
    sudo systemctl disable dnsmasq
    ```
- ```sudo vi /etc/hostapd/hostapd.conf```
    ```
    #2.4GHz setup wifi 80211 b,g,n
    interface=wlan0
    driver=nl80211
    ssid=hampi
    hw_mode=g
    channel=8
    wmm_enabled=0
    macaddr_acl=0
    auth_algs=1
    ignore_broadcast_ssid=0
    wpa=2
    wpa_passphrase=####WPA PASSCODE###
    wpa_key_mgmt=WPA-PSK
    wpa_pairwise=CCMP TKIP
    rsn_pairwise=CCMP

    #80211n - Change GB to your WiFi country code
    country_code=CA
    ieee80211n=1
    ieee80211d=1
    ```
- ```sudo vi /etc/default/hostapd```
    ```
    DAEMON_CONF="/etc/hostapd/hostapd.conf"
    ```
- Add to ```sudo vi /etc/dnsmasq.conf```
    ```
    #AutoHotspot config
    interface=wlan0
    bind-dynamic 
    server=8.8.8.8
    domain-needed
    bogus-priv
    dhcp-range=192.168.50.150,192.168.50.200,12h
    ```
- ```sudo vi /etc/network/interfaces``` shoud be:
    ```
    # interfaces(5) file used by ifup(8) and ifdown(8)
    # Please note that this file is written to be used with dhcpcd
    # For static IP, consult /etc/dhcpcd.conf and 'man dhcpcd.conf'
    # Include files from /etc/network/interfaces.d:
    source-directory /etc/network/interfaces.d
    ```
- ```sudo vi /etc/sysctl.conf``` uncomment:
    ```
    net.ipv4.ip_forward=1
    ```
- ```sudo vi /etc/dhcpcd.conf``` add at the bottom:
    ```
    nohook wpa_supplicant
    ```
- ```sudo vi /etc/systemd/system/autohotspot.service```
    ```
    [Unit]
    Description=Automatically generates an internet Hotspot when a valid ssid is not in range
    After=multi-user.target
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/bin/autohotspotN
    [Install]
    WantedBy=multi-user.target
    ```
- ```sudo systemctl enable autohotspot.service```
- ```sudo vi /usr/bin/autohotspotN```
    ```
    #!/bin/bash
    #version 0.961-N/HS-I

    #You may share this script on the condition a reference to RaspberryConnect.com 
    #must be included in copies or derivatives of this script. 

    #Network Wifi & Hotspot with Internet
    #A script to switch between a wifi network and an Internet routed Hotspot
    #A Raspberry Pi with a network port required for Internet in hotspot mode.
    #Works at startup or with a seperate timer or manually without a reboot
    #Other setup required find out more at
    #http://www.raspberryconnect.com

    wifidev="wlan0" #device name to use. Default is wlan0.
    ethdev="eth0" #Ethernet port to use with IP tables
    #use the command: iw dev ,to see wifi interface name 

    IFSdef=$IFS
    cnt=0
    #These four lines capture the wifi networks the RPi is setup to use
    wpassid=$(awk '/ssid="/{ print $0 }' /etc/wpa_supplicant/wpa_supplicant.conf | awk -F'ssid=' '{ print $2 }' | sed 's/\r//g'| awk 'BEGIN{ORS=","} {print}' | sed 's/\"/''/g' | sed 's/,$//')
    IFS=","
    ssids=($wpassid)
    IFS=$IFSdef #reset back to defaults


    #Note:If you only want to check for certain SSIDs
    #Remove the # in in front of ssids=('mySSID1'.... below and put a # infront of all four lines above
    # separated by a space, eg ('mySSID1' 'mySSID2')
    #ssids=('mySSID1' 'mySSID2' 'mySSID3')

    #Enter the Routers Mac Addresses for hidden SSIDs, seperated by spaces ie 
    #( '11:22:33:44:55:66' 'aa:bb:cc:dd:ee:ff' ) 
    mac=()

    ssidsmac=("${ssids[@]}" "${mac[@]}") #combines ssid and MAC for checking

    createAdHocNetwork()
    {
        echo "Creating Hotspot"
        ip link set dev "$wifidev" down
        ip a add 192.168.50.5/24 brd + dev "$wifidev"
        ip link set dev "$wifidev" up
        dhcpcd -k "$wifidev" >/dev/null 2>&1
        iptables -t nat -A POSTROUTING -o "$ethdev" -j MASQUERADE
        iptables -A FORWARD -i "$ethdev" -o "$wifidev" -m state --state RELATED,ESTABLISHED -j ACCEPT
        iptables -A FORWARD -i "$wifidev" -o "$ethdev" -j ACCEPT
        systemctl start dnsmasq
        systemctl start hostapd
        echo 1 > /proc/sys/net/ipv4/ip_forward
    }

    KillHotspot()
    {
        echo "Shutting Down Hotspot"
        ip link set dev "$wifidev" down
        systemctl stop hostapd
        systemctl stop dnsmasq
        iptables -D FORWARD -i "$ethdev" -o "$wifidev" -m state --state RELATED,ESTABLISHED -j ACCEPT
        iptables -D FORWARD -i "$wifidev" -o "$ethdev" -j ACCEPT
        echo 0 > /proc/sys/net/ipv4/ip_forward
        ip addr flush dev "$wifidev"
        ip link set dev "$wifidev" up
        dhcpcd  -n "$wifidev" >/dev/null 2>&1
    }

    ChkWifiUp()
    {
        echo "Checking WiFi connection ok"
            sleep 20 #give time for connection to be completed to router
        if ! wpa_cli -i "$wifidev" status | grep 'ip_address' >/dev/null 2>&1
            then #Failed to connect to wifi (check your wifi settings, password etc)
            echo 'Wifi failed to connect, falling back to Hotspot.'
                wpa_cli terminate "$wifidev" >/dev/null 2>&1
            createAdHocNetwork
        fi
    }

    chksys()
    {
        #After some system updates hostapd gets masked using Raspbian Buster, and above. This checks and fixes  
        #the issue and also checks dnsmasq is ok so the hotspot can be generated.
        #Check Hostapd is unmasked and disabled
        if systemctl -all list-unit-files hostapd.service | grep "hostapd.service masked" >/dev/null 2>&1 ;then
        systemctl unmask hostapd.service >/dev/null 2>&1
        fi
        if systemctl -all list-unit-files hostapd.service | grep "hostapd.service enabled" >/dev/null 2>&1 ;then
        systemctl disable hostapd.service >/dev/null 2>&1
        systemctl stop hostapd >/dev/null 2>&1
        fi
        #Check dnsmasq is disabled
        if systemctl -all list-unit-files dnsmasq.service | grep "dnsmasq.service masked" >/dev/null 2>&1 ;then
        systemctl unmask dnsmasq >/dev/null 2>&1
        fi
        if systemctl -all list-unit-files dnsmasq.service | grep "dnsmasq.service enabled" >/dev/null 2>&1 ;then
        systemctl disable dnsmasq >/dev/null 2>&1
        systemctl stop dnsmasq >/dev/null 2>&1
        fi
    }


    FindSSID()
    {
    #Check to see what SSID's and MAC addresses are in range
    ssidChk=('NoSSid')
    i=0; j=0
    until [ $i -eq 1 ] #wait for wifi if busy, usb wifi is slower.
    do
            ssidreply=$((iw dev "$wifidev" scan ap-force | egrep "^BSS|SSID:") 2>&1) >/dev/null 2>&1 
            #echo "SSid's in range: " $ssidreply
        printf '%s\n' "${ssidreply[@]}"
            echo "Device Available Check try " $j
            if (($j >= 10)); then #if busy 10 times goto hotspot
                    echo "Device busy or unavailable 10 times, going to Hotspot"
                    ssidreply=""
                    i=1
        elif echo "$ssidreply" | grep "No such device (-19)" >/dev/null 2>&1; then
                    echo "No Device Reported, try " $j
            NoDevice
            elif echo "$ssidreply" | grep "Network is down (-100)" >/dev/null 2>&1 ; then
                    echo "Network Not available, trying again" $j
                    j=$((j + 1))
                    sleep 2
        elif echo "$ssidreply" | grep "Read-only file system (-30)" >/dev/null 2>&1 ; then
            echo "Temporary Read only file system, trying again"
            j=$((j + 1))
            sleep 2
        elif echo "$ssidreply" | grep "Invalid exchange (-52)" >/dev/null 2>&1 ; then
            echo "Temporary unavailable, trying again"
            j=$((j + 1))
            sleep 2
        elif echo "$ssidreply" | grep -v "resource busy (-16)"  >/dev/null 2>&1 ; then
                echo "Device Available, checking SSid Results"
            i=1
        else #see if device not busy in 2 seconds
                    echo "Device unavailable checking again, try " $j
            j=$((j + 1))
            sleep 2
        fi
    done

    for ssid in "${ssidsmac[@]}"
    do
        if (echo "$ssidreply" | grep -F -- "$ssid") >/dev/null 2>&1
        then
            #Valid SSid found, passing to script
                echo "Valid SSID Detected, assesing Wifi status"
                ssidChk=$ssid
                return 0
        else
            #No Network found, NoSSid issued"
                echo "No SSid found, assessing WiFi status"
                ssidChk='NoSSid'
        fi
    done
    }

    NoDevice()
    {
        #if no wifi device,ie usb wifi removed, activate wifi so when it is
        #reconnected wifi to a router will be available
        echo "No wifi device connected"
        wpa_supplicant -B -i "$wifidev" -c /etc/wpa_supplicant/wpa_supplicant.conf >/dev/null 2>&1
        exit 1
    }

    chksys
    FindSSID

    #Create Hotspot or connect to valid wifi networks
    if [ "$ssidChk" != "NoSSid" ]
    then
        echo 0 > /proc/sys/net/ipv4/ip_forward #deactivate ip forwarding
        if systemctl status hostapd | grep "(running)" >/dev/null 2>&1
        then #hotspot running and ssid in range
                KillHotspot
                echo "Hotspot Deactivated, Bringing Wifi Up"
                wpa_supplicant -B -i "$wifidev" -c /etc/wpa_supplicant/wpa_supplicant.conf >/dev/null 2>&1
                ChkWifiUp
        elif { wpa_cli -i "$wifidev" status | grep 'ip_address'; } >/dev/null 2>&1
        then #Already connected
                echo "Wifi already connected to a network"
        else #ssid exists and no hotspot running connect to wifi network
                echo "Connecting to the WiFi Network"
                wpa_supplicant -B -i "$wifidev" -c /etc/wpa_supplicant/wpa_supplicant.conf >/dev/null 2>&1
                ChkWifiUp
        fi
    else #ssid or MAC address not in range
        if systemctl status hostapd | grep "(running)" >/dev/null 2>&1
        then
                echo "Hostspot already active"
        elif { wpa_cli status | grep "$wifidev"; } >/dev/null 2>&1
        then
                echo "Cleaning wifi files and Activating Hotspot"
                wpa_cli terminate >/dev/null 2>&1
                ip addr flush "$wifidev"
                ip link set dev "$wifidev" down
                rm -r /var/run/wpa_supplicant >/dev/null 2>&1
                createAdHocNetwork
        else #"No SSID, activating Hotspot"
                createAdHocNetwork
        fi
    fi
    ```
- ```sudo chmod +x /usr/bin/autohotspotN```
- Reboot

---
## GPS
- ```sudo apt -y install gpsd gpsd-clients python-gps chrony```
- ```ls -l /dev/serial/by-id```
    ```
    total 0
    lrwxrwxrwx 1 root root 13 Oct 26 19:18 usb-1a86_USB2.0-Serial-if00-port0 -> ../../ttyUSB0
    lrwxrwxrwx 1 root root 13 Oct 26 19:18 usb-u-blox_AG_-_www.u-blox.com_u-blox_7_-_GPS_GNSS_Receiver-if00 -> ../../ttyACM0
    ```
- ```sudo vi /etc/default/gpsd```
- Make / change to following setting:
    ```
    START_DAEMON=”true”
    USBAUTO=”true”
    DEVICES=”/dev/ttyACM0″
    GPSD_OPTIONS=”-n”
    ```
- ```sudo vi /etc/chrony/chrony.conf```
- Add the following line to the end of the file:
    ```
    refclock SHM 0 offset 0.5 delay 0.2 refid NMEA
    ```
- Reboot
- Check that gpsd and chronyd are active
    ```
    systemctl is-active gpsd
    systemctl is-active chronyd
    systemctl status gpsd
    ```
- Show the gps data:
    ```
    gpsmon -n
    cgps
    xgps
    ```
---
## Real Time Clock
- Enable i2c:
    - ```sudo raspi-config```
    - Enable i2c under Interfacing Options
- Check it is detected:
    ```
    pi@hampi:~ $ sudo i2cdetect -y 1
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
    00:          -- -- -- -- -- -- -- -- -- -- -- -- --
    10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
    60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- --
    70: -- -- -- -- -- -- -- --
    ```
- ```sudo modprobe rtc-ds1307```
- ```
    sudo su -
    echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    ```
- ```
    sudo hwclock -r
    date
    sudo hwclock -w
    sudo hwclock -r
    ```
- Add ```rtc-ds1307``` to ```/etc/modules```
- Add script to start RTC in ```sudo vi /etc/rc.local```
    ```
    echo ds1307 0x68 > /sys/class/i2c-adapter/i2c-1/new_device
    sudo hwclock -s
    date
    ```


---
## Fldigi & Flrig & Hamlib
- Install requirements:
    ```
    sudo apt-get install libfltk1.3-dev libjpeg9-dev libxft-dev libxinerama-dev libxcursor-dev libsndfile1-dev libsamplerate0-dev portaudio19-dev libusb-1.0-0-dev libpulse-dev texinfo
    ```
- Install flxmlrpc:
    ```
    cd Downloads
    wget http://www.w1hkj.com/files/flxmlrpc/flxmlrpc-0.1.4.tar.gz
    tar zxvf flxmlrpc-0.1.4.tar.gz
    cd flxmlrpc-0.1.4
    ./configure --enable-static
    make
    sudo make install
    sudo ldconfig
    ```
- Install hamlib:
    ```
    cd ~/Downloads
    sudo apt remove libhamlib2
    wget https://sourceforge.net/projects/hamlib/files/hamlib/3.3/hamlib-3.3.tar.gz/download -O hamlib-3.3.tar.gz
    tar zxvf hamlib-3.3.tar.gz
    cd hamlib-3.3
    ./configure --enable-static
    make
    sudo make install
    sudo ldconfig
    ```
- Install flrig:
    ```
    cd ~/Downloads
    sudo apt remove flrig
    sudo apt autoremove
    wget http://www.w1hkj.com/files/flrig/flrig-1.3.51.tar.gz
    tar zvf flrig-1.3.51.tar.gz
    cd flrig-1.3.51
    ./configure --enable-static
    make
    sudo make install
    sudo ldconfig
- Install fldigi:
    ```
    cd ~/Downloads
    wget http://www.w1hkj.com/files/fldigi/fldigi-4.1.15.tar.gz
    tar zvf fldigi-4.1.15.tar.gz
    cd fldigi-4.1.15
    ./configure --enable-static
    make
    sudo make install
    sudo ldconfig
---
## JS8Call
- ```sudo apt install libgfortran3 libqt5multimedia5 libqt5multimedia5-plugins libqt5multimediagsttools5 libqt5multimediaquick5 libqt5multimediawidgets5 libqt5qml5 libqt5quick5```
- ```wget http://files.js8call.com/2.2.0/js8call_2.2.0_armhf.deb```
- ```sudo dpkg -i js8call_2.2.0_armhf.deb```
- Config\
    ![js8call](js8call.png)\
    ![js8call-radio](js8call-radio.png)\
    ![js8call-sound](js8call-sound.png)\
    ![js8call-reply](js8call-reply.png)
---
## pat.io
- Install it
    ```
    wget https://github.com/la5nta/pat/releases/download/v0.10.0/pat_0.10.0_linux_armhf.deb
    sudo dpkg -i pat_0.10.0_linux_armhf.deb
    ```
- Run ```pat configure```
    ```
    "mycall": "VE2TFZ",
    "secure_login_password": "WINLINKPASSWORD",

    "locator": "FN35gj",

    "http_addr": "0.0.0.0:8080",

    "hamlib_rigs": {
      "ic718": {"address": "localhost:4532", "network": "tcp"}
    },

    "ardop": {
      "rig": "ic718",
    ```
- Binary files are in hampi/bin:\
    https://www.cantab.net/users/john.wiseman/Downloads/Beta/piardopc
    ```
    pi@hampi:~/hampi/bin $ arecord -l
    **** List of CAPTURE Hardware Devices ****
    card 2: CODEC [USB Audio CODEC], device 0: USB Audio [USB Audio]
      Subdevices: 1/1
      Subdevice #0: subdevice #0

- Install patmenu2
    ```
    cd ~
    git clone https://github.com/km4ack/patmenu2.git $HOME/patmenu2
    bash $HOME/patmenu2/setup

    ```




---
## Conky
- ```sudo apt install -y conky```
- ```cd```
- ```ln -sf hampi/conkyrc .conkyrc```
- ```sudo apt install -y ruby2.3```
- ```sudo gem install gpsd_client```
- ```sudo gem install maidenhead```
- ```mkdir .config/autostart```
- ```vi .config/autostart/.desktop```
    ```
    [Desktop Entry] 

    Type=Application

    Exec=conky
    ```
- Reboot