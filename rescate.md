mkdir -p /mnt/gentoo
mount /dev/mapper/vg-root /mnt/gentoo
#mkdir -p  /mnt/gentoo/{home,opt,tmp,var}
mount /dev/mapper/vg-home /mnt/gentoo/home
mount /dev/mapper/vg-opt /mnt/gentoo/opt
mount /dev/mapper/vg-tmp /mnt/gentoo/tmp
mount /dev/mapper/vg-var /mnt/gentoo/var
#mkdir /mnt/gentoo/var/tmp
mount /dev/mapper/vg-var_tmp /mnt/gentoo/var/tmp
swapon /dev/vg/swap

 

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --bind /run /mnt/gentoo/run
mount --make-slave /mnt/gentoo/run    

chroot /mnt/gentoo /bin/bash

source /etc/profile
export PS1="(chroot) $PS1"


mount /dev/nvme0n1p1 /efi


exit
cd
umount -l /mnt/gentoo/dev{/shm,/pts,}
umount -R /mnt/gentoo
reboot

