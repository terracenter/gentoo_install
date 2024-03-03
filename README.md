# Guía de instalación de Gentoo

## Iniciar entorno live-cd

Lo primero que necesitamos para instalar nuestro Gentoo es un entorno live-cd con UEFI vars habilitado.

Para asegurarnos de que arrancamos en modo UEFI, ejecutando:

```shell
efivar -l
```

Si el comando enumera las variables UEFI, estamos listos para ir :checkered_flag:

## Preparar disco duro

```shell
parted -l
```
```
Model: WDC PC SN530 SDBPMPZ-512G-1101 (nvme)
Disk /dev/nvme0n1: 512GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:
```
```bash
parted /dev/nvme0n1 mklabel gpt 
```
```bash
parted -a opt /dev/nvme0n1 mkpart ESP fat32 1MB 512MB
```
```bash
parted /dev/nvme0n1 set 1 esp on
```
```bash
parted -a opt /dev/nvme0n1 mkpart primary  512MB 100%
```
```bash
parted /dev/nvme0n1 set 2 lvm on
```
```bash
parted -l
```
```
Number  Start   End    Size   File system  Name     Flags
 1      1049kB  512MB  511MB  fat32        ESP      boot, esp
 2      512MB   512GB  512GB               primary  lvm
```
### Crear volúmenes LVM

Lo bueno de los volúmenes lógicos es que puedes modificarlos en cualquier momento. En este ejemplo, voy a crear los volúmenes lógicos, uno para:

- `root`     
- `home`     
- `var`
- `var/tmp`
- `opt`
- `tmp`
- `swap`

 **NOTA**:
   
   Pero siéntete libre de crear tantos como quieras.

Además, usaré el nombre vg para identificar este grupo de volúmenes. Este es un nombre trivial, así que de nuevo, se creativo :D

```shell
pvcreate /dev/nvme0n1p2
vgcreate vg /dev/nvme0n1p2
```

```shell
vgs
```

```shell
VG  #PV #LV #SN Attr   VSize    VFree   
vg   1   0   0 wz--n- <476.45g <476.45g
```

Si lo que ves tiene sentido, vamos a crear los volúmenes:

```shell
lvcreate --size 20G vg --name root
lvcreate --size 4G vg --name var
lvcreate --size 10G vg --name var_tmp
lvcreate --size 1G vg --name opt
lvcreate --size 2G vg --name tmp
lvcreate --size 2G vg --name swap
lvcreate --size 100G vg --name home
```

Para volver a comprobar lo que hemos hecho, ejecuta:
```shell
lvs
```

Deberías ver algo como:

```shell
    LV      VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home    vg -wi-a----- 100.00g                                                    
  opt     vg -wi-a-----   1.00g                                                    
  root    vg -wi-a-----  20.00g                                                    
  swap    vg -wi-a-----   2.00g                                                    
  tmp     vg -wi-a-----   2.00g                                                    
  var     vg -wi-a-----   4.00g                                                    
  var_tmp vg -wi-a-----  10.00g
```

### Crear sistemas de archivos
Lo bueno de Linux es que tenemos un montón de sistemas de archivos que podemos usar, pero esto también es lo malo :sweat_smile:  "qué elegir" y "cuándo" es muy probable que te venga a la cabeza en algún momento.

Puedes leer todo lo que quieras [aquí](https://wiki.gentoo.org/wiki/Filesystem), pero lo haremos fácil. Usaremos `ext4`, el sistema de archivos por defecto de muchas distribuciones de Linux.

Vamos a crear nuestros volúmenes principales de sistemas de ficheros, uno FAT32 para UEFI (:fearful: sí, lo sé, pero así es como funciona UEFI), y luego nuestros principales sistemas de ficheros `ext4`. Si has creado más volúmenes, recuerda crear un sistema de ficheros para todos ellos:

```shell
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.ext4 -F /dev/mapper/vg-root
mkfs.ext4 -F /dev/mapper/vg-home
mkfs.ext4 -F /dev/mapper/vg-opt
mkfs.ext4 -F /dev/mapper/vg-tmp
mkfs.ext4 -F /dev/mapper/vg-var
mkfs.ext4 -F /dev/mapper/vg-var_tmp
mkswap /dev/vg/swap
```

### Montar los nuevos sistemas de archivos

```shell
mkdir -p /mnt/gentoo
mount /dev/mapper/vg-root /mnt/gentoo
mkdir -p  /mnt/gentoo/{home,opt,tmp,var}
mount /dev/mapper/vg-home /mnt/gentoo/home
mount /dev/mapper/vg-opt /mnt/gentoo/opt
mount /dev/mapper/vg-tmp /mnt/gentoo/tmp
mount /dev/mapper/vg-var /mnt/gentoo/var
mkdir /mnt/gentoo/var/tmp
mount /dev/mapper/vg-var_tmp /mnt/gentoo/var/tmp
swapon /dev/vg/swap
```
NOTA:

  - Si `/tmp/` y `/var/tmp` necesita residir en una partición separada, asegúrate de cambiar sus permisos después de montarlo:

    ```
    chmod 1777 /mnt/gentoo/tmp
    chmod 1777 /mnt/gentoo/var/tmp
    ```
## Instalación de los archivos de instalación de Gentoo
```shell
date
```

Seleccione la zona horaria:

```shell
tzselect
```
A continuación, utiliza NTP para sincronizar la hora y la fecha:
```shell
chronyd -q
```

### Instalar el tarball stage3

Primero descargamos el tarball:

```shell
cd /mnt
wget -c https://distfiles.gentoo.org/releases/amd64/autobuilds/20240229T194908Z/stage3-amd64-systemd-mergedusr-20240229T194908Z.tar.xz
```
### Instalación del archivo de stage
```shell
tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner
```

### Configuración de las opciones de compilación

```shell
nano -w /mnt/gentoo/etc/portage/make.conf
```

```shell
COMMON_FLAGS="-march=native -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
```
Ahora vamos a configurar `MAKEOPTS`. `MAKEOPTS` describe el número de trabajos paralelos utilizados por Portage.

Siempre ha habido alguna discusión sobre qué establecer [aquí](https://wiki.gentoo.org/wiki/MAKEOPTS), pero yo voy a seguir lo que recomienda la wiki de Gentoo, que es: `menos o igual al mínimo del tamaño de RAM/2GB o del número de hilos de la CPU`.

Utilizaremos el número de hilos de la CPU en ejecución:

```shell
lscpu | awk '/^CPU\(s\):/ {print $2}'
```
Además, para mantener la capacidad de respuesta del sistema al compilar, el sistema de compilación permite limitar la carga máxima de compilación con el parámetro `--load-average`.

Con todo lo que hemos dicho en esta sección, vamos a editar el archivo `make.conf` de nuevo:

```shell
nano /mnt/gentoo/etc/portage/make.conf
```

And this time, we set `MAKEOPTS`:

```shell
MAKEOPTS="-j6 -l6"
```

## Instalación del sistema base Gentoo
### Chrooting
#### Copiar información DNS

Copie la información DNS de su entorno live-cd de trabajo en el nuevo sistema para asegurarse de que podremos resolver los nombres de dominio una vez que cambiemos a él:

```shell
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

#### Montaje de los sistemas de archivos necesarios
```bash
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```


##### Selección de espejos
```shell
mirrorselect -s3 -b10 -D >> /mnt/gentoo/etc/portage/make.conf
```
```shell
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```



#### Entrar en el nuevo entorno

Chroot en el nuevo entorno:

```shell
chroot /mnt/gentoo /bin/bash
```

```shell
source /etc/profile
export PS1="(chroot) $PS1"
```

##### Preparación de un cargador de arranque
```bash
mkdir /efi
mount /dev/nvme0n1p1 /efi
```


### Montaje de los sistemas de archivos necesarios
Además del sistema de ficheros LVM que creamos en nuestro disco local, durante el arranque del sistema se crean otros pseudo-sistemas de ficheros necesarios para hacer chroot en nuestro nuevo entorno:

:warning: Si está configurando un entorno sin `SystemD`, puede omitir las líneas `--make-rslave`

### Entrar en el nuevo entorno

¡¡¡Genial!!! Estamos dentro de nuestro sistema Gentoo :tada: Por desgracia, todavía necesita un poco más de tiempo de cocción :cake:.

Por favor, respiren hondo y sigamos rodando.

### Configuración de Portage
### Actualización del árbol Portage

A continuación, actualice la instantánea con la última versión del repositorio:

```shell
emerge-webrsync -v
emerge --sync -v
```

### Elegir el perfil adecuado

Un perfil Portage especifica valores por defecto para banderas USE globales y por paquete, especifica valores por defecto para la mayoría de las variables que se encuentran en `/etc/portage/make.conf`, y define un conjunto de paquetes del sistema. Los desarrolladores de Gentoo mantienen los perfiles como parte del árbol Portage.

Enumera los perfiles disponibles:

```shell
eselect profile list | grep stable | grep systemd
```
De la lista de salida, debemos seleccionar nuestra mejor opción de ajuste. Como el sistema que estamos construyendo está basado en SystemD, el mejor perfil que podemos elegir es systemd. Por supuesto, podemos elegir cualquier otro que tenga `systemd`, como `default/linux/amd64/17.0/desktop/gnome/systemd`, pero queremos que se configure un entorno mínimo.

From the output list, we must select our best fitting option. Because the system we're building is based on `SystemD`, the best profile we can choose is the `systemd`. Of course, we can choose any other that has `systemd` in it, like `default/linux/amd64/17.0/desktop/gnome/systemd`, but we want a minimal environment to be configured.

```shell
[..]
 [17]  default/linux/amd64/17.1/systemd/merged-usr (stable) *
[..]
```
Este debería ser el perfil seleccionado por defecto, como podemos ver por el `*` al final. Pero si no es así, asegúrese de elegirlo con `eselect`:


```shell
eselect profile set 17 # o el número que tiene en su lista
```

### Configuración de la variable USE

Como hemos dicho, las banderas [USE flags](https://wiki.gentoo.org/wiki/USE_flag) son una característica central de Gentoo. Por lo tanto, una buena comprensión de cómo tratar con ellos es necesaria para tener un sistema Gentoo personalizado y saludable.

As we said,  are a core feature of Gentoo. Therefore, a good understanding of how to deal with them is needed to have a customized and healthy Gentoo system.

Las variables USE pueden definirse para todo el sistema o por dominio de paquete. Puede obtener más información aquí:

- Todo el sistema en [/etc/portage/make.conf](https://wiki.gentoo.org/wiki//etc/portage/make.conf#USE)
- Por envase en [/etc/portage/package.use](https://wiki.gentoo.org/wiki//etc/portage/package.use)

Podemos ver la lista de banderas USE que están configuradas en nuestro sistema ejecutando:

```shell
emerge --info | grep ^USE
```

No te asustes por la lista. Lo más probable es que la aumentes cuando acabemos con esta sección.

Podemos encontrar una descripción de todas las banderas USE en `less /var/db/repos/gentoo/profiles/use.desc`.

Además, la utilidad llamada `quse` del paquete `portage-utils` puede decirnos qué paquete utiliza qué banderas USE.

For example, if you want to know which packages use the `systemd` flag we simply need to run:

```shell
quse systemd
```
Por ejemplo, si queremos saber qué paquetes utilizan la bandera `systemd` sólo tenemos que ejecutar:

| :hand: Reminder                                                                                                                                                                                                                                                                                                                                                                                           |
| :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| En este punto es normal si te sientes abrumado. Establecer y elegir los USE flags adecuados antes de instalar nada en nuestro sistema nos ahorrará tiempo más adelante. Es crucial recordar que las banderas USE que usamos siempre se pueden cambiar, y si elegimos algo que ya no queremos usar o cometimos un error, siempre podemos cambiarlas y volver a compilar los paquetes afectados. Así que, relájate :relaxed: |

Hemos dicho que vamos a utilizar una herramienta llamada `ufed` que nos ayudará a establecer las banderas USE que queramos en cualquier dominio..

```shell
emerge -aqv app-portage/eix app-portage/ufed
```
Y luego, ejecútalo para seleccionar las banderas USE con una :smile: interfaz de usuario 😄 Este paso te llevará algo de tiempo, así que ¡coge una :tea: o :coffee y hagámoslo!

```shell
ufed
```

| :warning: Required USE flags                                                                                                         |
| :----------------------------------------------------------------------------------------------------------------------------------- |
| Para asegurarnos de que nuestra configuración funcionará como se espera, necesitamos al menos estas banderas USE establecidas `cryptsetup` y `lvm` |



### Configuring the CPU Flags

USE flags that are available for our system. We could use the advantages of some CPU instruction sets using what are called CPU flags. You can read more on the [Gentoo wiki page](https://wiki.gentoo.org/wiki/CPU_FLAGS_X86).

To know what optimizations we can use for our CPU, we're going to use a tool called `cpuid2cpuflags`:

```shell
emerge -q app-portage/cpuid2cpuflags
```

And then run it to get those values:

```shell
cpuid2cpuflags
```

You should see an output like:

```shell
CPU_FLAGS_X86: aes avx avx2 avx512_bitalg avx512_vbmi2 avx512_vnni avx512_vp2intersect avx512_vpopcntdq avx512bw avx512cd avx512dq avx512f avx512ifma avx512vbmi avx512vl f16c fma3 mmx mmxext pclmul popcnt rdrand sha sse sse2 sse3 sse4_1 sse4_2 ssse3 vpclmulqdq
```
```bash
echo "*/* $(cpuid2cpuflags)" > /etc/portage/package.use/00cpu-flags
```

### Re-compile and update @world

After setting up the USE and CPU flags, we're ready to re-compile and update all packages that we have installed in our base system before moving forward:

# VIDEO_CARDS y ACCEPT_LICENSE 
```shell
nano  /etc/portage/make.conf
```
VIDEO_CARDS="intel"
ACCEPT_LICENSE="*"


```shell
emerge -aq --verbose --update --deep --newuse @world
```

## Configuring the base system


```

### Timezone

Set our time zone. We can use the tool `tzselect` to interactively select our country, and it will tell us what the value of `/etc/timezone` file should be.

```shell
tzselect
```

Take the last line from the output and add it to the timezone file like:

```shell
ln -sf ../usr/share/zoneinfo/America/Caracas /etc/localtime
```

### Configure locales

```shell
nano -w /etc/locale.gen
```
```bash
es_VE ISO-8859-1
es_VE.UTF-8 UTF-8
en_US ISO-8859-1
en_US.UTF-8 UTF-8
```

```shell
locale-gen
```
```shell
eselect locale list
```
```shell
 eselect locale set 9
```
```bash
nano /etc/env.d/02locale
```
```
LANG="es_VE.utf8"
LC_COLLATE="C.UTF-8"
```
```bash
env-update && source /etc/profile && export PS1="(chroot) ${PS1}"
```


### Configuring Linux kernel


## Configuración del núcleo Linux
### Installing a distribution kernel
```bash
echo ">=sys-kernel/installkernel dracut" > /etc/portage/package.use/installkernel
```
```bash
emerge -q  sys-kernel/gentoo-kernel-bin
```

### Instalación del núcleo
#### Grub
```bash
echo "sys-apps/systemd boot" > /etc/portage/package.use/systemd
```
```bash
echo "sys-kernel/installkernel dracut grub" > /etc/portage/package.use/installkernel
```
```bash
emerge -aqv sys-apps/systemd
```

## Configurar el sistema
### Información del sistema de archivos
```bash
nano etc/fstab
```
```
/dev/nvme0n1p1          /efi            vfat    umask=0077              0 2
/dev/mapper/vg-root     /               ext4    defaults,noatime        0 1
/dev/mapper/vg-home     /home           ext4    defaults,noatime        0 1
/dev/mapper/vg-opt      /opt            ext4    defaults,noatime        0 1
/dev/mapper/vg-tmp      /tmp            ext4    defaults,noatime        0 1
/dev/mapper/vg-var      /var            ext4    defaults,noatime        0 1
/dev/mapper/vg-var_tmp  /var/log        ext4    defaults,noatime        0 1
/dev/mapper/vg-swap     none            swap    sw                      0 0
```
```bash
mount -a
```

### Hostname

echo le > /etc/hostname



### Network

echo ">=net-wireless/wpa_supplicant-2.10-r3 dbus" > /etc/portage/package.use/wpa_supplicant 

 emerge -aqv net-misc/networkmanager

systemctl enable NetworkManager

### Información del sistema
#### Clave root
passwd

#### Configuración de inicio y arranque
systemd-machine-id-setup

systemd-firstboot --prompt

systemctl preset-all --preset-mode=enable-only

systemctl preset-all

## Instalación de herramientas del sistema

### Registrador del sistema
 Systemd incluye un registrador integrado llamado servicio systemd-journald. El servicio systemd-journald es capaz de manejar la mayor parte de la funcionalidad de registro descrita en la sección anterior del registrador del sistema. Es decir, la mayoría de las instalaciones que ejecutarán systemd como gestor de sistemas y servicios pueden omitir con seguridad la adición de utilidades syslog adicionales.

Vea man journalctl para más detalles sobre el uso de journalctl para consultar y revisar los registros del sistema.

Por varias razones, como en el caso de reenviar los registros a un host central, puede ser importante incluir mecanismos redundantes de registro del sistema en un sistema basado en systemd. Esta es una ocurrencia irregular para la audiencia típica del manual y se considera un caso de uso avanzado. Por lo tanto, no está cubierto por el manual.

### Demonio Cron
 Los temporizadores de systemd pueden ejecutarse a nivel de sistema o a nivel de usuario e incluyen la misma funcionalidad que un demonio cron tradicional. A menos que se necesiten capacidades redundantes, instalar un programador de tareas adicional como un demonio cron es generalmente innecesario y puede omitirse con seguridad.

### Opcional: Indexación de ficheros
emerge -q sys-apps/mlocate

### Opcional: Acceso shell remoto
systemctl enable sshd

### Opcional: Shell completion
emerge -q app-shells/bash-completion

### Sincronización horaria

emerge -q net-misc/chrony
systemctl enable chronyd.service

### Herramientas del sistema de archivos
emerge -q sys-fs/e2fsprogs  sys-fs/dosfstools sys-block/io-scheduler-udev-rules

## Configuración del gestor de arranque
### Por defecto: GRUB
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf

emerge -q sys-boot/grub

emerge -q sys-fs/lvm2

systemctl enable lvm2-monitor.service

nano /etc/default/grub

GRUB_CMDLINE_LINUX="dolvm"


grub-install --efi-directory=/efi

grub-mkconfig -o /boot/grub/grub.cfg

### crear usuario
useradd -m -G users,wheel,audio -s /bin/bash freddy
passpasswd freddy

### Sudo 
emerge -q sudo

nano /etc/sudoers

 %wheel ALL=(ALL:ALL) ALL

    ```shell
    emerge --ask sys-apps/pciutils
    ```

2. Configure the kernel by running `make menuconfig` inside `/usr/src/linux`:

    ```shell
    cd /usr/src/linux
    make menuconfig
    ```

3. After the required options are activated either as a module or part of the kernel binary, compile the kernel and the modules, and install them by:

    ```shell
    make && make modules_install
    make install
    ```

4. Install `sys-kernel/dracut`. This package will help you create an `initrmfs` (Init Ram Filesystem) with the required tools to make the kernel bootable:

    ```shell
    emerge --ask sys-kernel/dracut
    ```

5. Run `dracut` to generate an `initramfs` for our kernel version:

    ```shell
    dracut --kver=5.15.59-gentoo
    ```

##### Semi-automatic set up using genkernel

For those who prefer a more pleasant initial experience, I'll explain how to use `genkernel`.

What `genkernel` really does is configure a generic kernel that works with most of the hardware. Like the LiveCD that we're using does.

1. Install `genkernel`:

    ```shell
    emerge --ask sys-kernel/genkernel
    ```

2. Genkernel needs the `/boot` entry in the `/etc/fstab` file. So go to the [fstab](#fstab) section now and continue with point 2 when you're done.

3. Edit `/etc/genkernel.conf`:

    ```shell
    nano -w /etc/genkernel.conf
    ```

    Ensure that `LVM` and `LUKS` are set to `yes`; otherwise, the system will not boot. Leave the rest of the options as they are:

    ```shell
    # Add LVM support
    LVM="yes"

    # Add LUKS support
    LUKS="yes"
    ```

4. Once `genkernel` is configured, then run to generate the kernel binary:

    ```shell
    genkernel all
    ```

#### Configuring the modules

If we need to auto-load a kernel module each time to system boots, we should specify it in `/etc/conf.d/modules` file.

You can list your available modules with:

```shell
find /lib/modules/<kernel version>/ -type f -iname '*.o' -or -iname '*.ko' | less
```

### LVM Configuration

Install lvm2 tools if it is not yet installed:

```shell
emerge --ask sys-fs/lvm2
```

Then edit the package configurations:

```shell
nano -w /etc/lvm/lvm.conf
```

```shell
use_lvmetad = 1
issue_discards = 1
volume_list = ["vg"] # Our VG volume name, check with vgdisplay
```

### Fstab

Before editing `fstab` we need to know which UUID are using our devices inside and outside `lvm` and `luks` volumes:

```shell
blkid /dev/mapper/vg-root | awk '{print $2}' | sed 's/"//g'
UUID="576e229c-cf68-4010-8d85-ff8149158416"
blkid /dev/mapper/vg-home | awk '{print $2}' | sed 's/"//g'
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"
```

Then edit `/etc/fstab`:

```shell
nano -w /etc/fstab
```

```shell
/dev/sda1                                       /boot   vfat    noatime                                         1 2
UUID="576e229c-cf68-4010-8d85-ff8149158416"     /       ext4    discard,noatime,commit=600,errors=remount-ro    0 1
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"     /home   ext4    discard,noatime,commit=600                      0 0
tmpfs                                           /var/tmp tmpfs  nodev,nosuid                    0 0
tmpfs                                           /tmp    tmpfs   nodev,nosuid                    0 0
```

### Configuring crypttab

Warning!!! As we don't have encrypted partitions other than root, which the `systemd` must mount before the whole system start, we don't need to set it up there, so our `crypttab` must be empty.

### Configuring mtab

In the past, some utilities wrote information (like mount options) into `/etc/mtab`; thus, it was supposed to be a regular file. Nowadays all software is supposed to avoid this problem. Still, before switching the file to become a symbolic link to /proc/self/mounts.

To create the symlink, run:

```shell
ln -sf /proc/self/mounts /etc/mtab
```

### Systemd boot (bootloader)

Systemd-boot is a simple UEFI boot manager which executes configured EFI images. The default entry is selected by an on-screen menu.

It is simple to configure, but can only start EFI executables, such as the Linux kernel EFISTUB, UEFI Shell, GRUB, and go on.

Before installing the EFI binaries we need to check if EFI variables are accessible:

```shell
emerge --ask sys-libs/efivar
efivar -l
```

Verify that you have mounted /boot:

```shell
mount | grep boot
```

Once got it mounted then install the systemd-boot binaries:

```shell
bootctl --path=/boot install
```

This command will copy the systemd-boot binary to your EFI System Partition ($esp/EFI/systemd/systemd-bootx64.efi and $esp/EFI/Boot/BOOTX64.EFI - both of which are identical - on x64 systems) and add systemd-boot itself as the default EFI application (default boot entry) loaded by the EFI Boot Manager.

Every time there's a new version of the systemd you should copy the new binaries to that System Partition by running:

```shell
bootctl --path=/boot update
```

### Add bootloader entries

Add one entry into bootloader with this options:

```shell
nano -w /boot/loader/entries/gentoo.conf
```

```shell
title    Gentoo Linux
efi      /kernel-genkernel-x86_64-4.4.6-gentoo
options  initrd=/initramfs-genkernel-x86_64-4.4.6-gentoo crypt_root=/dev/sda2 root=/dev/mapper/vg-root root_trim=yes init=/usr/lib/systemd/systemd ro dolvm
```

Edit default loader:

```shell
nano -w /boot/loader/loader.conf
```

```shell
default gentoo
timeout 3
```

### Efibootmgr

Efibootmgr is not a bootloader itself it's a tool that interacts with the EFI firmware of the system, which itself is acting as a boot loader. With the efibootmgr application, boot entries can be created, reshuffled and updated.

First we need to install the package:

```shell
emerge --ask sys-boot/efibootmgr
```

To list the current boot entries:

```shell
efibootmgr -v
```

```shell
BootCurrent: 0003
Timeout: 1 seconds
BootOrder: 0003
Boot0000* Linux Boot Manager  HD(1,GPT,3eb8effe-8e1d-4670-987c-9b49b5f605b2,0x800,0x1ff801)/File(\EFI\systemd\systemd-bootx64.efi)
Boot0001* gentoo  HD(1,GPT,02f231b8-8f9a-471c-b3a9-dc7edb1bd70e,0x800,0xee000)/File(\EFI\gentoo\grubx64.efi)
Boot0003* Gentoo Linux  PciRoot(0x0)/Pci(0x1f,0x2)/Sata(2,32768,0)/HD(1,GPT,73f682fe-e07b-4870-be82-d85077f8aaa2,0x800,0x100000)/File(\EFI\systemd\systemd-bootx64.efi)
```

I'm only Gentoo in my system so I don't really need anything but the Gentoo entry so I just delete everything with:

```shell
efibootmgr -b <entry_id> -B
```

Once everything is delete we can add our new systemd-boot loader entry:

```shell
efibootmgr -c -d /dev/sda -p 2 -L "Gentoo" -l "\efi\boot\bootx64.efi"
```

Perfect, we're almost ready to reboot.

## Rebooting into our new Gentoo systemd

### Enable lvm2

```shell
systemctl enable lvm2-lvmetad.service
```

### Change the root password

While in chroot we need to change the root password of our new system just before rebooting it.

```shell
passwd
```

### And reboot to your new Gentoo system

Don't worry you don't need to cross your fingers, everything should goes fine :smirk:

```shell
exit # To exit from chroot
sync # To sync filesystems
reboot
```

## Post-installation

### Setting the Hostname

When booted using systemd, a tool called hostnamectl exists for editing /etc/hostname and /etc/machine-info. So we don't need to edit the file manually simlpy run:

```shell
hostnamectl set-hostname <hostname>
```

### Configuring Network

Let's plug our new Gentoo system to the world!! :earth_africa:

We can choose between two options, to use systemd as network manager or standalone one, I prefer networkmanager but here are the two options:

### Using systemd-networkd

systemd-networkd is useful for simple configuration of wired network interfaces. As it's disabled by default we need to configure it by creating a \*.network file under /etc/systemd/network.

Here is an example for a simple ethernet DHCP configuration:

```shell
nano -w /etc/systemd/network/50-dhcp.network
```

```shell
[Match]
Name=en*
[Network]
DHCP=yes
```

And then tell systemd to manage and start that service:

```shell
systemctl enable systemd-networkd.service
systemctl start systemd-networkd.service
```

### Using NetworkManager

Often NetworkManager is used to configure network settings. I personally use nmtui because it's easy and has a ncurses client. Just install it:

```shell
emerge --ask networkmanager
```

And now simply run the following command and follow a guided configuration process through nmtui:

```shell
nmtui
```

### Setting locales

Yes, you're right, we set the locales before but once booted with systemd, the tool localectl is used to set locale and console or X11 keymaps. So, let's set it again to be sure that everything goes fine.

To change the system locale, run the following command:

```shell
localectl set-locale LANG=en_US.utf8
```

Change the virtual console keymap:

```shell
localectl set-keymap es
```

And finally, to set the X11 layout:

```shell
localectl set-x11-keymap es
```

### Setting time and date

Time and date can be set using the timedatectl utility. That will also allow users to set up synchronization without needing to rely on net-misc/ntp or other providers than systemd's own implementation.

To set the local time of the system clock directly:

```shell
timedatectl set-time "yyyy-MM-dd hh:mm:ss"
```

To set time zone:

```shell
timedatectl list-timezones
timedatectl set-timezone Europe/Madrid
```

Set systemd-timesyncd as a simple SNTP daemon. Systemd-timesyncd that only implements a client side, focusing only on querying time from one remote server. It should be more than appropriate for most installations.

```shell
timedatectl set-ntp true
```

To check the status of the daemon:

```shell
timedatectl status
```

When starting, systemd-timesyncd will read the configuration file from /etc/systemd/timesyncd.conf. To add time servers or change the provided ones, uncomment the relevant line and list their host name or IP separated by a space. I'm using [the NTP pool project](http://www.pool.ntp.org) for the main servers and the default Gentoo as fallback:

```shell
nano -w /etc/systemd/timesyncd.conf
```

```shell
[Time]
NTP=0.europe.pool.ntp.org 1.europe.pool.ntp.org 2.europe.pool.ntp.org 3.europe.pool.ntp.org
FallbackNTP=0.gentoo.pool.ntp.org 1.gentoo.pool.ntp.org 2.gentoo.pool.ntp.org 3.gentoo.pool.ntp.org
```

### File indexing

In order to index the file system to provide faster file location capabilities we will install sys-apps/mlocate:

```shell
emerge --ask sys-apps/mlocate
```

To keep databse update we need to run `updatedb` often.

### Filesystem tools

Additional to the tools for managing ext2, ext3, or ext4 filesystems (sys-fs/e2fsprogs) which are already installed as a part of the *@system* set I like to install other filesystem utilities like VFAT, XFS, NTFS and so on:

```shell
emerge --ask sys-fs/xfsprogs sys-fs/exfat-utils sys-fs/dosfstools sys-fs/ntfs3g
```

### Adding a user for daily use

Until now we have done everything as root but working as root on a Unix/Linux system is dangerous and should be avoided as much as possible. Therefore it is strongly recommended to add a user for day-to-day use.

The groups the user is member of define what activities the user can perform. The following table lists a number of important groups:

|  Group  |                                 Description                                 |
| :-----: | :-------------------------------------------------------------------------: |
|  audio  |                    Be able to access the audio devices.                     |
|  games  |                           Be able to play games.                            |
| portage |               Be able to access portage restricted resources.               |
|   usb   |                       Be able to access USB devices.                        |
|  video  | Be able to access video capturing hardware and doing hardware acceleration. |
|  wheel  |                             Be able to use su.                              |

Once you selected which groups would you like to add simply run:

```shell
useradd -m -G users,wheel,audio,video,audio,usb -s /bin/bash <username>
```

To set the user password run:

```shell
passwd <username>
```

### Exec as root with sudo

Sudo is a way that a regular user could run commands as root user. It's very useful to avoid using root password each time we need to run a command which needs root perms. To install run:

```shell
emerge --ask app-admin/sudo
```

Then edit config file with command:

```shell
visudo
```

There's lot of examples inside the config file so we should not have any problem to set it up.

### Removing installation tarballs

With the Gentoo installation finished and the system rebooted, if everything has gone well, we can now remove the downloaded stage3 tarball from the hard disk. Remember that they were downloaded to the / directory.

```shell
rm /stage3-*.tar.bz2*
```

### Power consumption optimization with Powertop

```shell
emerge --ask sys-power/powertop
```

The first step we need to do is to calibrate it, it's quite long process but the most important is to keep our system until the whole process is finished.

```shell
powertop --calibrate
```

To apply automatically optimal settings at boot we need to create a new Systemd unit:

```shell
nano -w /etc/systemd/system/powertop.service
```

```shell
[Unit]
Description=Powertop tunings

[Service]
Type=oneshot
ExecStart=/usr/sbin/powertop --auto-tune

[Install]
WantedBy=multi-user.target
```

Enable at each boot:

```shell
systemctl enable powertop.service
```

### GPU

Prior to install xorg we will configure our video card, mine is Intel Generation 6 so we need to add this line into make.conf:

```shell
nano -w /etc/portage/make.conf
```

```shell
VIDEO_CARDS="intel i965"
```

Before installing xorg we need to configure it again by adding custom graphic card information:

```shell
nano -w /etc/X11/xorg.conf.d/20-intel.conf
```

```shell
Section "Device"
   Identifier  "Intel Graphics"
   Driver      "intel"
  Option      "AccelMethod"  "sna"
  Option      "DRI"          "3"
  Option      "Backlight"    "intel_backlight"
EndSection
```

### Input devices

We need to do the same with the input devices. I'm using a laptop so check if you need to add joystick, mouse, keyboard, ... :wink:

```shell
nano -w /etc/portage/make.conf
```

```shell
INPUT_DEVICES="evdev synaptics
```

### X server

At the time to write this guide wayland is available but I'ld like to use bspwm which only support xorg so I'll continue with xorg server installation and configuration:

```shell
emerge --ask xorg-server
```

### A bunch of useful stuff

```shell
emerge --ask app-admin/ccze app-arch/unp app-editors/vim app-eselect/eselect-awk app-misc/screen app-shells/gentoo-zsh-completions app-shells/gentoo-zsh-completions app-vim/colorschemes app-vim/eselect-syntax app-vim/genutils app-vim/ntp-syntax media-gfx/feh sys-process/htop x11-terms/rxvt-unicode
```

### Portage nice value

We're running portage with modified scheduling priority, to not impact whole system performance during compilation.

```shell
echo 'PORTAGE_NICENESS="15"' >> /etc/portage/make.conf
```

### Setting portage branches

Branch defines if portage use stable or testing packages. Every package in the portage tree has it stable and testing version. The ACCEPT_KEYWORDS variable defines what software branch to use on the system. It defaults to the stable software branch for the system's architecture, for instance amd64.

I recommend to stick with the stable branch. However, if stability is not that much important for you and/or want to help out Gentoo by submitting bug reports to [https://bugs.gentoo.org](https://bugs.gentoo.org), then the testing is your way.

There are two ways to approach the testing branch for packages:

1. System wide setting: to make or system set to testing branch.

>```shell
>nano -w /etc/portage/make.conf
>```
>```shell
>ACCEPT_KEYWORDS="~amd64"
>```

2. Per package setting: we can set testing branch only for particular packages (This is and example, set whatever you want).

>```shell
>nano -w /etc/portage/package.accept_keywords
>```
>```shell
>sys-kernel/gentoo-sources
>sys-power/powertop
>app-admin/pass
>```

### Masked and unmasked packages

Masking a package the way where Gentoo Developers block a package version from being auto installed. The reason why that package is masked is mentioned in the package.mask file (situated in /usr/portage/profiles/ by default). But if we still wants to use this package, then add the desired version (usually this will be the exact same line from the package.mask file in the profile) to the /etc/portage/package.unmask file (or in a file in that directory if it is a directory).

```shell
nano -w /etc/portage/package.unmask
```

```shell
>=app-emulation/docker-compose-1.5.2
>=app-emulation/docker-swarm-1.1.3
>=app-emulation/docker-1.7.1
```

It is also possible to ask Portage not to take a certain package or a specific version of a package into account. To do so, mask the package by adding an appropriate line to the /etc/portage/package.mask location (either in that file or in a file in this directory).

```shell
nano -w /etc/portage/package.mask
```

```shell
=sys-kernel/gentoo-sources-4.5.0
```

### Overlays

Overlays contain additional packages for your Gentoo system while the main repository contains all the software packages maintained by Gentoo developers, additional package trees are usually hosted by repositories. Users can add such additional repositories to the tree that are "laid over" the main tree - hence the name, overlays.

```shell
emerge --ask app-portage/layman
```

To list all available overlays simply run:

```shell
layman -L
```

To install an overlay run:

```shell
layman -a <overlay_name>
```

To keep your installed overlays up to date, run:

```shell
layman -S
```

#### Custom overlay

If we want to maintain a custom set of ebuilds we just only need to create a local overlay doing these few steps:

```shell
mkdir -p /usr/local/portage/{metadata,profiles}
echo '<overlay_name>' > /usr/local/portage/profiles/repo_name
echo 'masters = gentoo' > /usr/local/portage/metadata/layout.conf
chown -R portage:portage /usr/local/portage
```

Next, tell portage about the overlay:

```shell
mkdir -p /etc/portage/repos.conf
```

```shell
nano -w /etc/portage/repos.conf/local.conf
```

```shell
[<overlay_name>]
location = /usr/local/portage
auto-sync = no
```

### Virtualization (How to use Qemu & kvm)
Virtualization is widely use nowadays. Everybody wants to test some environments, apps, or simply have a virtualized machine with other operating systems. In order to do that we should use any virtualization platform available for desktop use such as virtualbox, kvm/qemu, xen or vmware. Among all these options I choose kvm/qemu because its performance and simplicity. So here we are!

First we need to check if our hardware support virtualization:

```shell
grep --color -E "vmx|svm" /proc/cpuinfo
```

We could also check if kvm device is available in our */dev* directory:

```shell
ls /dev/kvm
```

If both are available we can continue installing kvm, otherwise you shouldn't use virtualization on your machine.

#### Kernel options

Before continue building kvm with should check if we have some kernel options enabled as module or build-in.

```shell
[*] Virtualization  --->
    <*>   Kernel-based Virtual Machine (KVM) support
    <*>   KVM for Intel processors support <-- if we use Intel cpu, otherwise disable it
    <*>   KVM for AMD processors support <-- if we use AMD cpu, otherwise disable it
    <*>   Host kernel accelerator for virtio net
Device Drivers  --->
   [*] Network device support  --->
      [*]   Network core driver support
      <*>   Universal TUN/TAP device driver support
   [*] Networking support  --->
      Networking options  --->
         <*> The IPv6 protocol
         <*> 802.1d Ethernet Bridging
   File systems  --->
      <*> The Extended 4 (ext4) filesystem
      [*]   Ext4 Security Labels
```

Once we have kernel compiled and running with new options we can continue by installing package:

```shell
emerge --askv app-emulation/qemu
```

In order to run kvm as normal user and not as root we should add our user account to the *kvm* group:

```shell
gpasswd -a <username> kvm
```

Finally, to be sure that libvirt daemon is running all the time we should enable with systemd:

```shell
systemctl enable libvirtd.service
```

### Speed up the system with prelink

What is Prelink and how can it help me? I'm sure that most of you are asking this question right now well, most applications we have installed in our system use shared libraries. Every time a program call this libraries they need to be loaded into memory. As more libraries program needs as more time it takes to resolve all symbol references. So prelink simply "maps" this symbol references and makes applications run faster. Of course this is a summary of what it does, but it's enough for us.

The only thing we need to do is to prelink binaries every time we upgrade or install any new program o library. But don't worry, portage will automatically prelink our system for us each time we install a package if we have prelink installed in our system. That's great!!

So we will simply install prelink:

```shell
emerge --askv prelink
```

Then configure the package by running env-update and then editing config file:

```shell
env-update
```

Unfortunately we can't prelink files that were compiled by old versions of binutils. Be default prelink define a number of libraries and directories which as blacklisted to avoid prelinking them. We can add or remove directories in prelink config files:

```shell
nano /etc/prelink.conf
nano /etc/prelink.conf.d/*
```

Finally we will prelink our system with:

```shell
prelink -amR
```

Which is:

- **a**: prelink all binary files.
- **m**: conserve virtual memory space. If we have a lot of binaries to prelink it takes a lot of space during the process, this parameter will ensure that we not run out of memory.
- **R**: randomize the address ordering to enhance security against buffer overflows.

## Last notes

Although there's a lot of work to do, I stop this guide at that point which I think that is far from base system installation now you'd walk you path little padawan :smile:

Hope you enjoyed!
