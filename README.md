# boot_usb_with_ventoy_debian_live_persistence

**Bootable usb key with Ventoy Debian with live persistence.**  
***version: 1.0.0 date: 2022-09-09 author: [bestia.dev](https://bestia.dev) repository: [Github](https://github.com/bestia-dev/boot_usb_with_ventoy_debian_live_persistence)***  

![status](https://img.shields.io/badge/obsolete-red) 
![status](https://img.shields.io/badge/archived-red) 
![status](https://img.shields.io/badge/tutorial-yellow) 

Hashtags: #rustlang #tutorial #buildtool #developmenttool  
My projects on Github are more like a tutorial than a finished product: [bestia-dev tutorials](https://github.com/bestia-dev/tutorials_rust_wasm).

## Motivation

A bootable usb is the most basic of tools to diagnose computer problems and much more.
Having many usb keys with different boots is not practical.  
[Ventoy](https://www.ventoy.net/en/index.html) is an open source tool to create bootable USB drive for ISO/WIM/IMG/VHD(x)/EFI files. With ventoy, you don't need to format the disk over and over, you just need to copy the ISO/WIM/IMG/VHD(x)/EFI files to the USB drive and boot them directly. On system start, Ventoy will ask you with a user-friendly menu what ISO you want to boot from. I could have DebianLive, Clonezilla, GParted and Sergei Strelec WinPE all on une single usb key.

## Prepare usb key with Ventoy

I will do everything in Debian bash and try to avoid GUI like the plague. That way we have very nice incantation we can just copy and paste.  
Download `ventoy-1.0.79-linux.tar.gz` from [github](https://github.com/ventoy/Ventoy/releases).  
I will unpack the `ventoy-1.0.79-linux.tar.gz` into `~/.local/ventoy-1.0.79`.

```bash
curl -s -L https://github.com/ventoy/Ventoy/releases/download/v1.0.79/ventoy-1.0.79-linux.tar.gz --output ~/Downloads/ventoy-1.0.79-linux.tar.gz
cd ~/Downloads
tar -xvf ventoy-1.0.79-linux.tar.gz -C ~/.local
cd ~/.local
ls -la
cd ventoy-1.0.79
ls -la
```

Next, insert your empty USB key. All the data on it will be deleted anyway!
Mine has 32 GB and I will reserve/preserve 8GB space at the bottom of the disk for the persistent disk of DebianLive.
Now find out what is the Filesystem name of your usb key.

```bash
$ lsblk -f

NAME        FSTYPE FSVER LABEL
sda
|-sda1      exfat  1.0   empty_usb
...

```

On my system my USB key is `/dev/sda` (without the last number). I will use it for the `Ventoy2Disk` bash command. Yours could be maybe `/dev/sdb` or `/dev/sdc`. Be careful to choose the right one. This will delete all the data on that disk! I will leave 8GB for the persistent storage of DebianLive.

```bash
$ cd ~/.local/ventoy-1.0.79
$ sudo ./Ventoy2Disk.sh -i -r 8192 /dev/sda
...
Wait for partitions ...
/dev/sda1 exist OK
/dev/sda2 exist OK
partition exist OK
...
Install Ventoy to /dev/sda successfully finished.

$ lsblk -f

NAME        FSTYPE FSVER LABEL
sda                                                                                      
|-sda1      exfat  1.0   Ventoy                                       
`-sda2      vfat   FAT16 VTOYEFI
```

Now I want to format the last partition of `/dev/sda` as `ext4` for Debian Linux and label it `persistence`. This volume name `persistence` is mandatory for Debian Live persistence! First I need the exact Start and End numbers of the Free partition.

```bash
$ sudo parted /dev/sda print free

Model:  USB  SanDisk 3.2Gen1 (scsi)
Disk /dev/sda: 30.8GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
        1024B   1049kB  1048kB           Free Space
 1      1049kB  22.1GB  22.1GB  primary               boot
 2      22.1GB  22.2GB  33.6MB  primary  fat16        esp
        22.2GB  30.8GB  8590MB           Free Space

$ sudo parted /dev/sda mkpart primary ext4 22.2GB 30.8GB
$ sudo mkfs.ext4 /dev/sda3
$ sudo e2label /dev/sda3 persistence

$ sudo parted /dev/sda print 
Model:  USB  SanDisk 3.2Gen1 (scsi)
Disk /dev/sda: 30.8GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags: 

Number  Start   End     Size    Type     File system  Flags
 1      1049kB  22.1GB  22.1GB  primary               boot
 2      22.1GB  22.2GB  33.6MB  primary  fat16        esp
 3      22.2GB  30.8GB  8590MB  primary  ext4

$ lsblk -f
NAME        FSTYPE FSVER LABEL
sda                                                                                      
|-sda1      exfat  1.0   Ventoy
|-sda2      vfat   FAT16 VTOYEFI
`-sda3      ext4   1.0   persistence
```

The usb key volumes and partitions are automagically mounted as `/media/username/name`, but I don't like that because the username changes for everybody. I will mount it with a fixed name.

```bash
sudo umount /media/usb-ventoy
sudo mkdir /media/usb-ventoy
# for fat we can use umask, because it does not have file permissions
sudo mount /dev/sda1 /media/usb-ventoy/ -o umask=000
cd /media/usb-ventoy
ls -la 

sudo umount /media/usb-persistence
sudo mkdir /media/usb-persistence
# on ext4 cannot use umask, 
sudo mount /dev/sda3 /media/usb-persistence/
# I must use chmod 777 to give all permission to everybody. It is just a USB key. For everybody.
sudo chmod 777 /media/usb-persistence
cd /media/usb-persistence
ls -la
```

In the root of the persistence volume we need a file `persistence.conf` that contains the settings for persistence. We need only one simple line that means "save all the modified files".  

```bash
cd /media/usb-persistence
nano persistence.conf
```

One line content:  

```conf
/ union
```

Save and exit.

## Clonezilla live

<https://clonezilla.org>  
Clonezilla is a partition and disk imaging/cloning program similar to True Image® or Norton Ghost®. It helps you to do system deployment, bare metal backup and recovery. Clonezilla live is suitable for single machine backup and restore.  
Download the stable ISO from <https://clonezilla.org/downloads.php>.  
From the options I choose amd64, iso and auto, then click download.  
Just move this iso file to the usb key. Done !

```bash
mv ~/Downloads/clonezilla-live-3.0.1-8-amd64.iso /media/usb-ventoy/
```

## GParted live

<https://gparted.org>  
GParted is a free partition editor for graphically managing your disk partitions.
With GParted you can resize, copy, and move partitions without data loss.  
Download it from <https://gparted.org/download.php>. I choose the `gparted-live-1.4.0-5-amd64.iso`. Then move it to the Ventoy usb key. Done!  

```bash
mv ~/Downloads/gparted-live-1.4.0-5-amd64.iso /media/usb-ventoy/
```

## Sergei Strelec WinPE

<https://www.majorgeeks.com/files/details/sergei_strelecs_winpe.html>  
Sergei Strelec's WinPE creates a bootable DVD or thumb drive for PC maintenance, including partitioning, backup and restoring, diagnostics, data recovery, and more.  
Password: strelec
There are a handful of great WinPE builds out there, and this is one of them. The software list is exhausting and would require scrolling numerous pages if we were to list them all. That said, to give you an idea:  
Backups include Acronis, Nortons Ghost, Disk2vhd, Macrium, and more. Drive utilities include MiniTool, Macrorit, Defraggler,
Auslogics Disk Defrag, Killdisk, and more. Diagnostics include AIDA64, Burnin Test, HWiNFO, OCCT, CPU-Z, and more. Data recovery includes EASEUS, R-Studio, Active File Recovery, and more. DOS programs include MemTest86+, MemTest86, Ghost, BootIt Bare Metal, Gold Memory, and more.  
See what I mean? And this is only a partial list and does not even begin to cover the other categories, including drivers, antivirus, Windows installation, networking, and native mode.  
Because it was built using WinPE10 and WinPE8, many of these utilities might work on older operating systems, including Windows 7 and Vista, but that's at your own risk.  
Regardless of what your current computer problems are, Sergei Strelec's WinPE has you covered.

Download the 3.8 GB rar file from <https://www.majorgeeks.com/files/details/sergei_strelecs_winpe.html>. Why oh why rar? Whose idea was that? And even with a password? We must now install the strange rar for linux.

```bash
cd ~/Downloads
curl https://www.rarlab.com/rar/rarlinux-x64-612.tar.gz --output rarlinux-x64-612.tar.gz

tar -zxvf rarlinux-x64-612.tar.gz
cd rar
sudo cp -v rar unrar /usr/local/bin/

cd ~/Downloads
unrar WinPE10_8_Sergei_Strelec_x86_x64_2022.01.03_English.rar
# the rar password is: strelec
# we need only the iso file. Don't extract the rest.

mv ~/Downloads/WinPE10_8_Sergei_Strelec_x86_x64_2022.01.03_English.iso /media/usb-ventoy/
```

The iso is now moved to the usb-key. Done!

## Debian Live with non-free

I choose the non-free image because of the wifi driver on my notebook.
DebianLive could be used from bootable USB key with persistent storage. That is nice !  
The same usb key can also be used to install Debian on the computer. Very nice!  
I downloaded the 3.1GB iso file from  

```bash
cd ~/Downloads
curl -SfL https://cdimage.debian.org/cdimage/unofficial/non-free/cd-including-firmware/current-live/amd64/iso-hybrid/debian-live-11.5.0-amd64-xfce+nonfree.iso --output debian-live-11.5.0-amd64-xfce+nonfree.iso
```

If the download does not work, probably there is a newer version of Debian and you have to correct the file name.  

## Debian Live persistance

This image does not have the boot parameters persistence. We need to search and replace some words inside the iso file and create a new one. Interesting!  

```bash
LANG=C sed 's/splash quiet/persistence /;s/quiet splash/persistence /' \
 <./debian-live-11.5.0-amd64-xfce+nonfree.iso \
 >./debian-live-11.5.0-amd64-xfce+nonfree-persist.iso 
```

Now we have 2 iso images. One has the persistence parameter and the other has not. I will move them both to the usb key.  

```bash
mv ~/Downloads/debian-live-11.5.0-amd64-xfce+nonfree-persist.iso /media/usb-ventoy/
mv ~/Downloads/debian-live-11.5.0-amd64-xfce+nonfree.iso /media/usb-ventoy/
```

Debian recognizes a volume partition on the usb key to be its persistence storage if:

- the volume label is `persistence`
- it contains the file `persistence.conf` on the root with correct content

We already done all of that.  

## Other utilities

The usb key has still a lot of free space. I copied some utility programs I often use for Linux and Windows. The free space is not filled with many files like with a classic bootable usb. There are just a few isolated iso files. Nice !
My choice are: Total Commander, IrfanView, LibreOffice, ...

## Boot

Let's try it.  
Shutdown the computer. Insert the usb key.  
Turn on the computer. On my Lenovo I have to immediately  press F2 to get into the UEFI or Bios setup. There I choose "Usb boot enabled" and put usb boot to be first in line to boot the computer. Save and Exit. Turn on the computer again.
Now Ventoy presents a menu to boot from these options:  

- Clonezilla
- Debian live persistence
- GParted
- WinPE

I choose the DebianLive with persistence.
The user "user" password is "live".

## Debian Live Terminal

The main tool for Debian is the Terminal. It opens the Debian bash.
In the terminal use Shift-Ctrl-C and Shift-Ctrl-V for copying. This is a little annoying, but ctrl-c and ctrl-v are already taken.
Disable the "unsafe paste warning" in the menu "Edit - Preferences - Show unsafe paste dialog" of the terminal window GUI.

First I need to change to Slovenian keyboard.

```bash
sudo apt-get install debconf
sudo dpkg-reconfigure keyboard-configuration
```

It opens a Text UI to choose the keyboard options.
Generic 105 keys, Other-Slovenian with guillemets, ...
To apply the new setting:  

```bash
sudo udevadm trigger --subsystem-match=input --action=change
```

It will persist these changes because of persistence.  

## Open-source and free as a beer

My open-source projects are free as a beer (MIT license).  
I just love programming.  
But I need also to drink. If you find my projects and tutorials helpful, please buy me a beer by donating to my [PayPal](https://paypal.me/LucianoBestia).  
You know the price of a beer in your local bar ;-)  
So I can drink a free beer for your health :-)  
[Na zdravje!](https://translate.google.com/?hl=en&sl=sl&tl=en&text=Na%20zdravje&op=translate) [Alla salute!](https://dictionary.cambridge.org/dictionary/italian-english/alla-salute) [Prost!](https://dictionary.cambridge.org/dictionary/german-english/prost) [Nazdravlje!](https://matadornetwork.com/nights/how-to-say-cheers-in-50-languages/) 🍻

[//bestia.dev](https://bestia.dev)  
[//github.com/bestia-dev](https://github.com/bestia-dev)  
[//bestiadev.substack.com](https://bestiadev.substack.com)  
[//youtube.com/@bestia-dev-tutorials](https://youtube.com/@bestia-dev-tutorials)  
