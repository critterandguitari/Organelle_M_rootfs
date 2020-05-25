# General 

start with raspbian lite, boot it up so that the resize partition thing happens (tried to disable this but ran into trouble). then shutdown and create another primary partition with ext4 (give about 5 GB to root, 3 GB to the new one) using gparted or whatever from another machine.

then boot and run raspi-config

* change pw to music
* change hostname 
* disable wait for network on boot
* change wifi location to US
* change timezone
* enter wifi ssid 
* under interfacing disable serial console, enable port 
* enable ssh

install vim

    apt-get update 
    apt-get install vim 

change keyboard:

    vim /etc/default/keyboard 

change gb to us, reboot

change pi username to music. first make root password

    sudo passwd root

use 'music'.  then logout and login as root (have to disable autologin first).  change name

    usermod -l music pi

change home dir

    usermod -m -d /home/music music

change group name

    groupmod --new-name music pi
    
reboot

install git

    apt-get install git

configure

    git config --global user.email "..."
    git config --global user.name "..."
    
pull down this repo

    git clone https://github.com/owenosborn/Organelle_M_rootfs.git
    cd Organelle_M_rootfs

# setup WM8731 audio driver with SPI control

update kernel
 
    rpi-update 
    reboot 

fix audio driver for spi (replace 4.19.42 with whatever kernel version running)
 
    cd ~/Organelle_M_rootfs/audio
    sudo ./fixit.sh /lib/modules/4.19.42-v7+/kernel/sound/soc/bcm/snd-soc-audioinjector-pi-soundcard.ko

compile new dt overlay for spi

    cp audioinjector-wm8731-audio-spi-overlay.dts /boot
    sudo dtc -@ -I dts -O dtb -o /boot/overlays/wm8731-spi.dtbo /boot/audioinjector-wm8731-audio-spi-overlay.dts

# config.txt

comment:

    #dtparam=audio=on

uncomment:

    disable_overscan=1
    hdmi_force_hotplug=1
    dtparam=i2c_arm=on
    dtparam=spi=on
    
add these:

    dtoverlay=wm8731-spi
    dtoverlay=gpio-poweroff,gpiopin=12,active_low="y"
    dtoverlay=pi3-miniuart-bt
    dtoverlay=midi-uart0 
    dtoverlay=pi3-act-led,gpio=24,activelow=on
    gpu_mem=64

# install packages 
    
    apt-get install zip jwm xinit x11-utils x11-xserver-utils lxterminal pcmanfm adwaita-icon-theme gnome-themes-standard gtk-theme-switch conky libasound2-dev liblo-dev liblo-tools python-pip mpg123 dnsmasq hostapd puredata wiringpi
    
python stuff 

    pip2 install Cython
    pip2 install pyliblo

cherrypy, try running it again if you get an error

    pip2 install jaraco.functools==2.0
    pip2 install cherrypy==11.0.0
    
node js

    cd 
    curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
    sudo apt-get install -y nodejs
    
install modules (might move these to the EYESY_OS repo), for now install 
    
    cd ~/EYESY_OS/web/node
    npm install websocket
    npm install tail
    
## openFrameworks

for compiling, set gpu memory small (64), then increase after done

    wget https://openframeworks.cc/versions/v0.11.0/of_v0.11.0_linuxarmv6l_release.tar.gz
    mkdir openFrameworks && sudo tar vxfz of_v0.11.0_linuxarmv6l_release.tar.gz -C openFrameworks --strip-components 1
    rm of_v0.11.0_linuxarmv6l_release.tar.gz 
    chown -R music:music openFrameworks
    cd openFrameworks/scripts/linux/debian
    sudo ./install_dependencies.sh && sudo ./install_codecs.sh && sudo apt-get clean
    
fix thing for headless operation:
 
    vim openframeworks/libs/openFrameworksCompiled/project/linuxarmv6l/config.linuxarmv6l.default.mk

comment out this line:   
    
    USE_PI_LEGACY = 0
    
compile 

    cd && sudo make Release -C openFrameworks/libs/openFrameworksCompiled/project
    
add flag to project config.make

    PROJECT_LDFLAGS += -latomic 
    
## ofxLua

    sudo apt-get install luajit-5.1
    cd openframeworks/addons/
    git clone git://github.com/danomatika/ofxLua.git
    cd ofxLua
    git submodule init
    git submodule update
    
try it 

    cd luaExample
    cp ../../../examples/graphics/graphicsExample/Makefile ./
    cp ../../../examples/graphics/graphicsExample/config.make ./
    make
    
then increase gpu memory to 256 

# config

    systemctl disable hciuart.service
    systemctl disable dnsmasq.service
    systemctl disable hostapd.service

make stuff in /root readable 

    chmod +xr /root
    
make sdcard and usb directories
    
    mkdir /sdcard
    chown music:music /sdcard
    
    mkdir /usbdrive
    chown music:music /usbdrive

allow sudo with no password

    sudo visudo

add this to end of file

    music ALL=(ALL) NOPASSWD: ALL
    
add this to /etc/fstab to mount the patches partition:

    /dev/mmcblk0p3 /sdcard  ext4 defaults,noatime 0 0
   
reboot and change owner

    chown music:music /sdcard 
    
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

then 
    
    systemctl daemon-reload

# make it read only

clean up

    apt-get autoremove --purge
    
add to /boot/cmdline.txt

    fastboot noswap ro
 
remove fsck.repair=yes, add fsck.mode=skip

move /var/spool to /tmp
    
    rm -rf /var/spool
    ln -s /tmp /var/spool

in /etc/ssh/sshd_config

    UsePrivilegeSeparation no

in /usr/lib/tmpfiles.d/var.conf replace "spool 0755" with "spool 1777"

move dhcpd.resolv.conf to tmpfs
    
    touch /tmp/dhcpcd.resolv.conf
    rm /etc/resolv.conf
    ln -s /tmp/dhcpcd.resolv.conf /etc/resolv.conf
    
in /etc/fstab add "ro" to /boot and /, then add:

    tmpfs /var/log tmpfs nodev,nosuid 0 0
    tmpfs /var/tmp tmpfs nodev,nosuid 0 0
    tmpfs /tmp     tmpfs nodev,nosuid 0 0
    
stop time sync cause it is not working anyway.  (also causes issues with LINK when the clock changes abruptly). need solution

    timedatectl set-ntp false
    
reboot

# release

    pull latest Organelle_OS and deploy
    remove .viminfo
	remove git config
    clear command history
    run fsck
    reboot and test

