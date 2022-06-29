# otus-linux-day07
Загрузка Linux

# **Prerequisite**
- Host OS: Windows 10.0.19043
- Guest OS: CentOS 7.8.2003
- VirtualBox: 6.1.34
- Vagrant: 2.2.19

# **Содержание ДЗ**

* Вход в систему без пароля
* Переименовывание VolumeGroup в системе, установленной на LVM
* Добавление модуля в ininrd

# **Выполнение**

### Вход в систему без пароля

- **Способ 1.** init=/bin/sh

При выборе ядра для загрузки нажать `e`. Попадаем в окно где мы можем изменить параметры загрузки. 
Из параметров загрузки исключаются `rhgb quiet` для наблюдения загрузки. 
Добавляется `init=/bin/sh`
После загрузки корневая файловая система в режиме RO. Требуется перемонтировать в RW:
```
sh-4.2# mount -o remount,rw /
```
После перемонтирования:
```
sh-4.2# mount | grep root
/dev/mapper/centos-root on / type xfs (rw,relatime,attr2,inode64,noquota)
```

- **Способ 2.** rd.break

При выборе ядра для загрузки нажать `e`. Попадаем в окно где мы можем изменить параметры загрузки. 
Из параметров загрузки исключаются `rhgb quiet` для наблюдения загрузки. 
Добавляется `rd.break`. 
После загрузки в Emergency Mode корневая файловая система в режиме RO и смонтирована в `/sysroot`. 
Требуется перемонтировать в RW и перейти в неё:
```
switch_root:/# mount | grep /sysroot
/dev/mapper/centos-root on /sysroot type xfs (ro,relatime,attr2,inode64,noquota)
switch_root:/# mount -o remount,rw /sysroot
switch_root:/# chroot /sysroot
sh-4.2# 
```
Если требуется выполнить сброс пароля root, требуется обновить метки безопасности для SELinux:
```
sh-4.2# passwd root
sh-4.2# touch /.autorelabel
```

- **Способ 3.** rw init=/sysroot/bin/sh

При выборе ядра для загрузки нажать `e`. Попадаем в окно где мы можем изменить параметры загрузки. 
Из параметров загрузки исключаются `rhgb quiet` для наблюдения загрузки. 
Вместо `ro` выполняется замена на `rw init=/sysroot/bin/sh`
После загрузки корневая файловая система в режиме RW смонтирована в `/sysroot`. 
```
:/# mount | grep /sysroot
/dev/mapper/centos-root on /sysroot type xfs (rw,relatime,attr2,inode64,noquota)
```

### Переименовывание VolumeGroup в системе, установленной на LVM

Текущее состояние:
```
[root@localhost ~]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  centos   1   2   0 wz--n- <19.00g    0
```

Переименование Volume Group:
``` 
[root@localhost ~]# vgrename centos OtusRoot
  Volume group "centos" successfully renamed to "OtusRoot"
```

Правка `/etc/fstab` и состояние после правки:
```
[root@localhost ~]# sed -i 's/centos/OtusRoot/g' /etc/fstab
[root@localhost ~]# cat /etc/fstab

#
# /etc/fstab
# Created by anaconda on Tue Jun 21 11:14:17 2022
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/OtusRoot-root /                       xfs     defaults        0 0
UUID=e55d8e0d-bdc4-4e6a-ba4d-fd27e2d055cd /boot                   xfs     defaults        0 0
/dev/mapper/OtusRoot-swap swap                    swap    defaults        0 0
```

Правка `/etc/default/grub` и состояние после правки:
```
[root@localhost ~]# sed -i 's/centos/OtusRoot/g' /etc/default/grub
[root@localhost ~]# cat /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=OtusRoot/root rd.lvm.lv=OtusRoot/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```

Правка `/boot/grub2/grub.cfg` и состояние после правки:
```
[root@localhost ~]# sed -i 's/mapper\/centos/mapper\/OtusRoot/g' /boot/grub2/grub.cfg
[root@localhost ~]# sed -i 's/rd.lvm.lv=centos/rd.lvm.lv=OtusRoot/g' /boot/grub2/grub.cfg
[root@localhost ~]# cat /boot/grub2/grub.cfg | grep OtusRoot
	linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/OtusRoot-root ro crashkernel=auto rd.lvm.lv=OtusRoot/root rd.lvm.lv=OtusRoot/swap rhgb quiet LANG=en_US.UTF-8
	linux16 /vmlinuz-0-rescue-0cbc06930fcb314b9fa76ae712b8551f root=/dev/mapper/OtusRoot-root ro crashkernel=auto rd.lvm.lv=OtusRoot/root rd.lvm.lv=OtusRoot/swap rhgb quiet
```

### Добавление модуля в initrd

В каталоге ` /usr/lib/dracut/modules.d/` создаётся каталог для тестового модуля туда помещаются скрипты для модуля:
```
[root@localhost ~]# mkdir /usr/lib/dracut/modules.d/01test
[root@localhost ~]# cp /mnt/*.sh /usr/lib/dracut/modules.d/01test/
[root@localhost ~]# cd /usr/lib/dracut/modules.d/01test/
[root@localhost 01test]# ll
total 8
-rwxr-xr-x. 1 root root 126 Jun 29 06:10 module-setup.sh
-rwxr-xr-x. 1 root root 334 Jun 29 06:10 mtest.sh
[root@localhost 01test]# cat module-setup.sh 
#!/bin/bash

check() {
    return 0
}

depends() {
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh"
}
[root@localhost 01test]# cat test.sh 
#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'

Hello! You are in dracut module!

 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/'\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
```

Выполняется пересборка `initrd`:
```
 dracut -f -v
```

# **Результаты**

Полученный в ходе работы утилиты `script` файл `typescript` при переименовании Volume Group и создании модуля initrd 
помещен в публичный репозиторий.

- **GitHub** - https://github.com/jimidini77/otus-linux-day08
