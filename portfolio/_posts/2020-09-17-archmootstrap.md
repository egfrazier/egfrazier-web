---
author: egfraz
title: Shell Script for Bootstrapping Arch Linux
---
About a year ago I switched from Ubuntu to Arch Linux. After repeating the install process multiple times, I created a shell script to automate most of the steps in the [Arch Installation Guide](http://wiki.archlinux.org/index.php/installation_guide). This script was created on a laptop with a single drive with not existing operating before install.

A couple things to note:
- As with any operating system installation, extreme care should be taken to ensure you are always referencing the correct drive. This script does not do any checking for existing partitions or data and will rewrite anything on the drive.
- This script uses sfdisk to programatically create the partitions based information in a separate file in the same directory as the scrip. For more information on sfdisk, go here.
- A WiFi connection is assumed and will be setup by the script using IWD and credentials you set in the variable section.

Here are the script contents. It is also available at [http://egfrazier.com/arch-bootstrap.sh](http://egfrazier.com/arch-bootstrap.sh).

```
    #!/bin/bash

    # Command line args
    # =================

    # Variables
    IFACE_NAME=$1
    WIFI_DEV_IDX=""
    WIFI_NET_NAME=""
    WIFI_NET_PASSWORD=""
    SYS_DEV_NAME=''
    SYS_PART_PREFIX=''
    PARTITION_COUNT=0
    BOOT_PART_SIZE=0
    ROOT_PART_SIZE=0
    HOME_PART_SIZE=0
    HOSTNAME=''
    PART_SCHEME_FILE=''
    ROOT_PASSWD=''
    LOCALE_REGION=''
    LOCALE_ZONE=''

    PROMPT="archinit> "

    function print {
	echo $PROMPT $1
    }

    # ===============================
    # SECTION I: WORKING FROM ARCHISO
    # ===============================

    # Verify boot mode
    # ================
    print "Verifying UEFI boot mode"
    ls /sys/firmware/efi/efivars &> /dev/null

    if [ $? -eq 0 ]; then
	print "UEFI boot mode verified"
    else
	print "/sys/firmware/efi/efivars not found"
	print "This system has not been booted in UEFI mode."
	print "Please refer to Arch Linux Installation Guide"
	print "on how to proceed in BIOS boot mode. Aborting..."
	exit
    fi

    # Connect to the Internet
    # =======================

    # Check that the wireless interface exists
    print "Verifying that $IFACE_NAME interface exists..."
    ls /sys/class/net/$IFACE_NAME &> /dev/null
    if [ $? -eq 0 ]; then
	print "$IFACE_NAME interface exists"
    else
	print "$IFACE_NAME interface does not exists"
	print "Please review your network interface configuratin. Aborting..."
	exit
    fi

    # Ensure rfkill is not blocking wireless
    rfkill unblock $WIFI_DEV_IDX

    # Start IWD
    systemctl start iwd

    # Start network connection using IWD
    iwctl station $IFACE_NAME scan
    iwctl station $IFACE_NAME get-networks
    iwctl station $IFACE_NAME connect $WIFI_NET_NAME --passphrase $WIFI_NET_PASSWORD

    # Check that the interface is up
    IFACE_STATE=$(cat /sys/class/net/$IFACE_NAME/operstate)
    IFACE_CARRIER=$(cat /sys/class/net/$IFACE_NAME/carrier)

    if [ $IFACE_STATE == "up" ]; then
	print "$IFACE_NAME state is UP"
    else
	print "$IFACE_NAME state is DOWN. Aborting..."
	exit
    fi

    if [ $IFACE_CARRIER -eq 1 ]; then
	print "$IFACE_NAME carrier is UP"
    else
	print "$IFACE_NAME carrier is DOWN. Aborting..."
	exit
    fi

    WIFI_CONNECT_STATUS=$(iwctl station $IFACE_NAME show | grep 'connected' | awk '{print $2}')

    if [ $WIFI_CONNECT_STATUS == 'connected' ]; then
	print "WiFi connection is UP"
    else
	print "WiFi connection is DOWN. Aborting..."
	exit
    fi

    # Ping test
    ping -c 1 google.com
    if [ $? -eq 0 ]; then
	print "Ping test successful"
    else
	print "Ping test unsuccessful. Aborting..."
	exit
    fi

    # Update system clock
    timedatectl set-ntp true

    # Partition the disks

    # Back up existing partition scheme
    sfdisk -d /dev/$SYS_DEV_NAME > $SYS_DEV_NAME_part_scheme_preinstall_$(date +'%Y%m%d-%H%M').dump

    # Create partition scheme from file
    sfdisk -W always -w always /dev/$SYS_DEV_NAME < $PART_SCHEME_FILE

    # Format and check the partitions
    mkfs.fat -F32 /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"1
    fsck /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"1
    mkfs.ext4 /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"2
    fsck /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"2
    mkfs.ext4 /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"3
    fsck /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"3

    # Mount the partitions
    mount /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"2 /mnt
    mkdir -p /mnt/boot
    mkdir -p /mnt/home
    mount /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"1 /mnt/boot
    mount /dev/"$SYS_DEV_NAME$SYS_PART_PREFIX"3 /mnt/home

    # Intall essential packages
    pacstrap /mnt base linux linux-firmware efibootmgr iwd git dhcpcd base-devel

    # Genrate fstab
    genfstab -U /mnt >> /mnt/etc/fstab

    # ===============================
    # SECTION II: WORKING FROM CHROOT
    # ===============================

    # Chroot into the new Arch install
    # and perform final configurations
    cat <<EOF | arch-chroot /mnt
    ln -sf /usr/share/zoneinfo/$LOCALE_REGION/$LOCALE_ZONE /etc/localtime
    hwclock --systohc
    locale-gen
    sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
    echo 'LANG=en_US.UTF-8' >> /etc/locale.conf
    echo $HOSTNAME > /etc/hostname 
    echo "127.0.1.1        $HOSTNAME.localdomain $HOSTNAME" > /etc/hosts
    echo root:$ROOT_PASSWD | chpasswd
    pacman --noconfirm -S grub
    grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub
    grub-mkconfig -o /boot/grub/grub.cfg
    EOF
```                          
