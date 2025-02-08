# General 

Start with lite version of bullseye (legacy). Boot it up so that the resize partition thing happens. Then shutdown and create another primary partition with ext4 (give about 5 GB to root, 3 GB to the new one) using gparted or whatever from another machine.

It should ask to create new user. use music,music

then boot and run raspi-config

* change hostname to eyesy 
* under interfacing disable serial console, enable port 
* enable ssh

install vim

    sudo apt-get update 
    sudo apt-get install vim 

change keyboard:

    sudo vim /etc/default/keyboard 

change gb to us, reboot

install git

    sudo apt-get install git

configure

    git config --global user.email "..."
    git config --global user.name "..."

# setup WM8731 audio driver with SPI control

check music_rootfs/audio-linux-6.6 for info
disable defaut card:
sudo echo "blacklist snd_soc_hdmi_codec" | sudo tee /etc/modprobe.d/blacklist-snd_soc_hdmi_codec.conf

also add this to blacklist file
install snd_soc_hdmi_codec /bin/true

or maybe disable the default card using raspi-config or somethingt 

# config.txt

comment:

    #dtparam=audio=on
    #hdmi_force_hotplug=1 <-- ??  might not be relavent for new EY

uncomment:

    disable_overscan=1
    dtparam=i2c_arm=on
    dtparam=spi=on
    
add these:

    dtoverlay=wm8731-spi
    dtoverlay=gpio-poweroff,gpiopin=12,active_low="y"
    dtoverlay=pi3-miniuart-bt
    dtoverlay=midi-uart0 
    dtoverlay=pi3-act-led,gpio=24,activelow=on
    gpu_mem=64

reboot

# install packages 
  
    sudo apt-get install zip jwm xinit x11-utils x11-xserver-utils lxterminal pcmanfm adwaita-icon-theme gtk-theme-switch conky libasound2-dev liblo-dev liblo-tools mpg123 dnsmasq hostapd puredata swig fbi

    sudo apt-get install gnome-themes-extra python3-pip python3-liblo python3-pygame python3-psutil

# config

    sudo systemctl disable hciuart.service
    sudo systemctl disable dnsmasq.service
    sudo systemctl disable hostapd.service
    (or leave these two running for internet during dev)
    sudo systemctl disable wpa_supplicant
    sudo systemctl disable dhcpcd

make stuff in /root readable 

    sudo chmod +xr /root
    
make sdcard and usb directories
    
    sudo mkdir /sdcard
    sudo chown music:music /sdcard
    
    sudo mkdir /usbdrive
    sudo chown music:music /usbdrive

allow sudo with no password

    sudo visudo

add this to end of file

    music ALL=(ALL) NOPASSWD: ALL
    
add this to /etc/fstab to mount the patches partition:

    /dev/mmcblk0p3 /sdcard  ext4 defaults,noatime 0 0
   
reboot and change owner

    sudo chown music:music /sdcard 
    
remove this if it got added along the way

    sudo rm -fr /sdcard/lost+found/
    
enable rt.  in /etc/security/limits.conf add to end:

    @music - rtprio 99
    @music - memlock unlimited
    @music - nice -10
    
fiddle with pcmanfm till it works. some of this stuff gets put in config file, but others get stored who knows where.  in preferences uncheck "Display simplified user interface.." in Layout.  uncheck everything under auto mount in Volume Management. change home to /sdcard under Advanced.  uncheck "Add deleted files to wastebasket" in General

in /etc/systemd/system.conf add:

    DefaultTimeoutStartSec=10s
    DefaultTimeoutStopSec=5s
    
boot stuff.  cmdline.txt should look like this:

    dwc_otg.lpm_enable=0 root=PARTUUID=9d5fbf22-02 console=tty3 rootfstype=ext4 elevator=deadline fsck.mode=skip rootwait noswap fastboot loglevel=0 logo.nologo vt.global_cursor_default=0

for faster booting

    sudo systemctl disable raspi-config.service
    sudo systemctl disable triggerhappy.service
    
remove plymouth all together (splash screen provided by EYESY_OS)

    sudo apt-get purge --remove plymouth
    
disable tty1

    sudo systemctl disable getty@tty1.service

then 

    sudo systemctl daemon-reload

reboot

# install other software

## openFrameworks

for compiling, set gpu memory small (64) in /boot/config.txt, then increase after done

    cd
    wget https://openframeworks.cc/versions/v0.11.2/of_v0.11.2_linuxarmv6l_release.tar.gz
    mkdir openFrameworks && sudo tar vxfz of_v0.11.2_linuxarmv6l_release.tar.gz -C openFrameworks --strip-components 1
    rm of_v0.11.2_linuxarmv6l_release.tar.gz 
    sudo chown -R music:music openFrameworks
    cd openFrameworks/scripts/linux/debian
    sudo ./install_dependencies.sh && sudo ./install_codecs.sh && sudo apt-get clean
    
force it to use legacy driver:
 
    cd
    vim openFrameworks/libs/openFrameworksCompiled/project/linuxarmv6l/config.linuxarmv6l.default.mk

comment out this line:   
    
    USE_PI_LEGACY = 0
    
compile

    cd && sudo make Release -C openFrameworks/libs/openFrameworksCompiled/project
    
## ofxLua

increase swap to 1024 megs for compiling (set CONF_SWAPSIZE=1024 in /etc/dphys-swapfile and reboot), then we'll set it to 0 and disable when finished. 

    cd
    sudo apt-get install luajit-5.1
    cd openFrameworks/addons/
    git clone git://github.com/danomatika/ofxLua.git
    cd ofxLua
    git submodule init
    git submodule update
    scripts/generate_bindings.sh

## eyesy-oflua

    cd
    cd openFrameworks/apps/myApps/
    git clone https://github.com/owenosborn/ofEYESY.git
    mv ofEYESY eyesy
    cd eyesy
    make

disable swap:

    sudo systemctl disable dphys-swapfile

also set CONF_SWAPSIZE=0 in /etc/dphys-swapfile for good measure

increase gpu memory to 256 in /boot/config.txt, reboot

## node js

    cd 
    curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
    sudo apt-get install -y nodejs
    
install modules (might move these to the EYESY_OS repo), for now install 
    
    cd ~/EYESY_OS/web/node
    npm install websocket
    npm install tail
    


## cherrypy 

try running it again if you get an error

    pip2 install cherrypy==11.0.0


# make it read only

clean up

    sudo apt-get autoremove --purge
    
add to /boot/cmdline.txt

    ro
 
remove fsck.repair=yes, add fsck.mode=skip

move /var/spool to /tmp
    
    rm -rf /var/spool
    ln -s /tmp /var/spool

in /etc/ssh/sshd_config

    UsePrivilegeSeparation no

in /usr/lib/tmpfiles.d/var.conf replace "spool 0755" with "spool 1777"

move dhcpd.resolv.conf to tmpfs
    
    sudo touch /tmp/dhcpcd.resolv.conf
    sudo rm /etc/resolv.conf
    sudo ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf
    
in /etc/fstab add "ro" to /boot and /, then add:

    tmpfs /var/log tmpfs nodev,nosuid 0 0
    tmpfs /var/tmp tmpfs nodev,nosuid 0 0
    tmpfs /tmp     tmpfs nodev,nosuid 0 0
    
stop time sync cause it is not working anyway.  (also causes issues with LINK when the clock changes abruptly). need solution

    timedatectl set-ntp false
    
reboot

# release

clean out home folder

remove wifi config (change to music, coolmusic)

set shift params to default

remove files in Grabs

remove .viminfo

remove git config

clear command history:

    cat /dev/null > ~/.bash_history && history -c && exit
    
    
shutdown and on another machine 

run zerofree on ext partitions

    sudo zerofree -v /dev/sda2
    sudo zerofree -v /dev/sda3

run fsck

    sudo fsck /dev/sda1
    sudo fsck /dev/sda2
    sudo fsck /dev/sda3

reboot and test

on another machine dd and zip it up

    sudo dd if=/dev/rdisk1 of=EYESY-v3.0.img bs=1m
    zip -db EYESY-v2.0.img.zip EYESY-v3.0.img

# experimental zone

maybe also use swapoff in cmdline.txt

install wiring pi 
    git clone https://github.com/WiringPi/WiringPi.git
    cd WiringPi
    ./build debian
    mv debian-template/wiringpi_3.10_armhf.deb .
    sudo apt install ./wiringpi_3.10_armhf.deb
    sudo apt install ./wiringpi-3.0-1.deb

use this for CPU 
    echo -n performance | sudo tee /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

install waitress for flask server

update wifi drivers:
sudo apt update
sudo apt install firmware-realtek

install nload

don't persist logs
add 
Storage=volatile
to
/etc/systemd/journald.conf
then remove old
sudo rm -rf /var/log/journal
restart

Don't log nmcli commands
Open the sudoers file for editing using visudo:

sudo visudo

Add a rule to disable logging for music user:

Defaults:music !syslog

systemctl disable NetworkManager-wait-online.service

actually don't install / use waitress server(?)

make sure vc4.tv_norm=NTSC is in cmdline.txt
