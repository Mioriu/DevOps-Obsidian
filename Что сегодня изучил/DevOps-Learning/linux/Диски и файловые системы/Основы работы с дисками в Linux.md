В Windows вы привыкли видеть диски как `C:`, `D:`, `E:`. В Linux всё устроено иначе — нет букв дисков, зато есть единое дерево каталогов. Сегодня разберёмся, как Linux работает с дисками, научимся их просматривать, монтировать и управлять пространством.

### Как Linux видит диски

В Linux все устройства, включая диски, представлены как файлы в директории `/dev` (от слова "devices"). Это может показаться странным, но на самом деле очень удобно — с диском можно работать как с обычным файлом.

Типичные имена дисков:

```ruby
/dev/sda  — первый SATA/SCSI диск
/dev/sdb  — второй SATA/SCSI диск
/dev/sdc  — третий SATA/SCSI диск

/dev/nvme0n1  — первый NVMe SSD
/dev/nvme1n1  — второй NVMe SSD

/dev/sda1  — первый раздел на диске sda
/dev/sda2  — второй раздел на диске sda
```

Один физический диск может быть побит на виртуальные разделы:

```bash
Физический диск /dev/sda (500 GB):
┌──────────────────────────────────────┐
│  sda1    │    sda2    │     sda3     │
│  /boot   │    swap    │     /home    │
│  1 GB    │    8 GB    │    491 GB    │
└──────────────────────────────────────┘
```

### Просмотр дисков и разделов

Самая удобная команда для просмотра дисков — `lsblk` (list block devices):

```dart
$ lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 238.5G  0 disk 
├─sda1   8:1    0   512M  0 part /boot/efi
├─sda2   8:2    0   237G  0 part /
└─sda3   8:3    0     1G  0 part [SWAP]
sdb      8:16   0   1.8T  0 disk 
└─sdb1   8:17   0   1.8T  0 part /home/data
```

Что мы видим:

- `NAME` — имя устройства
- `SIZE` — размер
- `TYPE` — тип (disk — диск, part — раздел)
- `MOUNTPOINT` — куда примонтирован

Для более подробной информации используйте флаг `-f`:

```delphi
$ lsblk -f
NAME   FSTYPE LABEL  UUID                                 MOUNTPOINT
sda                                                       
├─sda1 vfat          7B5C-8A21                           /boot/efi
├─sda2 ext4          a1b2c3d4-5678-90ab-cdef-123456789012 /
└─sda3 swap          98765432-abcd-ef01-2345-6789abcdef01 [SWAP]
```

### Проверка свободного места

Команда `df` (disk free) показывает, сколько места занято и свободно:

```bash
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2       233G   45G  176G  21% /
/dev/sda1       511M   35M  477M   7% /boot/efi
/dev/sdb1       1.8T  856G  875G  50% /home/data
tmpfs           7.8G     0  7.8G   0% /dev/shm
```

Флаг `-h` означает "human readable" — показывать размеры в удобном виде (GB, MB), а не в блоках.

Чтобы узнать, какие папки занимают больше всего места — `du -sh`:

```bash
$ du -sh /home/*
1.2G    /home/user1
456M    /home/user2
15G     /home/projects
89M     /home/backup
```

## Файловые системы

Файловая система — это способ организации данных на диске. Она определяет структуру каталогов, способ именования файлов, методы хранения метаданных (права доступа, временные метки, владельцы) и алгоритмы размещения данных на физическом носителе. Разные ФС имеют разные возможности:

- **ext4** — стандарт для Linux, надёжная и быстрая
- **xfs** — хороша для больших файлов
- **btrfs** — современная, с поддержкой снапшотов
- **ntfs** — для совместимости с Windows
- **fat32** — для флешек (работает везде, но файлы до 4GB)

Чтобы создать файловую систему в разделе:

```shell
# Создаём ext4 на /dev/sdb1
$ sudo mkfs.ext4 /dev/sdb1

# Создаём NTFS для совместимости с Windows
$ sudo mkfs.ntfs /dev/sdb1

# Создаём FAT32 для флешки
$ sudo mkfs.vfat -F 32 /dev/sdb1
```

**Внимание:** эти команды уничтожат все данные в разделе!

### Практический пример: подключение внешнего диска

Допустим, вы купили новый внешний диск для бэкапов. Вот полный процесс его подготовки:

```bash
# 1. Подключаем диск и смотрим, как он определился
$ lsblk
... 
sdc      8:32   0   2T  0 disk   # Вот наш новый диск!

# 2. Создаём таблицу разделов (если диск новый)
$ sudo fdisk /dev/sdc
Command: g   # Создать новую GPT таблицу
Command: n   # Новый раздел
# Принимаем значения по умолчанию (Enter, Enter, Enter)
Command: w   # Записать изменения

# 3. Создаём файловую систему
$ sudo mkfs.ext4 /dev/sdc1
Creating filesystem with 524288000 4k blocks...

# 4. Создаём точку монтирования
$ sudo mkdir /mnt/backup

# 5. Монтируем
$ sudo mount /dev/sdc1 /mnt/backup

# 6. Проверяем
$ df -h /mnt/backup
Filesystem      Size  Used Avail Use% Mounted on
/dev/sdc1       2.0T   28K  1.9T   1% /mnt/backup
```

### Проверка состояния диска

Для проверки здоровья диска используйте `SMART`:

```shell
# Установка утилиты
$ sudo apt install smartmontools  # Debian/Ubuntu
$ sudo yum install smartmontools  # RHEL/CentOS

# Быстрая проверка
$ sudo smartctl -H /dev/sda
SMART overall-health self-assessment test result: PASSED

# Подробная информация
$ sudo smartctl -a /dev/sda
```

### Полезные команды для работы с дисками

```shell
# Показать все блочные устройства с деревом
$ lsblk -t

# Показать UUID всех устройств
$ blkid

# Информация о разделах
$ sudo fdisk -l

# Использование диска по директориям (топ-10)
$ du -h / 2>/dev/null | sort -rh | head -10

# Найти большие файлы (больше 100MB)
$ find / -type f -size +100M 2>/dev/null

# Проверить скорость чтения диска
$ sudo hdparm -t /dev/sda

# Безопасно очистить диск
$ sudo dd if=/dev/zero of=/dev/sdX bs=1M status=progress
```