# Pasar el sistema del NAS a un SSD

El NAS tenia el sistema y los datos en un mismo disco de 3 TB. Lo pase a un SSD de 256 GB y el disco de 3 TB quedo solo para datos. Lo hice clonando con rsync en caliente y despues con un ultimo rsync con los servicios frenados. No reinstale nada.

Los pasos estan anotados de memoria. Revisar los nombres de disco antes de correr cualquier cosa, porque un `sdX` mal puesto borra el disco equivocado.

## Particiones del SSD

GPT con tres particiones. La de bios_grub es para que el SSD pueda arrancar en una maquina vieja con BIOS, y la ESP es para UEFI.

```
sudo parted /dev/sdX -- mklabel gpt
sudo parted /dev/sdX -- mkpart biosgrub 1MiB 2MiB
sudo parted /dev/sdX -- set 1 bios_grub on
sudo parted /dev/sdX -- mkpart ESP fat32 2MiB 514MiB
sudo parted /dev/sdX -- set 2 esp on
sudo parted /dev/sdX -- mkpart root ext4 514MiB 100%
sudo mkfs.vfat -F32 /dev/sdX2
sudo mkfs.ext4 /dev/sdX3
```

No le puse swap.

## Copia

```
sudo mount /dev/sdX3 /mnt/ssd
sudo mkdir -p /mnt/ssd/boot/efi
sudo mount /dev/sdX2 /mnt/ssd/boot/efi
sudo rsync -aAXH --numeric-ids --info=progress2 \
  --exclude={"/proc/*","/sys/*","/dev/*","/run/*","/tmp/*","/mnt/*","/media/*","/lost+found","/srv/archivos/*"} \
  / /mnt/ssd/
```

La carpeta compartida por Samba no entraba en el SSD, asi que la deje afuera de la copia. Se queda en el disco viejo.

## Que cambiar en la copia

* `/mnt/ssd/etc/fstab` con los UUID nuevos. Se sacan con `blkid`. La raiz y la ESP apuntan al SSD y el disco viejo pasa a montarse en `/mnt/disco`.
* La carpeta de Samba vuelve a su lugar con un bind, asi la config de Samba no cambia.

```
UUID=<uuid del disco viejo>  /mnt/disco     ext4  defaults,nofail  0 2
/mnt/disco/srv/archivos      /srv/archivos  none  bind,nofail      0 0
```

* Sin swap, el initramfs se queda esperando la particion de resume al arrancar. Se arregla con `RESUME=none` en `/etc/initramfs-tools/conf.d/resume`.
* Apague os-prober con `GRUB_DISABLE_OS_PROBER=true` en `/etc/default/grub`, porque si no le agrega al menu el sistema viejo del otro disco.

## GRUB

```
for d in dev proc sys run; do sudo mount --bind /$d /mnt/ssd/$d; done
sudo chroot /mnt/ssd
update-initramfs -u -k all
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable --no-nvram
update-grub
exit
```

Con `--removable` GRUB queda en `EFI/BOOT/BOOTX64.EFI`, que es la ruta que cualquier placa UEFI busca sola. Con `--no-nvram` no toca las entradas de arranque de la placa. Las dos cosas sirven porque el SSD despues va a ir a otra maquina.

Al principio le puse GRUB para BIOS y para UEFI a la vez. Despues de confirmar que arrancaba en UEFI deje solo el de UEFI y desinstale `grub-pc`.

## Ultimo paso

Primero arranque desde el SSD, con el disco viejo montado en `/mnt/disco`, y revise que todos los servicios levantaran. Despues frene los servicios y corri un ultimo rsync desde `/mnt/disco` hacia `/` para traer lo que habia cambiado mientras tanto. Desde ahi el sistema vivo es el del SSD y el del disco viejo quedo congelado. Lo dejo un tiempo por si hace falta volver y despues lo borro.

## Pendiente para cuando cambie la maquina

* La placa nueva tiene otra MAC, asi que hay que actualizar la reserva de DHCP y el WOL.
* Si la placa nueva tiene Secure Boot prendido, no va a arrancar. No instale shim.
