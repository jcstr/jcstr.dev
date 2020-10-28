---
title: "Así instalo Arch Linux"
date: 2020-10-27T21:31:37-06:00
draft: true
---

Te doy la bienvenida a mi primer post! el cual dedicare a la comunidad mexicana de [Arch Linux](https://t.me/archlinuxmx).

Uno de los principales obstáculos que se encuentran en algunas comunidades de Software Libre y de Código Abierto, particularmente en México, es el no tener el dominio del idioma inglés, pero no hay ningún problema, para ello decidí escribir esta guía al español para poder ayudar a quienes tienen el interés de instalar **Arch Linux** y así poder usar el mismo día.

## ¿Para quién está dirigida está guía?
* Para aquellos que ~~son expertos~~ tienen la actitud de hacer las cosas por sí mismos.
* Está guía contiene todos los pasos enfocados para una instalación directa en la máquina.

## ¿Qué obtendré al finalizar esta guía?
* Una instalación de Arch Linux _cifrada_ bajo el esquema [LVM-LUKS](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Encrypting_an_entire_system_(Espa%C3%B1ol)#LVM_sobre_LUKS) (Osea, un sistema Linux con el disco cifrado {... para más seguridad :lock:}).

:zap:**Consejo**: Antes de instalar Arch Linux en tu máquina recomiendo que aprendas a familiarizarte con Arch desde una máquina virtual, ya sea con [Docker](https://picodotdev.github.io/blog-bitix/2014/11/como-instalar-y-guia-de-inicio-basica-de-docker/) o [Vagrant](https://fortinux.gitbooks.io/humble_tips/content/capitulo_1_usando_aplicaciones_en_linux/tutorial_instalar_vagrant_para_usar_ambientes_virtuales_en_gnulinux.html).


## Preliminares :checkered_flag:
Antes que nada, para poder instalar Arch Linux, necesitamos iniciar la imagen `.iso` desde una USB en la máquina que se desea instalar y para efectos prácticos, yo asumo que ya se tiene lista la USB cargada con la imagen de Arch Linux, además de que hay diferentes maneras de poder cargar la imagen de Arch en la USB yo sugiero [leer esta página de la wiki](https://wiki.archlinux.org/index.php/USB_flash_installation_medium_(Espa%C3%B1ol)) para conocer más respecto a este tema. 

## Configurando el teclado del instalador :cd:
Una vez que el instalador inició es momento de configurar el teclado acorde al idioma/región que tenemos en nuestro teclado, para ello yo asumo la configuración para `la-latin` (Español Latinoamericano):

```
# loadkeys la-latin1
```

Recomiendo [leer esta página de la wiki]() para ahondar más en la configuración del teclado.

## Conectandose a Internet :earth_americas:
Para conectarnos a Inernet desde el instalador  , necesitamos ejectuar una serie de comandos con `iwctl`:

Con este comando conocemos los dispositivos Wi-Fi que podemos usar para conectarnos:
```
# iwctl device list
```
Ahora, procedemos a buscar redes con el dispositivo WiFi especificado:
```
# iwctl station DEVICE scan
```
Listamos todas las redes disponibles:
```
# iwctl station DEVICE get-networks
```
Finalmente para conectarnos a una red en específico ejecutamos:
```
# iwctl --passphrase=PASSPHRASE station DEVICE connect SSID
```
Para verificar que nos hemos conectado efectivamente hacemos ping:
```
# ping archlinux.org
```
Listo! ya podemos continuar con la guía de instalación para poder instalar todo lo necesario.

## Verificar el modo de arranque (boot)
Para este caso en particular yo asumo el modo [UEFI](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface_(Espa%C3%B1ol)) y para verificar que la máquina tiene dicho modo, con este comando verifico la existencia de todas las variables que integran el firmware:

```
# ls /sys/firmware/efi/efivars
```
:zap:**Consejo**: Si la salida del comando no muestra nada, entonces tu máquina no tiene el modo UEFI, por lo que sugiero [leer esta página de la wiki](https://wiki.archlinux.org/index.php/Arch_boot_process_(Espa%C3%B1ol)).


## Verificando las propiedades del disco 
Para conocer el disco y sus respectivas particiones ejecutamos:
```
# fdisk -l
```

Una vez que tenemos el conocimiento sobre nuestro disco, procedemos a darle formato a las particiones necesarias.

## Preparando la particion de arranque (boot)
Para darle formato a la partición de arranque utilizaremos `gdisk`:

```
# gdisk /dev/sda
o
y
n
enter
+512MIB
ef00

n
enter
8e00
w
y
```

## Dandole formato a la partición boot
Para este caso en particular es necesario darle formato a la partición boot con el sistema de archivos FAT32 el cual es un requisito para poder instalar systemd-boot:
```
# mkfs.fat -F32 /dev/sda1
```
Y con esto ya tenemos lista la partición de arranque, ahora, procedemos a preparar las demás particiones y configurar el esquema LVM-LUKS.

## Cifrando Arch en nuestra máquina :lock:

## [LVM-LUKS](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)

## Encryption configuration ([cryptsetup](https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Cryptsetup_usage))
```
# cryptsetup luksFormat /dev/sda2
```

## Open the encrypted partition and create group and volumes (root, hoome, swap)
### Unlock the partition
```
# cryptsetup open --type luks /dev/sda2 lvm 
```
### Create respective groups
```
# pvcreate /dev/mapper/lvm
```
```
# vgcreate volume /dev/mapper/lvm
```
```
# lvcreate -L2G volume -n swap
```
```
# lvcreate -L50G volume -n root
```
```
# lvcreate -l 100%FREE volume -n home
```

## Formating partitions
### root
```
# mkfs.ext4 /dev/mapper/volume-root
```
### home
```
# mkfs.ext4 /dev/mapper/volume-home
```
### swap
```
# mkswap /dev/mapper/volume-swap 
```

### Create boot and home directories under /mnt
```
# mkdir /mnt/home
# mkdir /mnt/boot
```
### Mount partitions
```
# mount /dev/mapper/volume-root /mnt
# mount /dev/sda1 /mnt/boot
# mount /dev/mapper/volume-home /mnt/home
```

### Activate swap
```
# swapon /dev/mapper/volume-swap
```

---

## [Core](https://wiki.archlinux.org/index.php/Installation_guide#Install_essential_packages) installation
```
# pacstrap /mnt base base-devel linux linux-firmware vim networkmanager mkinitcpio lvm2 cryptsetup
```

## Generate [fstab](https://wiki.archlinux.org/index.php/Fstab)
```
# genfstab -U /mnt >> /mnt/etc/fstab
```

## Change root into the new system
```
# arch-chroot /mnt
```

## Local Time settings
```
# ln -s /usr/share/zoneinfo/America/Mexico_city /etc/localtime
# hwclock --systohc --utc
```

## Localization
```
# vim /etc/locale.gen

es_MX.UTF-8 UTF-8
```
### Generate Locale
```
# locale-gen 
```
### Save to config file
```
# locale > /etc/locale.conf
```
## Network configuration

### Hostname
```
# vim /etc/hostname
```
### Add entries to hosts
```
# vim /etc/hosts
127.0.0.1	  localhost
::1		      localhost
127.0.1.1	  myhostname.localdomain	  myhostname
```

### _Read carefuly_ the HOOKS named and its order, specially for this encrypted system
```
# vim /etc/mkinitcpio.conf 

HOOKS=(base udev autodetect modconf block keyboard encrypt lvm2 filesystems fsck)
```
### Recreate [initramfs](https://wiki.archlinux.org/index.php/Installation_guide#Initramfs) 
```
# mkinitcpio -P
```

## Establish root password
```
# passwd 
```

## Install bootloader
```
bootctl --path=/boot install
```

## Configure boot loaders
```
# vim /boot/loader/loader.conf

default arch 
timeout 3 
editor 0
```
```
# vim /boot/loader/entries/arch.conf

title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=86a872ee-b133-4e13-8283-d99024361d79:volume root=/dev/mapper/volume-root quiet rw
```
### Obtain the cryptdevice UUID under vim with:
``` 
:read ! blkid /dev/sda2 
``` 

## Logout from arch-chroot, umount partitions and reboot
`Ctrl+D` 

```
# umount -R /mnt 
# reboot
```
---
## Personalización  
Ahora procedemos a personalizar nuestra instalación de Arch.

Primero iniciamos sesión como root y agregamos nuestra cuenta de usario: 
```
# useradd -m -G $tu_nombre_de_usuario 
# passwd $tu_nombre_de_usuario
```

Ahora, instalamos sudo:
```
# sudo pacman -S sudo
```

Activamos sudo para los usuarios:
## Activate sudo for users
```
# vim /etc/sudoers
%wheel ALL=(ALL) ALL
```
Y listo! ya con esto ya podemos iniciar sesión como usario 
## Configurando Pacman
para configurar Pacman es necesario editar el archivo que se encuentra en `/etc/pacman.conf`.
Agregamos este huevo de pascua que tiene pacman! cuando uses pacman vas a notar de que se trata
```
# vim /etc/pacman.conf

ILoveCandy
```

## Update system and reboot
```
# sudo pacman -Syu && reboot
``` 

## Iniciando sesión como usario regular

```
$ sudo pacman-key init
```

Y finalmente tienes Arch Linux instalado en tu máquina!
Ahora solo queda personalizarlo exactamente como te gustaría usarlo, podria ser instalar un entorno gráfico, un administrador de ventanas o incluso usarlo así como está.

---
## ¿Comentarios? 
Escríbeme un [tweet](https://twitter.com/51v4n)!