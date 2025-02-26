# Guía 2025 para Instalar Gentoo en UEFI para Systemd (QEMU VM)

## 1. Creación de Imagen y Configuración de QEMU (corregido)

```bash
qemu-img create -f qcow2 gentoo_vm.qcow2 50G

qemu-system-x86_64 -enable-kvm -m 6G -smp 4 \
  -drive file=gentoo_vm.qcow2,format=qcow2,if=virtio \
  -cdrom install-amd64-minimal-*.iso \
  -boot d \
  -netdev user,id=n1 -device virtio-net,netdev=n1 \
  -device virtio-rng-pci \
  -bios /usr/share/edk2/ovmf/OVMF_CODE.fd \  # Usar versión completa de OVMF
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2/ovmf/OVMF_VARS.fd
```

## 2. Particionamiento (gdisk)

```bash
gdisk /dev/vda
# Comandos:
# o (Crear nueva tabla GPT)
# n (Nueva partición)
# Partición 1: 
#   Primer sector: Enter (default)
#   Último sector: +512M
#   Tipo: EF00 (EFI System)
# n (Nueva partición)
# Partición 2:
#   Resto del espacio
#   Tipo: 8300 (Linux filesystem)
# w (Escribir cambios)
```

## 3. Formateo

```bash
mkfs.fat -F 32 -n EFI /dev/vda1  # -n para etiqueta
mkfs.ext4 -L GENTOO /dev/vda2
```

## 4. Montaje

```bash
mount /dev/vda2 /mnt/gentoo
mkdir -p /mnt/gentoo/boot/efi
mount /dev/vda1 /mnt/gentoo/boot/efi
```

## 5. Instalación Stage3 y Portage

```bash
# Descargar stage3 systemd
links https://www.gentoo.org/downloads/mirrors/

# Usar la reguion y mirror mejor para su caso.
# En mi caso la navegacion para stage3 y .DIGESTS es:
# US > https://gentoo.osuosl.org/ > releases > amd64 > autobuilds > current-stage3-amd64-desktop-systemd/  
# → Descargar archivo .tar.xz y .DIGESTS

# Verificar checksum manualmente:
cat stage3-*.DIGESTS.xz  | grep -A1 SHA512 | grep -E '.tar.xz$'

sha512sum stage3*.tar.xz

# Extraer stage3:

tar xpvf stage3-*.tar.xz --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo

# Descargar Portage usando links:
links https://www.gentoo.org/downloads/mirrors/
# US > https://gentoo.osuosl.org/ > snapshot
# → Seleccionar snapshot portage-latest.tar.xz

mkdir -p var/db/repos/gentoo && cd var/db/repos/gentoo

tar xpf portage-latest.tar.xz -C /mnt/gentoo/var/db/repos/gentoo --strip-components=1
```

## 6. Configuración make.conf (esencial)
> Editar /mnt/gentoo/etc/portage/make.conf:

```bash
MAKEOPTS="-j$(nproc + 1)"  
USE="systemd elogind"
EMERGE_DEFAULT_OPTS="--jobs=$(nproc) --load-average=$(nproc)"
GRUB_PLATFORMS="efi-64"
SYSTEMD_ECLASS_ENABLED="1"
```

## 7. Fstab (verificar UUIDs)

```bash
blkid  # Verificar UUIDs
genfstab -U /mnt/gentoo >> /mnt/gentoo/etc/fstab
```

## 8. Entrar al chroot

```bash
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount -t proc proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys && mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev && mount --make-rslave /mnt/gentoo/dev
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"

systemctl preset-all # Preparar servicios base systemd
```

## 9. Configurar Kernel (alternativa segura)

```bash
emerge --ask sys-kernel/gentoo-kernel-bin
```

## 10. Configuración de GRUB 

```bash
emerge --ask sys-boot/grub efibootmgr
```

# Verificar montaje EFI:

```
mount | grep efi  # debe mostrar /boot/efi
```

# Instalar GRUB:

```
grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable
grub-mkconfig -o /boot/grub/grub.cfg
```

# Verificar instalación EFI:

```
ls /boot/efi/EFI/BOOT/  # Debe contener BOOTX64.EFI y otros
```

## 11. Configuración de Red

```bash
emerge --ask net-misc/dhcpcd
systemctl enable dhcpcd
```

## 12. Usuario y Sudo

```bash
emerge --ask sudo
useradd -m -G users,wheel,audio,video <usuario>
passwd <usuario>

# Editar sudoers:
echo "%wheel ALL=(ALL) ALL" >> /etc/sudoers
```

## 13. Finalización (pasos críticos)
```bash
# Configurar zona horaria
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime

# Configurar hostname
echo "gentoo-pc" > /etc/hostname

sync

exit
umount -R /mnt/gentoo
reboot
```
