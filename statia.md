# Настройка шифрование на lmde
 Здесь будет описыватся, как я шизанулся и решил сделать себе шифрование... Годик назад, я и набрёл на статью на хабре в которой как пожилой человек умер от старости, и человеку (который писал статью) отдали ноутбук дедушки, под "достование" из него данных, и вот тогда у меня интересная мысль, появилось. А, что если мой ноут  просто пропадёт, я же не хочу чтоб его вскрыли. [https://habr.com/ru/articles/717494/] И тогда я загорелся желаннием зашифровать его. И вот к чему я за год пришёл. 

 Во первых, дистр для сия подделия буду использовать LMDE, так то, патчер пойдёт даже на бубунте (:}), но для меня lmde просто удобный. 
 Во вторых, я не хочу оставлять /boot не шифрованым, как это делают 80% дистров, с "шифрованием". Потому, что в буте есть ядро, а это ядро вскывается... так что вшыть какое нибудь говно, легко [https://habr.com/ru/articles/457542/] (Как пример взлом, через initramfs)
 В третьих, только grub. Потому, что тот же systemd-bootloader просто переносит /boot в /boot/efi, и там работает не systemd, а обычный initramfs, что как мы выяснили не безопастно.
 И напоследок, я буду юзать lvm (зачем?) потому, что он просто удобен для управление разделами, ну и плюс я не хочу выделять целлый раздел под swap.

 Также для примера я буду использовать диски /dev/sda, но вы меняете на свой тип диска, и номер раздела)


 ## Скачка Lmde 
 Скачиваем образ с (оффициального сайта)[https://linuxmint.com/download_lmde.php]
 и проверяем его на ошибки и правильность:
 ```
 wget https://mirrors.edge.kernel.org/linuxmint/debian/sha256sum.txt
 wget https://mirrors.edge.kernel.org/linuxmint/debian/sha256sum.txt.gpg
 gpg --keyserver hkp://keys.openpgp.org:80 --recv-key 27DEB15644C6B3CF3BD7D291300F846BA25BAE09
 sha256sum -b lmde-6-cinnamon-64bit.iso
 gpg --verify sha256sum.txt.gpg sha256sum.txt
 ```
 ## Базовая настройка

 /dev/sda
 1. Запускаем образ, и начинаем разметку диска (через cfdisk), нам нужно два раздела: (скрин 1)
 | --------------------------- | ------------ |
 | EFI         | 100 - 200 MB  |  /dev/sda1
 | LVM / CRYPT | Все остальное |  /dev/sda2
 | --------------------------- | ------------ |

 2. Создаем зашифрованый раздел, с lvm. 
 ```
 cryptsetup luksFormat /dev/sda2
 cryptsetup open /dev/sda2 cr_linux
 pvcreate /dev/mapper/cr_linux
 vgcreate vg /dev/mapper/cr_linux
 lvcreate -L 4G -n swap vg
 lvcreate -l 100%FREE -n root vg
 mkswap /dev/vg/swap
 mkfs.fat -F 32 /dev/sda1
 mkfs.ext4 /dev/vg/root
 ```
 (скрин 2, 3)
 3. Запускаем установщик expert mode с помощью команды 
 ```
 live-installer-expert-mode
 ```
 Проходим по пункт, и доходим до разметки диска. Нажимаем ручную разметку, а дальше продвинутый метод.
 (скрин 4, 5 )
 
 Терь заготовленый раздел монтируем в /target, а именно
 ```
 mkdir /target
 mount /dev/vg/root /target
 ```
 Снимаем галку с установки загрузчика.
 (7)
 После установки, нажимаем продолжить и закрываем установщик.
 
 ## GRUB
 1. Подготовка. Монтируем временные разделы в /mnt, а потом включаем swap
 ```
 umount -R /target
 mount /dev/vg/root /mnt
 mount --mkdir /dev/sda1 /mnt/boot/efi
 swapon /dev/vg/swap 
 mount --bind /dev /mnt/dev/
 mount --bind /dev/pts /mnt/dev/pts
 mount --bind /proc /mnt/proc
 mount --bind /sys  /mnt/sys
 mount --bind /run  /mnt/run
 mount --bind /etc/resolv.conf /mnt/etc/resolv.conf
 chroot /mnt
 ```

 2. Сборка GRUB.
 Теперь начинаем сборку:
 ```bash
 apt update
 apt install -y gnulib libdevmapper-dev libfreetype-dev gettext autogen git bison help2man texinfo efibootmgr libisoburn1 libisoburn-dev mtools pkg-config m4 libtool automake autoconf flex fuse3 libfuse3-dev gawk
 mv /usr/bin/mawk /usr/bin/mawk_bu
 ln -s /usr/bin/gawk /usr/bin/mawk
 cd /tmp
 ```
 Дальше склонируем репозиторий.
 ```bash
 git clone https://github.com/olafhering/grub.git 
 cd grub
 git clone https://github.com/somebodykot/grub-extras.git 
 git clone https://github.com/somebodykot/grub-improved-luks2-git.git 
 git clone https://github.com/coreutils/gnulib.git 
 ```
 Теперь применяем патчи, и собираем GRUB.
 ```bash
 patch -Np1 -i ./grub-improved-luks2-git/add-GRUB_COLOR_variables.patch
 patch -Np1 -i ./grub-improved-luks2-git/detect-archlinux-initramfs.patch 
 patch -Np1 -i ./grub-improved-luks2-git/argon_1.patch
 patch -Np1 -i ./grub-improved-luks2-git/argon_2.patch
 patch -Np1 -i ./grub-improved-luks2-git/argon_3.patch
 patch -Np1 -i ./grub-improved-luks2-git/argon_4.patch
 patch -Np1 -i ./grub-improved-luks2-git/argon_5.patch
 patch -Np1 -i ./grub-improved-luks2-git/grub-install_luks2.patch
 sed 's|/usr/share/fonts/dejavu|/usr/share/fonts/dejavu /usr/share/fonts/TTF|g' -i "configure.ac"
 sed 's| ro | rw |g' -i "util/grub.d/10_linux.in"
 sed 's|GNU/Linux|Linux|' -i "util/grub.d/10_linux.in"
 
 rm -rf ./grub-extras/lua

 export GRUB_CONTRIB=./grub-extras
 export GNULIB_SRCDIR=./gnulib
 CFLAGS=${CFLAGS/-fno-plt}

 ./bootstrap
 mkdir ./build_x86_64-efi
 cd ./build_x86_64-efi
 ../configure --with-platform=efi --target=x86_64 --prefix="/usr" --sbindir="/usr/bin" --sysconfdir="/etc" --enable-boot-time --enable-cache-stats --enable-device-mapper --enable-grub-mkfont --enable-grub-mount --enable-mm-debug --disable-silent-rules --disable-werror  CPPFLAGS="$CPPFLAGS -O2"
 make
 ```
 Устанавливаем и собираем конфиг GRUB. (Игнорируем ошибку связаную с EFI переменными)
 ```bash
 make DESTDIR=/ bashcompletiondir=/usr/share/bash-completion/completions install
 install -D -m0644 ../grub-improved-luks2-git/grub.default /etc/default/grub
 grub-mkconfig -o /boot/grub/grub.cfg
 grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB
 ```
 1. Делаем загрузочный образ.
  Создаём конфиг в /boot/grub/grub-pre.cfg, с такими параметрами:
  ```bash
  set crypto_uuid=blkid /dev/sda2 (сюда надо написать свой UUID зашифрованого диска)
  cryptomount -u $crypto_uuid
  set root=lvm/vg-root (а здесь надо написать ваш lvm раздел)
  set prefix=($root)/boot/grub
  insmod normal
  normal
  ```
  Создаём образ для загрузки и копируем его в /boot/efi.
  ```bash
  grub-mkimage -p /boot/grub -O x86_64-efi -c /boot/grub/grub-pre.cfg -o /tmp/grubx64.efi luks2 part_gpt cryptodisk gcry_rijndael argon2 gcry_sha256 ext2 lvm
  install -v /tmp/grubx64.efi /boot/efi/EFI/GRUB/grubx64.efi
  ```
  И напоследок монтируем переменые efi и добавляем загрузочную запись.
  ```bash
  exit 
  mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars
  sudo efibootmgr -c -d /dev/sda -p 1 -L "debian" -l "\EFI\GRUB\grubx64.efi"
  chroot /mnt
  ``` 
  Но тут важно указать правильно раздел с efi (Например, у меня efi на первом разделе     (/dev/sda1), значит я пишу -p 1. А если у вас efi на 2 (/dev/sda2), то пишите -p 2)

 ## Дополнительная настройка (Initramfs / fstab)
  1. fstab. Создание.
  ```
  #### Static Filesystem Table File
  UUID=id                                     /boot/efi   vfat    defaults     0   1
  UUID=id                                     /           ext4    defaults     0   1
  UUID=id                                     swap        swap    defaults     0   2
  ```
  1" EFI раздел (Пример: /dev/sda1).
  2" Раздел / (Пример: /dev/vg/root).
  3" Swap раздел (Пример: /dev/vg/swap).

 2. Initramfs. Ключи для рассшифровки.
 ```
 mkdir /etc/crypt.d/
 dd if=/dev/urandom of=/etc/crypt.d/key1.bin bs=1024 count=4
 chmod 0400 /etc/crypt.d/key1.bin
 cryptsetup luksAddKey --pbkdf pbkdf2 /dev/sda2 /etc/crypt.d/key1.bin 
 ```

 2. Initramfs. Настраиваем /etc/crypttab
 ```
 название для раздела UUID=id диска путь до ключа luks
 ```
 Пример написание:
 ```
 cr_linux UUID=8044773b-70b2-45f0-86c1-e3a4eb2f1076 /etc/crypt.d/key1.bin luks,discard
 ```

 3. Initramfs. Настраиваем конфиги initramfs
 В файле: /etc/cryptsetup-initramfs/conf-hook
 Раскомментируйте KEYFILE_PATTERN, и впишите туда путь до ключа, пример:
 ```
 KEYFILE_PATTERN=/boot/keyfile
 ```
 и в файл: /etc/initramfs-tools/initramfs.conf
 в конец, добавляем: UMASK=0077

 И в конце собираем initramfs и обновляем конфиг grub.
 ```
 update-initramfs -u
 grub-mkconfig -o /boot/grub/grub.cfg
 ```



 
