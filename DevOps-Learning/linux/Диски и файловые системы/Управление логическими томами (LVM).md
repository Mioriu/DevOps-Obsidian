`LVM` (Logical Volume Manager) даёт гибкость, которой нет у обычных разделов.

### Проблемы обычных разделов

С обычными разделами есть **ограничения:**

- **Закончилось место** на разделе? Придётся **переразмечать** весь диск
- Нужно **объединить два диска** в один большой раздел? **Невозможно**
- Хотите **сделать снимок состояния** раздела? Нужны **сторонние инструменты**
- **Перенести раздел** на другой диск без остановки работы? **Очень сложно**

`LVM` решает все эти проблемы, добавляя слой абстракции между физическими дисками и файловыми системами.

### Архитектура `LVM`: три уровня

`LVM` состоит из **трёх уровней**, каждый со **своей ролью**:

```bash
    Файловая система (ext4, xfs...)
              ↑
    ┌─────────────────────────┐
    │   LV (Logical Volume)   │ ← То, что вы монтируете
    │    Логический том       │
    └─────────────────────────┘
              ↑
    ┌─────────────────────────┐
    │   VG (Volume Group)     │ ← Пул хранилища
    │    Группа томов         │
    └─────────────────────────┘
              ↑
    ┌────────┐ ┌────────┐ ┌────────┐
    │   PV   │ │   PV   │ │   PV   │ ← Физические диски
    │/dev/sda│ │/dev/sdb│ │/dev/sdc│
    └────────┘ └────────┘ └────────┘
```

### Разберём каждый уровень подробнее:

**PV (Physical Volume)** — физический том

Это обычный диск или раздел, подготовленный для `LVM`.

**VG (Volume Group)** — группа томов

Объединение нескольких PV в **единый пул** хранилища.

**LV (Logical Volume)** — логический том

Виртуальный раздел, создаваемый из пространства `VG`. Это то, что вы форматируете и монтируете.

### Наглядный пример работы LVM

Допустим, у вас есть три диска: `100GB, 200GB и 300GB`. Вот как их можно организовать:

```yaml
Обычные разделы:                  С использованием LVM:
┌──────────┐                      ┌──────────┐
│ sda 100G │                      │ sda 100G │──┐
├──────────┤                      └──────────┘  │
│ root 30G │                      ┌──────────┐  │
│ home 70G │                      │ sdb 200G │──┼──→ VG: storage (600GB)
└──────────┘                      └──────────┘  │         │
┌──────────┐                      ┌──────────┐  │         ├─ LV: root (50GB)
│ sdb 200G │                      │ sdc 300G │──┘         ├─ LV: home (300GB)
├──────────┤                      └──────────┘            ├─ LV: backup (200GB)
│ data 200G│                                              └─ свободно (50GB)
└──────────┘
┌──────────┐
│ sdc 300G │      Проблемы:                    Преимущества:
├──────────┤      • Фиксированные размеры      • Гибкие размеры
│backup 300│      • Нельзя использовать        • Можно расширять
└──────────┘        пространство между         • Единый пул места
                    дисками                    • Легко добавить диск
```

### Установка и проверка LVM

Сначала убедимся, что `LVM` установлен:

```shell
# Установка LVM (обычно уже есть)
$ sudo apt install lvm2          # Debian/Ubuntu
$ sudo yum install lvm2          # RHEL/CentOS

# Проверка версии
$ lvm version
  LVM version:     2.03.11(2)
  Library version: 1.02.175
  Driver version:  4.45.0
```

### Создание LVM: пошаговое руководство

Создадим `LVM-структуру` с нуля. Предположим, у нас есть **два диска**: `/dev/sdb (100GB) и /dev/sdc (200GB)`.

**Шаг 1: Создаём физические тома (PV)**

```bash
# Подготавливаем диски для LVM
$ sudo pvcreate /dev/sdb /dev/sdc
  Physical volume "/dev/sdb" successfully created.
  Physical volume "/dev/sdc" successfully created.

# Проверяем созданные PV
$ sudo pvs
  PV         VG Fmt  Attr PSize   PFree
  /dev/sdb      lvm2 ---  100.00g 100.00g
  /dev/sdc      lvm2 ---  200.00g 200.00g

# Подробная информация
$ sudo pvdisplay /dev/sdb
```

**Шаг 2: Создаём группу томов (VG)**

```shell
# Объединяем PV в группу томов с именем "storage"
$ sudo vgcreate storage /dev/sdb /dev/sdc
  Volume group "storage" successfully created

# Смотрим информацию о VG
$ sudo vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  storage   2   0   0 wz--n- 299.99g 299.99g

# Подробная информация
$ sudo vgdisplay storage
  --- Volume group ---
  VG Name               storage
  VG Size               299.99 GiB
  PE Size               4.00 MiB    # Physical Extent - базовый блок
  Total PE              76798
  Free PE               76798
```

**Шаг 3: Создаём логические тома (LV)**

```shell
# Создаём логический том для домашней директории
$ sudo lvcreate -n home -L 150G storage
  Logical volume "home" created.

# Создаём том для резервных копий
$ sudo lvcreate -n backup -L 100G storage
  Logical volume "backup" created.

# Создаём том, используя проценты свободного места
$ sudo lvcreate -n projects -l 50%FREE storage
  Logical volume "projects" created.

# Смотрим созданные LV
$ sudo lvs
  LV       VG      Attr       LSize   
  backup   storage -wi-a----- 100.00g
  home     storage -wi-a----- 150.00g
  projects storage -wi-a-----  25.00g
```

**Шаг 4: Создаём файловые системы и монтируем**

```shell
# Создаём файловые системы
$ sudo mkfs.ext4 /dev/storage/home
$ sudo mkfs.ext4 /dev/storage/backup
$ sudo mkfs.xfs /dev/storage/projects

# Создаём точки монтирования
$ sudo mkdir -p /mnt/{home,backup,projects}

# Монтируем
$ sudo mount /dev/storage/home /mnt/home
$ sudo mount /dev/storage/backup /mnt/backup
$ sudo mount /dev/storage/projects /mnt/projects

# Проверяем
$ df -h | grep storage
/dev/mapper/storage-home     147G   61M  140G   1% /mnt/home
/dev/mapper/storage-backup    98G   61M   93G   1% /mnt/backup
/dev/mapper/storage-projects  25G   45M   25G   1% /mnt/projects
```

### Главное преимущество: изменение размера на лету

С `LVM` можно изменять размер томов **без остановки системы**:

**Увеличение логического тома:**

```bash
# Добавляем 50GB к тому home
$ sudo lvextend -L +50G /dev/storage/home
  Size of logical volume storage/home changed from 150.00 GiB to 200.00 GiB.
  Logical volume storage/home successfully resized.

# Расширяем файловую систему
$ sudo resize2fs /dev/storage/home    # Для ext4
$ sudo xfs_growfs /mnt/projects      # Для XFS

# Проверяем новый размер
$ df -h /mnt/home
/dev/mapper/storage-home  197G   61M  187G   1% /mnt/home
```

**Уменьшение логического тома (только для `ext4`):**

```shell
# ВАЖНО: сначала уменьшаем файловую систему!
$ sudo umount /mnt/backup
$ sudo e2fsck -f /dev/storage/backup
$ sudo resize2fs /dev/storage/backup 80G

# Теперь уменьшаем сам том
$ sudo lvreduce -L 80G /dev/storage/backup
  WARNING: Reducing active logical volume to 80.00 GiB.
  THIS MAY DESTROY YOUR DATA (filesystem etc.)
Do you really want to reduce storage/backup? [y/n]: y
  Size of logical volume storage/backup changed from 100.00 GiB to 80.00 GiB.

$ sudo mount /dev/storage/backup /mnt/backup
```

### Добавление нового диска в существующую VG

Новый диск? Легко добавить его в пул:

```shell
# Новый диск /dev/sdd (500GB)
$ sudo pvcreate /dev/sdd
  Physical volume "/dev/sdd" successfully created.

# Добавляем в существующую VG
$ sudo vgextend storage /dev/sdd
  Volume group "storage" successfully extended

# Смотрим результат
$ sudo vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  storage   3   3   0 wz--n- 799.99g 524.99g
           ↑                 ↑       ↑
       3 диска          Общий размер  Свободно
```

### Снимки (Snapshots)

`LVM` позволяет создавать моментальные снимки томов. Это как сохранение в игре — можно вернуться к этому состоянию:

```shell
# Создаём снимок тома home размером 10GB
$ sudo lvcreate -L 10G -s -n home_snapshot /dev/storage/home
  Logical volume "home_snapshot" created.

# Делаем рискованные изменения в /mnt/home...
# Что-то пошло не так?

# Восстанавливаемся из снимка
$ sudo umount /mnt/home
$ sudo lvconvert --merge /dev/storage/home_snapshot
  Merging of volume storage/home_snapshot started.
  
# Монтируем обратно
$ sudo mount /dev/storage/home /mnt/home
# Всё вернулось как было!
```

Снимки полезны для:

- Резервного копирования работающей системы
- Тестирования опасных изменений
- Создания согласованных бэкапов баз данных

### Перемещение данных между дисками

Нужно заменить старый диск? `LVM` позволяет перенести данные без остановки:

```bash
# Смотрим, где физически находятся данные
$ sudo pvs -o+pv_used
  PV         VG      Fmt  Attr PSize   PFree   Used
  /dev/sdb   storage lvm2 a--  100.00g  20.00g  80.00g
  /dev/sdc   storage lvm2 a--  200.00g 150.00g  50.00g
  /dev/sdd   storage lvm2 a--  500.00g 500.00g      0

# Перемещаем данные с /dev/sdb на другие диски
$ sudo pvmove /dev/sdb
  /dev/sdb: Moved: 10.00%
  /dev/sdb: Moved: 45.50%
  /dev/sdb: Moved: 100.00%

# Теперь можем безопасно удалить /dev/sdb из VG
$ sudo vgreduce storage /dev/sdb
  Removed "/dev/sdb" from volume group "storage"

# И удалить метку PV
$ sudo pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
```

### Thin Provisioning (тонкое выделение)

`LVM` поддерживает _тонкие_ тома, которые выделяют место по мере необходимости:

```shell
# Создаём пул для тонких томов
$ sudo lvcreate -L 100G --thinpool thin_pool storage
  Thin pool volume with chunk size 64.00 KiB created.

# Создаём тонкие тома (можем "переподписать" место)
$ sudo lvcreate -V 50G --thin -n thin_vol1 storage/thin_pool
$ sudo lvcreate -V 50G --thin -n thin_vol2 storage/thin_pool
$ sudo lvcreate -V 50G --thin -n thin_vol3 storage/thin_pool

# Создали 150GB томов из 100GB пула!
# Место выделяется только при реальной записи
```

### RAID через LVM

`LVM` может создавать `RAID-массивы` для надёжности:

```bash
# RAID 1 (зеркало) - данные дублируются
$ sudo lvcreate --type raid1 -m 1 -L 50G -n raid_mirror storage
  Logical volume "raid_mirror" created.

# RAID 5 (с контролем чётности) - нужно минимум 3 PV
$ sudo lvcreate --type raid5 -i 2 -L 100G -n raid5_vol storage
  Logical volume "raid5_vol" created.

# Проверка состояния RAID
$ sudo lvs -a -o name,copy_percent,devices storage
  LV                        Cpy%Sync Devices
  raid_mirror               100.00   raid_mirror_rimage_0(0),raid_mirror_rimage_1(0)
  [raid_mirror_rimage_0]            /dev/sdb(0)
  [raid_mirror_rimage_1]            /dev/sdc(0)
```

### Практический сценарий: миграция с обычных разделов на `LVM`

У вас есть сервер с **обычными разделами**, и вы хотите перейти на `LVM`:

**1. Исходная ситуация**

```bash
$ df -h
/dev/sda1   20G  15G  4.0G  79% /
/dev/sda2   80G  60G   16G  79% /home
```

**2. Добавляем новый диск `/dev/sdb` для `LVM`**

```bash
$ sudo pvcreate /dev/sdb
$ sudo vgcreate vg_new /dev/sdb
```

**3. Создаём `LV` такого же размера**

```bash
$ sudo lvcreate -n lv_home -L 80G vg_new
$ sudo mkfs.ext4 /dev/vg_new/lv_home
```

**4. Копируем данные**

```bash
$ sudo mkdir /mnt/new_home
$ sudo mount /dev/vg_new/lv_home /mnt/new_home
$ sudo rsync -avxHAX /home/ /mnt/new_home/
```

**5. Переключаемся на новый том**

```bash
$ sudo umount /home
$ sudo umount /mnt/new_home
$ sudo mount /dev/vg_new/lv_home /home
```

**6. Обновляем `/etc/fstab`**

```bash
$ sudo nano /etc/fstab
# Меняем /dev/sda2 на /dev/vg_new/lv_home
```

**7. Освобождаем старый раздел и добавляем в `LVM`**

```bash
$ sudo pvcreate /dev/sda2
$ sudo vgextend vg_new /dev/sda2
```

### Резервное копирование метаданных `LVM`

`LVM` **автоматически** сохраняет метаданные, но можно делать и **ручные копии**:

```shell
# Автоматические бэкапы находятся здесь
$ ls /etc/lvm/backup/
storage

# Ручное резервное копирование
$ sudo vgcfgbackup storage
  Volume group "storage" successfully backed up.

# Восстановление из бэкапа
$ sudo vgcfgrestore storage
  Restored volume group storage
```

### Типичные проблемы и решения

**1. "Device is busy" при удалении LV**

```shell
# Проверяем, что использует том
$ sudo lsof /dev/storage/test_lv
$ sudo fuser -vm /dev/storage/test_lv

# Принудительное отключение
$ sudo umount -l /mnt/test
$ sudo lvremove /dev/storage/test_lv
```

**2. Недостаточно места для снимка**

```shell
# Снимок переполнился и стал неактивным
$ sudo lvs
  LV            VG      Attr       LSize   
  home_snapshot storage swi-I-s---  10.00g  # I = Invalid

# Увеличиваем размер снимка
$ sudo lvextend -L +5G /dev/storage/home_snapshot

# Или удаляем, если не нужен
$ sudo lvremove /dev/storage/home_snapshot
```

**3. Восстановление после сбоя диска**

```shell
# Если один из PV недоступен
$ sudo vgchange -ay --partial storage
  Partial mode. Incomplete logical volumes will be processed.

# Удаляем сбойный PV (данные будут потеряны!)
$ sudo vgreduce --removemissing storage
```