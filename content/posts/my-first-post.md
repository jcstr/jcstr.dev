---
title: "Así instalo Arch Linux"
date: 2020-10-27T21:31:37-06:00
draft: false
---

Te doy la bienvenida a mi primer post! el cual dedicare a la [comunidad Mexicana de Arch Linux](https://archlinux.mx).

Uno de los principales obstáculos que se encuentran en algunas comunidades de Software Libre y de Código Abierto, particularmente en México, es el no tener el dominio del idioma inglés, pero no hay ningún problema, para ello decidí escribir esta guía al español para poder ayudar a quienes tienen el interés de instalar **[Arch Linux](https://archlinux.org)** y así poder usar el mismo día.

## ¿Para quién está dirigida está guía?
* Para quienes ~~son expertos~~ tienen la actitud de hacer las cosas por sí mismos.
* Está guía contiene todos los pasos enfocados para una instalación directa en la máquina.

## ¿Qué obtendré al finalizar esta guía?
* Una instalación de Arch Linux _cifrada_ bajo el esquema [LVM-LUKS](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Encrypting_an_entire_system_(Espa%C3%B1ol)#LVM_sobre_LUKS).

(Osea, un sistema Linux con el disco cifrado {... para más seguridad :lock:}).

:zap:**Consejo**: Antes de instalar Arch Linux en tu máquina recomiendo que aprendas a familiarizarte con Arch desde una máquina virtual, ya sea con [Docker](https://picodotdev.github.io/blog-bitix/2014/11/como-instalar-y-guia-de-inicio-basica-de-docker/) o [Vagrant](https://fortinux.gitbooks.io/humble_tips/content/capitulo_1_usando_aplicaciones_en_linux/tutorial_instalar_vagrant_para_usar_ambientes_virtuales_en_gnulinux.html).


## Preliminares :checkered_flag:
Antes que nada, para poder instalar Arch Linux, necesitamos iniciar la imagen `.iso` desde una USB en la máquina que se desea instalar y para efectos prácticos, yo asumo que ya se tiene lista la USB cargada con la imagen de Arch Linux, además de que hay diferentes maneras de poder cargar la imagen de Arch en la USB yo sugiero [leer esta página de la wiki](https://wiki.archlinux.org/index.php/USB_flash_installation_medium_(Espa%C3%B1ol)) para conocer más respecto a este tema. 

## Configurando el teclado del instalador :cd:
Una vez que el instalador inició es momento de configurar el teclado acorde al idioma/región que tenemos en nuestro teclado, para ello yo asumo la configuración para `la-latin` (Español Latinoamericano):

```
# loadkeys la-latin1
```

Recomiendo [leer esta página de la wiki](https://wiki.archlinux.org/index.php/Linux_console_(Espa%C3%B1ol)/Keyboard_configuration_(Espa%C3%B1ol)) para ahondar más en la configuración del teclado.

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
Para crear la partición de arranque (`/dev/sda1/`) utilizaremos `gdisk`:

```
# gdisk /dev/sda
```
Pulsamos la tecla: `o`

Pulsamos la tecla: `y`

Puslamos la tecla: `n`

Pulsamos la tecla: `Enter`

Definimos el tamaño de la partición: `+512MIB`

Definimos la partición para [EFI](https://wiki.archlinux.org/index.php/EFI_system_partition_(Espa%C3%B1ol)): `ef00`

Pulsamos la tecla: `n`

Pulsamos la tecla: `enter`

Definimos el tipo de partición [LVM](https://wiki.archlinux.org/index.php/GPT_fdisk): `8e00`

Escribimos los cambios: `w`

Confirmamos: `y`

## Dandole formato a la partición boot
Para este caso en particular es necesario darle formato a la partición boot con el sistema de archivos FAT32 el cual es un requisito para poder instalar [systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot_(Espa%C3%B1ol)):
```
# mkfs.fat -F32 /dev/sda1
```
Y con esto ya tenemos lista la partición de arranque, ahora, procedemos a preparar las demás particiones y configurar el esquema LVM-LUKS.

## Cifrando Arch en nuestra máquina :lock:
A partir de aquí vamos a configurar `/dev/sda2` para usar el esquema LVM-LUKS.

Y antes de empezar conviene comprender, y bien, ¿por qué usar el esquema `LVM-LUKS`?

En palabras simples, LVM (Logical Volume Manager) nos ayuda a crear particiones virtuales mientras que LUKS (Linux Unified Key Setup) nos ayuda a cifrar la particion.

Así quedaría todo el disco una vez instalado Arch:
```
+-----------------------------------------------------------------------+ +----------------+
| Volumen lógico 1      | Volumen lógico 2      | Volumen lógico 3      | | Boot partition |
|                       |                       |                       | |                |
| [SWAP]                | /                     | /home                 | | /boot          |
|                       |                       |                       | |                |
| /dev/MyVolGroup/swap  | /dev/MyVolGroup/root  | /dev/MyVolGroup/home  | |                |
|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _|_ _ _ _ _ _ _ _ _ _ _ _| |                |
|                                                                       | |                |
|                         LUKS2 encrypted partition                     | |                |
|                           /dev/sda2                                   | | /dev/sda1      |
+-----------------------------------------------------------------------+ +----------------+
```
(diagrama tomado de la [wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)).

Ahora si, continuamos con la guía.

## Configurando el cifrado con [cryptsetup](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Device_encryption_(Espa%C3%B1ol)#Utilizaci%C3%B3n_de_cryptsetup)
Para poder cifrar la partición `/dev/sda2` necesitamos usar `cryptsetupt`, que nos ayuda a crear, acceder y administrar unidades de almacenamiento cifrados.

Para este caso simplemente ejecutamos:
```
# cryptsetup luksFormat /dev/sda2
```

Ingresamos la constraseña para cifrar la partición:
```
Enter passphrase for /dev/sda2:
```
Y listo! cryptsetup cifro nuestra particion.

## Desbloqueamos la particion cifrada y creamos root, home y swap
Para desbloquear la partición usamos cryptsetup:
```
# cryptsetup open --type luks /dev/sda2 lvm 
```
## Creamos los respectivos grupos 
Con pvcreate creamos un volumen físico necesario para usar junto con LVM: 
```
# pvcreate /dev/mapper/lvm
```
Ahora creamos el grupo de volumen bajo el mismo volumen físico que acabamos de crear: 
```
# vgcreate volume /dev/mapper/lvm
```
Con lvcreate creamos un volumen lógico dentro de un grupo de volumen existente.
Para este caso asignamos 2G para swap el tamaño de swap es proporcional a la memoria de cada máquina, puede ser más o puede ser menos, siguiendo la buena práctica se asigno 2G por que la máquina tiene 4G.
```
# lvcreate -L2G volume -n swap
```
Creamos el volumen lógico para root:
```
# lvcreate -L50G volume -n root
```
Y dejamos el espacio que queda libre para home:
```
# lvcreate -l 100%FREE volume -n home
```
Una vez creado el grupo y los volumenes lógicos, nos queda darle el formato a cada volumen lógico.

## Dandole el formato a los volumenes lógicos
Para `root` y `home` usaremos `ext4`:
```
# mkfs.ext4 /dev/mapper/volume-root
```
```
# mkfs.ext4 /dev/mapper/volume-home
```
Y para swap usaremos mkswap:
```
# mkswap /dev/mapper/volume-swap 
```

### Creamos los directorios home y boot sobre /mnt
```
# mkdir /mnt/home
# mkdir /mnt/boot
```
### Montamos las particiones
```
# mount /dev/mapper/volume-root /mnt
# mount /dev/sda1 /mnt/boot
# mount /dev/mapper/volume-home /mnt/home
```

### Activamos swap:
```
# swapon /dev/mapper/volume-swap
```

## Instalación base
Una vez montadas las particiones procedemos a instalar los paquetes necesarios para tener un sistema funcional en nuestra máquina:
```
# pacstrap /mnt base base-devel linux linux-firmware vim networkmanager mkinitcpio lvm2 cryptsetup
```

:zap:**Consejo**: recomiendo [leer esta pagina de la wiki](https://wiki.archlinux.org/index.php/Installation_guide#Installation) para conocer más a detalle sobre qué paquetes se pueden instalar para cada necesidad, esto se debe a que se han simplificado los paquetes que se incluyen en el paquete `base`. 

Para este caso decidí instalar `linux linux-firmware vim networkmanager mkinitcpio lvm2 cryptsetup`, paquetes necesarios para nuestra instalación cifrada.

## Generamos el archivo fstab
¿De qué nos sirve el archivo fstab? 
fstab es usado para definir cómo las particiones, distintas unidades de almacenamiento o sistemas de archivos remotos deben ser montados e integrados en el sistema.

```
# genfstab -U /mnt >> /mnt/etc/fstab
```

## Movemos root al nuevo sistema instalado
```
# arch-chroot /mnt
```

## Configuración inicial
En esta parte de la guía ya tenemos un sistema mínimo funcional, ahora solo nos queda terminar de configurar e instalar más paquetes necesarios.

## Configuramos la zona horaria 
```
# ln -s /usr/share/zoneinfo/America/Mexico_city /etc/localtime
# hwclock --systohc --utc
```

## Lenguaje
Para cambiar nuestro sistema de idioma (una vez que se instale) lo hacemo editando el archivo /etc/locale.gen y simplemente descomentamos el idioma que nos gustaría usar con Arch.
```
# vim /etc/locale.gen

es_MX.UTF-8 UTF-8
```
Habiendo configurado el idioma deseado proceedemos a generar los locales para nuestra instalación:
```
# locale-gen 
```

Guardamos nuestra configuración en `locale.conf`:
```
# locale > /etc/locale.conf
```

## Configuración de red

Hostname
```
# vim /etc/hostname
```
Agregar entradas a los hosts
```
# vim /etc/hosts
127.0.0.1	  localhost
::1		      localhost
127.0.1.1	  myhostname.localdomain	  myhostname
```

## IMPORTANTE :warning: 
Al estar configurando nuestra instalación bajo el esquema de cifrado LVM-LUKS tenemos que ordenar los HOOKS (scripts que se ejecutan el disco RAM inicial) de esta manera:
```
# vim /etc/mkinitcpio.conf 

HOOKS=(base udev autodetect modconf block keyboard encrypt lvm2 filesystems fsck)
```
¿Qué podría suceder si se ignora el orden de esto?
Arch no podría iniciar de manera correcta, nos bloquearía el uso del teclado y no podremos ingresar la contraseña para desbloquear nuestra partición.

## Recreamos initramfs
Para nuestra instalación cifrada es necesario recrear el initramfs (un esquema necesario para cargar un sistema de archivos temporal en la memoria, que se usa en el arranque de nuestro sistema Linux)
```
# mkinitcpio -P
```

## Establecemos la constraseña a root
```
# passwd 
```

## Instalamos el gestor de arranque (systemd-boot)

```
bootctl --path=/boot install
```
¿Por qué instalamos systemd-boot y no GRUB?
Por que systemd-boot esta pensado para el modo UEFI.

## Configuramos los cargadores de arranque (boot loaders)
En este archivo definimos qué sistema nos gustaría iniciar:
```
# vim /boot/loader/loader.conf

default arch 
timeout 3 
editor 0
```
En este otro archivo indicamos el título de nuestra entrada (Arch Linux), el kernel y en las optiones indicamos la partición cifrada `root`:
```
# vim /boot/loader/entries/arch.conf

title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options cryptdevice=UUID=86a872ee-b133-4e13-8283-d99024361d79:volume root=/dev/mapper/volume-root quiet rw
```
:zap:**Consejo**: Para obtener el UUID del cryptdevice dentro de `vim` con este comando:
``` 
:read ! blkid /dev/sda2 
``` 

## Salimos de arch-chroot, desmontamos las particiones y reiniciamos
`Ctrl+D` 

```
# umount -R /mnt 
# reboot
```
A este punto ya tenemos un sistema base funcional en nuestra máquina (es momento de desconectar la USB), ya solo nos quedan algunos detalles por configurar.

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

Activamos sudo para todos los usuarios:

```
# vim /etc/sudoers
%wheel ALL=(ALL) ALL
```
Y listo! ya con esto ya podemos iniciar sesión como usario y usar `sudo`.

## Configurando Pacman
Para configurar Pacman es necesario editar el archivo que se encuentra en `/etc/pacman.conf`.

¿Qué podemos configurar en ese archivo?
* Directorios para root, la base de datos de pacman, caché, logs, gpg
* Cambiar el modo en que pacman nos muestra algunas opciones como el tamaño total de la descarga de paquetes, usar el log, etc.
* Agregar/Quitar repositorios

Agregamos este huevo de pascua que tiene `pacman`! cuando uses pacman vas a notar de que se trata
;)
 
 ```
# vim /etc/pacman.conf

ILoveCandy
```

## Actualizamos y reiniciamos 
```
# sudo pacman -Syu && reboot
``` 

## Iniciando sesión como usario regular
Una vez que iniciamos sesión como usuario regular activamos el llavero de pacman:
```
$ sudo pacman-key init
```

Y finalmente tienes Arch Linux instalado en tu máquina!


## Haz que Arch funcione como a ti te guste 
Ahora solo queda personalizarlo exactamente como a ti te gustaría usarlo, podria ser instalar un [entorno gráfico](https://wiki.archlinux.org/index.php/Desktop_environment_(Espa%C3%B1ol)), un [administrador de ventanas](https://wiki.archlinux.org/index.php/Window_manager_(Espa%C3%B1ol)) o incluso dejarlo así como está. Esa es la parte más genial de Arch Linux, tú eliges como usarlo :D 

Haz llegado al final de esta guía, espero que te haya servido para poder instalar Arch Linux en tu máquina! 

<a href="https://twitter.com/gccstr?ref_src=twsrc%5Etfw" class="twitter-follow-button" data-show-count="false">Follow @gccstr</a><script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>