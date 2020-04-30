# Lenovo Thinkpad T430 Laptop Ubuntu 20.04 Desktop Install

Some notes gathered from various sources on the internet gathered into one place to help anyone wanting to install Ubuntu onto a Thinkpad T430 laptop. The Ubuntu installation will work just fine without any of the tweaks below as the Thinkpad hardware is well supported out of the box, but the changes below help it run even better and might iron out one or two niggles.

Background on the hardware of the [T430 is available on the ThinkWiki page](http://www.thinkwiki.org/wiki/Category:T430). 

Note the machine used for this install had the standard intel integrated graphics.

## Initial OS Install

Backup anything important on the laptop before starting. Check the backup works.

You can use legacy BIOS installation, but as this laptop supports UEFI might as well use it. The easiest way to ensure the install uses the UEFI installation method is to download the Ubuntu 20.04 iso image and create a live USB Ubuntu boot drive from Windows using [Rufus](https://rufus.ie/). When configuring Rufus, choose the UEFI option to ensure the install uses UEFI also, select the Ubuntu ISO downloaded and create a live USB.

This installation uses UEFI with secure boot, so disable the legacy BIOS boot option in BIOS options of the T430 and enable secure boot.

Boot from USB drive, changing boot order if necessary in BIOS. Choose installation options e.g.  encrypted LVM volume and 3rd party extras. As secure boot has been enabled, the installer asks for another password (this is required on reboot to register those extra drivers using MOK and just needs something simple like monkey123 because it’s not saved, it’s just used to prove you are still at the keyboard - this is not the LVM password). Follow the installation procedure noting the warnings especially if you have any files or operating systems you want to retain.

Install updates

    sudo apt-get update
    sudo apt-get upgrade

## Install Applications

Install a few essential packages to help the Thinkpad hardware work better with Ubuntu. As an alternative to the command line, Synaptic provides a GUI to do this:

    sudo apt-get install synaptic

### Install Applications e.g. using Synaptic Package Manager

* libdvd-pkg (codec for DVD Movies, run sudo dpkg-reconfigure libdvd-pkg to build)
* lame (MP3 codec)
* ubuntu-restricted-extras
* nautilus-admin (Makes it easier to edit configuration files as administrator)
* TLP laptop power saving tools (check using sudo tlp-stat -s)
* acpi-call-dkms Thinkpad battery (no need to set thresholds as I work normally on battery and AC, but check using sudo tlp-stat -b)
* acpitool

## Hardware & System Configuration

### Wifi - Losing Internet Connectivity Randomly

There can be problems, as n mode wifi with the intel 6205 doesn’t work reliably with all routers. Add one of the lines below (they are in order of least impact on speed i.e. try the top one first) to the end of the file /etc/modprobe.d/iwlwifi.conf .  These deconflict the Bluetooth and enables a consistent wifi connection:

* options iwlwifi 11n_disable=1
* options iwlwifi 11n_disable=1 swcrypto=1 bt_coex_active=0 power_save=0
* options iwlwifi bt_coex_active=0 11n_disable=8

### Bluetooth - Copy Firmware File

As described here [https://github.com/winterheart/broadcom-bt-firmware](https://github.com/winterheart/broadcom-bt-firmware) with a default install there is an error in the bluetooth initialisation which says something like 

“bluetooth hci0: firmware: failed to load brcm/BCM20702A1-0a5c-21e6.hcd“

To resolve this, copy the file BCM20702A1-0a5c-21e6.hcd from the above git repository to  /lib/firmware/brcm/ and reboot. Check there were no errors using dmesg | grep -i blue

### Ubuntu Live Patching - Enable (only for LTS versions)

This is enabled in update settings, log-in with an Ubuntu-One account to turn on.

### ThinkVantage Button - Use it to Launch a Terminal Window

Go to: Settings/Devices/Keyboard/Custom Shortcuts \
Click Add (+) . . . and name the button 'Launch Terminal.' In the 'command' box type: gnome-terminal \
then push the ThinkVantage button. This should successfully map the button to launch a terminal window.

### SSD - Set Swappiness

Edit /etc/sysctl.conf \
Search for vm.swappiness and change its value as desired. If vm.swappiness does not exist, add it to the end of the file:

    vm.swappiness=10

### Fan - **Set up the Thinkpad Fan to Operate Correctly**

This is needed because without this fan maxes out at 4500 RPM, which can lead to throttling the processor on high load. Described at [http://thinkwiki.de/Thinkfan](http://thinkwiki.de/Thinkfan)

Install packages

    sudo apt-get install thinkfan lm-sensors

After installing packages start:

    sudo sensors-detect

Choose the default answers (just hit Y or enter) should be okay

Then look for thermal sensors and note the output - These are included as the hwmon lines in the config file at next step. The location and number of these sensors varies a lot depending upon laptop type and linux distribution:

    find /sys/devices -type f -name "temp*_input"

Now open and edit /etc/thinkfan.conf as root and edit to read (careful not to add any blank lines or spaces at the end of the last line. Also note that hwmon3 may initially show as hwmon4 or something else, it also seems to change after reboot so initial file locations should be checked if it fails to load successfully). Note - adjust these values to suit particular workloads, processors and preferences:

    tp_fan /proc/acpi/ibm/fan

    hwmon /sys/devices/platform/coretemp.0/hwmon/hwmon3/temp3_input
    hwmon /sys/devices/platform/coretemp.0/hwmon/hwmon3/temp1_input
    hwmon /sys/devices/platform/coretemp.0/hwmon/hwmon3/temp2_input
    hwmon /sys/devices/virtual/thermal/thermal_zone0/hwmon1/temp1_input
    
    (0, 0, 42) 
    (1, 40, 47)
    (2, 45, 52)
    (3, 50, 57)
    (4, 55, 62)
    (5, 60, 70)
    (6, 68, 79)
    (7, 77, 87)
    (127, 85, 32767)

The values in brackets are for the fan levels with the minimal and maximal temperatures before changing up/down the fan level. eg.: (0, 0, 55) means: fan level: 0 between 0-42 C. Fan turns on (at fan speed level 1) above 42C turns off below 40C and steps to fan level 2 above 47C. (because of level 1 config (1, 40, 47).  Note that level 126 is full-speed and 127 is disengaged (max RPM about 5500). Values for levels must overlap otherwise thinkfan reverts to auto. Level 7 is about 4500 RPM. Setting this Disengaged level keeps the temperature about 80 degrees at maximum load rather than getting too high (87 degrees is listed as “high” and 105 as “critical”) which will cause the laptop to shut down.

Enable service: via a kernel module. Issue command as root:

    sudo modprobe thinkpad_acpi fan_control=1

To make the module load at boot (issue commands as root from terminal (all one line):

    sudo echo "options thinkpad_acpi fan_control=1" | sudo tee /etc/modprobe.d/thinkpad_acpi.conf

Restart Laptop

Now start the thinkfan service from command line (also as root)

    sudo systemctl start thinkfan.service

if there is no error, check service status (may need to reboot):

    sudo systemctl status thinkfan.service

Enable the service to start at boot time:

    sudo systemctl enable thinkfan.service

Need to edit /etc/default/thinkfan so the line beginning "START" reads START=yes

check if the service is really working by checking (from terminal)

    cat /proc/acpi/ibm/fan and sensors

example output is:

    status:        enabled
    speed:         1967
    Level:         1
    commands:      level &lt;level> (&lt;level> is 0-7, auto, disengaged, full-speed)
    commands:      enable, disable
    commands:      watchdog &lt;timeout> (&lt;timeout> is 0 (off), 1-120 (seconds))

Note that the level is shown as a number and not auto. (if it is auto, then thinkpad_acpi kernel module isn't loaded or there is a configuration file error)

### Secure Boot - Enable and register Machine Owner Keys (MOK)

Most of this happens automatically on installation, but if the installation was carried out with secure boot turned off then this needs to be followed to register additional keys for 3rd party drivers. Ubuntu provides a helper tool that simplifies the configuration process:

Create a key (This will check to see if a key already exists and reuse it if possible.)

    sudo update-secureboot-policy --new-key 

Enroll the key into the MOK database (If the command indicates that secure boot is not enabled, you will need to enable it first. If the command completes with no message, the key has already been imported)

    sudo update-secureboot-policy --enroll-key 

If the command indicates that no DKMS modules are installed, you will need to run this command instead 

    sudo mokutil --import /var/lib/shim-signed/mok/MOK.der

Enter a temporary password at the input password: prompt, and repeat it when asked. (This password will only be used once at the next reboot to ensure you are physically present. It is okay to use something simple like password or monkey123)

Reboot the computer. A blue screen titled "Perform MOK management" will start instead of Linux.

(If you do not respond within 10 seconds the computer will continue the boot process and load Linux. If this happens you can repeat). Select the "Enroll MOK" option, and then "Continue", and "Yes".  When asked for a password, type in the temporary password entered above.

Select "Reboot"

Check secure boot status using

    mokutil --sb-state
