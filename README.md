# Gentoo install guide

![gentoo_logo](images/200px-gentoo-logo-dark.svg.png)

- [Gentoo install guide](#gentoo-install-guide)
  - [Introduction](#introduction)
  - [Installation concerns](#installation-concerns)
  - [Start live-cd environment](#start-live-cd-environment)
    - [Prepare Hard disk](#prepare-hard-disk)
    - [Prepare encrypted container](#prepare-encrypted-container)
    - [Create LVM volumes](#create-lvm-volumes)
    - [Create filesystems](#create-filesystems)
    - [Mount the new filesystems](#mount-the-new-filesystems)
  - [Installing the Gentoo base system](#installing-the-gentoo-base-system)
    - [Install the stage3 tarball](#install-the-stage3-tarball)
    - [Configuring compile options](#configuring-compile-options)
    - [Selecting mirrors](#selecting-mirrors)
    - [Configuring the main Gentoo repository](#configuring-the-main-gentoo-repository)
    - [Copy DNS info](#copy-dns-info)
    - [Mounting the necessary filesystems](#mounting-the-necessary-filesystems)
    - [Entering the new environment](#entering-the-new-environment)
    - [Configuring Portage](#configuring-portage)
    - [Updating the Portage tree](#updating-the-portage-tree)
    - [Choosing the right profile](#choosing-the-right-profile)
    - [Configuring the USE variable](#configuring-the-use-variable)
    - [Configuring the CPU Flags](#configuring-the-cpu-flags)
    - [Re-compile and update @world](#re-compile-and-update-world)
  - [Configuring the base system](#configuring-the-base-system)
    - [Allow licenses for packages](#allow-licenses-for-packages)
    - [Timezone](#timezone)
    - [Configure locales](#configure-locales)
    - [Configuring Linux kernel](#configuring-linux-kernel)
      - [Installing external firmware](#installing-external-firmware)
      - [Installing the sources](#installing-the-sources)
        - [Manual set up](#manual-set-up)
        - [Semi-automatic set up using genkernel](#semi-automatic-set-up-using-genkernel)
      - [Configuring the modules](#configuring-the-modules)
    - [LVM Configuration](#lvm-configuration)
    - [Fstab](#fstab)
    - [Configuring crypttab](#configuring-crypttab)
    - [Configuring mtab](#configuring-mtab)
    - [Systemd boot (bootloader)](#systemd-boot-bootloader)
    - [Add bootloader entries](#add-bootloader-entries)
    - [Efibootmgr](#efibootmgr)
  - [Rebooting into our new Gentoo systemd](#rebooting-into-our-new-gentoo-systemd)
    - [Enable lvm2](#enable-lvm2)
    - [Change the root password](#change-the-root-password)
    - [And reboot to your new Gentoo system](#and-reboot-to-your-new-gentoo-system)
  - [Post-installation](#post-installation)
    - [Setting the Hostname](#setting-the-hostname)
    - [Configuring Network](#configuring-network)
    - [Using systemd-networkd](#using-systemd-networkd)
    - [Using NetworkManager](#using-networkmanager)
    - [Setting locales](#setting-locales)
    - [Setting time and date](#setting-time-and-date)
    - [File indexing](#file-indexing)
    - [Filesystem tools](#filesystem-tools)
    - [Adding a user for daily use](#adding-a-user-for-daily-use)
    - [Exec as root with sudo](#exec-as-root-with-sudo)
    - [Removing installation tarballs](#removing-installation-tarballs)
    - [Power consumption optimization with Powertop](#power-consumption-optimization-with-powertop)
    - [GPU](#gpu)
    - [Input devices](#input-devices)
    - [X server](#x-server)
    - [A bunch of useful stuff](#a-bunch-of-useful-stuff)
    - [Portage nice value](#portage-nice-value)
    - [Setting portage branches](#setting-portage-branches)
    - [Masked and unmasked packages](#masked-and-unmasked-packages)
    - [Overlays](#overlays)
      - [Custom overlay](#custom-overlay)
    - [Virtualization (How to use Qemu \& kvm)](#virtualization-how-to-use-qemu--kvm)
      - [Kernel options](#kernel-options)
    - [Speed up the system with prelink](#speed-up-the-system-with-prelink)
  - [Last notes](#last-notes)

## Introducción

Gentoo es una distribución Linux que, a diferencia de las distribuciones binarias como Arch, Debian y muchas otras, el software se compila localmente según las preferencias y optimizaciones del usuario.

El nombre Gentoo procede de la especie de pingüinos, conocidos por ser los más rápidos del mundo.

Llevo un rato usando Gentoo, y lo que más me gusta de Gentoo es:

* Es divertido
* La posibilidad de tenerlo todo bajo control
* Personalización profunda Uno aprende mucho sobre GNU/Linux usándolo.

Descargo de responsabilidad: Gentoo es la distribución perfecta para utilizar la cita de un antiguo adagio que dice: Un gran poder conlleva una gran responsabilidad. Recuerda que aunque Gentoo te da mucha flexibilidad y personalización, también requiere tiempo, dedicación y paciencia.

Si eres ese tipo de persona, habla claro y entra en :sunrise:

| :warning:  Descargo de responsabilidad                                                                                                                              |
| :------------------------------------------------------------------------------------------------------------------------------------------------ |
| Esta guía pretende ser una experiencia de aprendizaje, e intentaré explicar todos los pasos para que entiendas lo que estamos haciendo y por qué. |

## Problemas de instalación

> Detente antes de seguir leyendo. Es importante saber que esta guía sólo contempla una instalación con UEFI, disco crypt/luks, particionado lvm y sistema init systemd. Por favor, recuerda que si quieres una configuración diferente, no tomes esta guía paso a paso sino como una pauta general.

## Iniciar entorno live-cd

Lo primero que necesitamos para instalar nuestro Gentoo es un entorno live-cd con UEFI vars habilitado.

Para asegurarnos de que arrancamos en modo UEFI, ejecutando:

```shell
efivar -l
```

Si el comando enumera las variables UEFI, estamos listos para ir :checkered_flag:

### Preparar disco duro

Te sugiero que uses cualquiera con el que estés familiarizado, yo te mostraré el proceso usando `gdisk`, pero otros como `cgdisk` o `parted` harán el trabajo.

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

### Preparar contenedor encriptado

En este punto, tenemos el disco particionado. A continuación, queremos crear un contenedor cifrado que contendrá los volúmenes LVM.

Para obtener el mejor rendimiento al trabajar con contenedores encriptados, debemos utilizar un algoritmo de encriptación soportado por nuestra CPU. Debemos ser prudentes a la hora de elegirlo porque no podremos cambiarlo después. Por suerte para nosotros, `cryptosetup` está aquí :ok_hand:

Ejecutar:

```shell
cryptsetup benchmark
```

Deberías ver algo como esto:

```shell
# Tests are approximate using memory only (no storage IO).
PBKDF2-sha1      2685213 iterations per second for 256-bit key
PBKDF2-sha256    4843307 iterations per second for 256-bit key
PBKDF2-sha512    1934642 iterations per second for 256-bit key
PBKDF2-ripemd160 1086607 iterations per second for 256-bit key
PBKDF2-whirlpool  757641 iterations per second for 256-bit key
argon2i       7 iterations, 1048576 memory, 4 parallel threads (CPUs) for 256-bit key (requested 2000 ms time)
argon2id      7 iterations, 1048576 memory, 4 parallel threads (CPUs) for 256-bit key (requested 2000 ms time)
#     Algorithm |       Key |      Encryption |      Decryption
        aes-cbc        128b      1762.8 MiB/s      6652.8 MiB/s
    serpent-cbc        128b       111.2 MiB/s       767.8 MiB/s
    twofish-cbc        128b       271.7 MiB/s       488.4 MiB/s
        aes-cbc        256b      1401.0 MiB/s      5413.3 MiB/s
    serpent-cbc        256b       112.9 MiB/s       785.0 MiB/s
    twofish-cbc        256b       277.4 MiB/s       475.3 MiB/s
        aes-xts        256b      5438.5 MiB/s      5523.9 MiB/s
    serpent-xts        256b       714.5 MiB/s       692.0 MiB/s
    twofish-xts        256b       451.1 MiB/s       447.8 MiB/s
        aes-xts        512b      4724.3 MiB/s      4778.2 MiB/s
    serpent-xts        512b       744.7 MiB/s       703.9 MiB/s
    twofish-xts        512b       453.0 MiB/s       455.3 MiB/s
```
Elija la opción con mejor rendimiento global. Por ejemplo, en la salida anterior sería `aes-xts` y el tamaño de la clave `PBKDF2-sha256`.
```
aes-xts        256b      5438.5 MiB/s      5523.9 MiB/s
```
Cuando haya elegido la suya, proceda a ejecutarla:

```shell
cryptsetup -v --cipher aes-xts-plain64 --key-size 256 -y luksFormat /dev/nvme0n1p2
```
El comando le pedirá que confirme con un `YES`. Después de un segundo, debería ver algo como `Command successful`.

```
Enter passphrase for /dev/nvme0n1p2: 
Verify passphrase:
```
A continuación, tenemos que abrir (descifrar) el contenedor para empezar a crear los volúmenes en su interior.

Cuando abrimos el contenedor, necesitamos especificar un nombre de etiqueta para identificar el contenedor una vez abierto. Como puedes ver, en el comando de abajo, he usado **cryptcontainer**, sé creativo :stuck_out_tongue_winking_eye:

```shell
cryptsetup open --type luks /dev/nvme0n1p2 cryptcontainer
```

Y... hemos terminado con la encriptación. Ahora, ¡hora del volumen! :sound:

### Crear volúmenes LVM

| :information_source: Punto de información                                                                                                                                             |
| :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Para aquellos de ustedes que no están familiarizados con este acrónimo, LVM significa Logical Volume Management. Puedes leer más sobre ello [aquí](https://wiki.gentoo.org/wiki/LVM). |

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

Además, usaré el nombre vg0 para identificar este grupo de volúmenes. Este es un nombre trivial, así que de nuevo, se creativo :D

```shell
pvcreate /dev/mapper/cryptcontainer
vgcreate vg0 /dev/mapper/cryptcontainer
```
Está bien siempre y cuando veas un `Physical volume "/dev/mapper/cryptcontainer" successfully created.` siguiente.


At this point, let's check that what we have is what we want:

```shell
vgs
```

You should see something like:

```shell
VG  #PV #LV #SN Attr   VSize    VFree   
vg0   1   0   0 wz--n- <476.45g <476.45g
```

Si lo que ves tiene sentido, vamos a crear los volúmenes:

```shell
lvcreate --size 20G vg0 --name root
lvcreate --size 4G vg0 --name var
lvcreate --size 10G vg0 --name var_tmp
lvcreate --size 1G vg0 --name opt
lvcreate --size 2G vg0 --name tmp
lvcreate --size 2G vg0 --name swap
lvcreate --size 100G vg0 --name home
```

Para volver a comprobar lo que hemos hecho, ejecuta:
```shell
lvs
```

Deberías ver algo como:

```shell
    LV      VG  Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  home    vg0 -wi-a----- 100.00g                                                    
  opt     vg0 -wi-a-----   1.00g                                                    
  root    vg0 -wi-a-----  20.00g                                                    
  swap    vg0 -wi-a-----   2.00g                                                    
  tmp     vg0 -wi-a-----   2.00g                                                    
  var     vg0 -wi-a-----   4.00g                                                    
  var_tmp vg0 -wi-a-----  10.00g
```
Si es así, bien hecho :muscle: ¡Sigamos rodando!


### Crear sistemas de archivos
Lo bueno de Linux es que tenemos un montón de sistemas de archivos que podemos usar, pero esto también es lo malo :sweat_smile:  "qué elegir" y "cuándo" es muy probable que te venga a la cabeza en algún momento.

Puedes leer todo lo que quieras [aquí](https://wiki.gentoo.org/wiki/Filesystem), pero lo haremos fácil. Usaremos `ext4`, el sistema de archivos por defecto de muchas distribuciones de Linux.

Vamos a crear nuestros volúmenes principales de sistemas de ficheros, uno FAT32 para UEFI (:fearful: sí, lo sé, pero así es como funciona UEFI), y luego nuestros principales sistemas de ficheros `ext4`. Si has creado más volúmenes, recuerda crear un sistema de ficheros para todos ellos:

```shell
mkfs.vfat -F32 /dev/nvme0n1p1
mkfs.ext4 -F /dev/mapper/vg0-root
mkfs.ext4 -F /dev/mapper/vg0-home
mkfs.ext4 -F /dev/mapper/vg0-opt
mkfs.ext4 -F /dev/mapper/vg0-tmp
mkfs.ext4 -F /dev/mapper/vg0-var
mkfs.ext4 -F /dev/mapper/vg0-var_tmp
mkswap /dev/vg0/swap
```

### Montar los nuevos sistemas de archivos

Con nuestros volúmenes lógicos creados y nuestros sistemas de ficheros listos, montemos nuestras particiones para empezar a construir nuestro sistema Gentoo:

```shell
mkdir -p /mnt/gentoo
mount /dev/mapper/vg0-root /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount /dev/nvme0n1p1 /mnt/gentoo/boot
mkdir -p  /mnt/gentoo/{home,opt,tmp,var}
mount /dev/mapper/vg0-home /mnt/gentoo/home
mount /dev/mapper/vg0-opt /mnt/gentoo/opt
mount /dev/mapper/vg0-tmp /mnt/gentoo/tmp
mount /dev/mapper/vg0-var /mnt/gentoo/var
mkdir /mnt/gentoo/var/tmp
mount /dev/mapper/vg0-var_tmp /mnt/gentoo/var/tmp
swapon /dev/vg0/swap
```

## Instalación del sistema base Gentoo

Antes de instalar Gentoo, asegúrate de que la fecha y la hora están configuradas correctamente. Un reloj mal configurado puede conducir a resultados extraños en el futuro, y usted no quiere esto :)

Para comprobar la fecha actual del sistema, ejecuteÑ

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

Para evitar instalar Linux desde cero, los increíbles desarrolladores de Gentoo proporcionan una compilación de Fase 3, principalmente un entorno base-binario-semi-funcional-no-arrancable (no es broma :satisfied: ) creado para ahorrarnos toneladas de tiempo.

Vamos a coger ese entorno **base-binario-semi-operativo-no-arrancable**, y lo untaremos en nuestra estructura de directorios Gentoo. Esto creará todos los binarios y archivos necesarios para empezar a compilar nuestro sistema Gentoo.

Primero descargamos el tarball:

```shell
curl -o /mnt/gentoo/stage3-amd64-systemd.tar.xz -L https://mirror.bytemark.co.uk/gentoo//releases/amd64/autobuilds/current-stage3-amd64-systemd/stage3-amd64-systemd-20231210T170356Z.tar.xz
```
Y lo desempaquetamos en nuestro directorio raíz que está montado en el directorio `/mnt/gentoo`:


```shell
cd /mnt/gentoo/
tar xvf stage3-*.tar.xz --xattrs
```

En este punto, tenemos todos los archivos para empezar a configurar nuestro nuevo entorno Gentoo.

Ahora es cuando nuestra CPU empieza a entrar en pánico :worried:.

### Configuración de las opciones de compilación

Me gustaría presentarle `Portage`, la piedra angular de Gentoo.

Para aquellos que no lo sepan (todavía), Portage es el sistema de autocompilación y la herramienta de gestión de paquetes de Gentoo. Tomará el código fuente de cualquier cosa que queramos instalar (definido en el archivo `ebuild), lo compilará (basándose en las banderas de arquitectura de la CPU y qué características están disponibles para cada paquete con lo que se conoce como "banderas de uso"), y lo instalará en nuestro sistema. También existe la opción de descargar paquetes precompilados, pero a mí me gusta hacer trabajar duro a mi CPU :D

Esto puede resultar intimidante para algunos de ustedes, pero no se preocupen, lo haremos paso a paso.

Y las primeras son las banderas de la CPU. Los flags de CPU indican al compilador las opciones soportadas nativamente por nuestra CPU. Esto se traduce en que nuestra CPU, y no cualquier otra compilará los binarios del sistema.

La forma más fácil es ir a la wiki de Gentoo [aquí](https://wiki.gentoo.org/wiki/Safe_CFLAGS) y comprobar los mejores COMMON_FLAGS a utilizar para tu CPU, pero si estás interesado, también puedes hacerlo manualmente por:

``shell
nano -w /mnt/gentoo/etc/portage/make.conf
```

And set the `-march=tigerlake` (or the type you got) at the beginning of `COMMON_FLAGS` to something like:

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
MAKEOPTS="--jobs 8 --load-average 9"
```

### Selección de espejos
Gentoo utiliza el espejo de closes para sincronizar el índice de paquetes, por lo que configurar los mejores espejos para tu ubicación es esencial. Por suerte la herramienta `mirrorselect` va a hacer el trabajo duro por nosotros :D

:warning: Asegúrate de que tienes acceso a Internet desde tu live-cd:

```shell
mirrorselect -D -s4 -o >> /mnt/gentoo/etc/portage/make.conf
```
Ahora debería tener una entrada para GENTOO_MIRRORS en `/mnt/gentoo/etc/portage/make.conf`.

### Configuración del repositorio principal de Gentoo

Copie el archivo de configuración del repositorio Gentoo del paquete Portage:

```shell
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

### Copiar información DNS

Copie la información DNS de su entorno live-cd de trabajo en el nuevo sistema para asegurarse de que podremos resolver los nombres de dominio una vez que cambiemos a él:

```shell
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
```

### Montaje de los sistemas de archivos necesarios

Además del sistema de ficheros LVM que creamos en nuestro disco local, durante el arranque del sistema se crean otros pseudo-sistemas de ficheros necesarios para hacer chroot en nuestro nuevo entorno:

:warning: Si está configurando un entorno sin `SystemD`, puede omitir las líneas `--make-rslave`

```shell
mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run
```

### Entrar en el nuevo entorno

Chroot en el nuevo entorno:

```shell
chroot /mnt/gentoo /bin/bash
```

```shell
source /etc/profile
export PS1="(chroot) $PS1"
```
¡¡¡Genial!!! Estamos dentro de nuestro sistema Gentoo :tada: Por desgracia, todavía necesita un poco más de tiempo de cocción :cake:.

Por favor, respiren hondo y sigamos rodando.

### Configuración de Portage

Portage hace el trabajo pesado de la gestión de repositorios y paquetes. Todo, desde la resolución de dependencias, la construcción del código fuente y la instalación del software en nuestro sistema, lo hace Portage, utilizando algunas de las herramientas que proporciona, como `emerge`.

Primero, necesitamos sincronizar los repositorios remotos con el árbol local de Portage para saber qué paquetes están disponibles para ser instalados.

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
eselect profile list
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

```shell
emerge -aq --verbose --update --deep --newuse @world
```

## Configuring the base system

In addition to Portage, some other options should be configured before we can take our Gentoo out of the oven :pizza:

### Allow licenses for packages

Each package in our Gentoo system defines what kind of license it uses. Setting what licenses we accept in our system is crucial to avoid problems installing `free` or `binary redistributable` packages. Also, to get installed automatically without asking us every time they get installed or updated.

You can read more about Gentoo licenses [here](https://wiki.gentoo.org/wiki/License_groups#Existing_license_groups).

Licenses acceptance is set in our `/etc/portage/make.conf` in the variable `ACCEPT_LICENSE`, like:

```shell
ACCEPT_LICENSE="-* @FREE @BINARY-REDISTRIBUTABLE"
```

### Timezone

Set our time zone. We can use the tool `tzselect` to interactively select our country, and it will tell us what the value of `/etc/timezone` file should be.

```shell
tzselect
```

Take the last line from the output and add it to the timezone file like:

```shell
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

### Configure locales

Locales are a set of information that most programs use to determine the country and language of your system. Most users would use one or a few locales instead of all. This could save us compiling time, to avoid compiling all languages available for each package, and space on the filesystem by not installing man pages that we can't understand :)

Edit the `/etc/locale.gen` and uncomment the locale you are interested. Make sure you prioritize `UTF-8` locales because at least one is needed for the build system to work correctly.

```shell
nano -w /etc/locale.gen
```

Now, execute `locale-gen` to generate all locales specified in the previous file, and write them to the locale-archive `/usr/lib/locale/locale-archive`.

```shell
locale-gen
```

The more locales you select, the slower this process will be.

To check if the locales you've selected are active, run the following command and see if they are on the list. No worries if there're some locales you didn't choose, that's part of the system.

```shell
locale -a
```

Once the locale is available, let's ensure that it is the default one used in our system. Again we use `eselect` for this, now with the locale module:

```shell
eselect locale list
```

You should see something like:

```shell
Available targets for the LANG variable:
  [1]   C
  [2]   C.utf8
  [3]   POSIX
  [4]   en_US.utf8
  [5]   C.UTF8 *
  [ ]   (free form)
```

Pick what locale you would activate and then run the following command with the number that you want to select:

```shell
eselect locale set 4
```

Now reload the environment:

```shell
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```

### Configuring Linux kernel

While Portage is the core of Gentoo Linux system, the Linux kernel is the operating system's core. Therefore, this last offers an interface for programs to access the hardware via the modules, aka drivers.

The first step is installing the kernel source code. Gentoo's recommendation is, of course, the kernel package called `sys-kernel/gentoo-sources`. It is maintained by the Gentoo team and patched to fix security vulnerabilities and functionality problems and improve compatibility with rare system architectures. But, as always, there're many other options available.

To get a complete list of kernel sources with short descriptions can be found by searching with emerge:

```shell
emerge --search sources
```

#### Installing external firmware

Linux kernel is an open-source project, but some hardware manufacturers don't like the idea of open-source the code of its firmware (or modules), releasing a compiled binary version of them. These binaries can't be included in the Linux kernel repository, and they are packed in a package called `sys-kernel/linux-firmware`.

So, before compiling the kernel, it is essential to know that some devices require additional firmware to make the system run properly. Second, these are more common than we think, from network adapters, wireless cards, graphic cards, and CPUs.

For the most common firmware requirements, we will install the `sys-kernel/linux-firmware`:

```shell
emerge --ask sys-kernel/linux-firmware
```

Additionally, check the [Microocode page](https://wiki.gentoo.org/wiki/Microcode) in the Gentoo wiki to see if you need any additional firmware to make your CPU work properly. For example, intel CPUs are usually required to install the Intel microcode package `sys-firmware/intel-microcode`.

#### Installing the sources

We're not going to get "too crazy" and install Zen kernel (my favorite). Instead, we will install the default Gentoo kernel:

```shell
emerge --ask sys-kernel/gentoo-sources
```

Once installed, we have to select what kernel we want our system to use by using our magic wand `eselect` :zap:

```shell
eselect kernel list
```

You will probably see only one kernel on the list:

```shell
Available kernel symlink targets:
  [1]   linux-5.15.59-gentoo
```

So we go with that:

```shell
eselect kernel set 1
```

What this actually does is create a symlink `/usr/src/linux` to point to `/usr/src/linux-5.15.59-gentoo`

From this point, only two things are missing: configure and compile it. There are two ways to do that:

1. Manual configuration and automated build
2. Semi-automated by using `genkernel`
3. Fully automated via `distribution kernels` (not covered in this guide. Follow the official documentation [here](https://wiki.gentoo.org/wiki/Project:Distribution_Kernel))

##### Manual set up

The manual process it's intimidating for beginners because you can break your system if you miss specific options. In addition, the process involves a lot of reading and digging.

As much as I would love to spend time explaining every single option of the kernel, doing so will be time-consuming and at least double the size of this guide :bangbang:

So, if you feel brave, I will give you some hints:

1. Install `sys-apps/pciutils`. This package contains a tool called `lspci` that you will use to identify what PCI hardware you have in your system. You will have to enable all drivers on the kernel.

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
volume_list = ["vg0"] # Our VG volume name, check with vgdisplay
```

### Fstab

Before editing `fstab` we need to know which UUID are using our devices inside and outside `lvm` and `luks` volumes:

```shell
blkid /dev/mapper/vg0-root | awk '{print $2}' | sed 's/"//g'
UUID="576e229c-cf68-4010-8d85-ff8149158416"
blkid /dev/mapper/vg0-home | awk '{print $2}' | sed 's/"//g'
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
options  initrd=/initramfs-genkernel-x86_64-4.4.6-gentoo crypt_root=/dev/sda2 root=/dev/mapper/vg0-root root_trim=yes init=/usr/lib/systemd/systemd ro dolvm
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
