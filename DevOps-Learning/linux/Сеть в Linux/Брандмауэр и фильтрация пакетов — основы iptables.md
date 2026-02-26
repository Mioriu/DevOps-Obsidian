**iptables** — это классический инструмент Linux для настройки сетевого экрана (брандмауэра). Он позволяет контролировать и фильтровать сетевой трафик на уровне IP-пакетов, обеспечивая защиту от несанкционированного доступа и сетевых атак.

### Как устроен iptables?

iptables работает на основе правил, определяющих, как обрабатывать сетевые пакеты. Правила организованы в **цепочки (chains)**, которые находятся в **таблицах (tables)**. Основные таблицы:

- `filter` — используется чаще всего (для фильтрации пакетов).
- `nat` — управление трансляцией сетевых адресов (NAT).
- `mangle` — для изменения (маркировки) пакетов.

Основные цепочки таблицы `filter`:

- **INPUT** — пакеты, входящие на сервер.
- **OUTPUT** — исходящие с сервера пакеты.
- **FORWARD** — пакеты, пересылаемые через сервер.

### Просмотр текущих правил iptables

Чтобы посмотреть текущие правила iptables, выполните:

```bash
sudo iptables -L -n -v
```

- `-L` — вывести текущие правила.
- `-n` — не разрешать IP-адреса в имена.
- `-v` — подробный вывод.

### Добавление правил iptables

Правила добавляются командой `iptables -A` (Append). Примеры:

**Разрешить SSH (порт 22):**

```css
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

**Разрешить веб-сервер (порт 80):**

```css
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

**Разрешить все исходящие соединения (по умолчанию рекомендуется):**

```bash
sudo iptables -A OUTPUT -j ACCEPT
```

### Удаление правил iptables

Удалить конкретное правило можно командой:

```bash
sudo iptables -D INPUT номер_правила
```

Номер правила можно увидеть командой:

```bash
sudo iptables -L --line-numbers
```

### Политики по умолчанию (default policies)

Для безопасности важно определить стандартное поведение цепочек. По умолчанию рекомендуется:

- Запретить все входящие соединения, кроме явно разрешённых.
- Разрешить все исходящие соединения.

**Настройка политик по умолчанию:**

```css
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

### Типичный пример базовой конфигурации iptables

Пример типичной базовой конфигурации для сервера:

```css
# Сброс текущих правил
sudo iptables -F
sudo iptables -X

# Политики по умолчанию
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Разрешаем loopback-интерфейс
sudo iptables -A INPUT -i lo -j ACCEPT

# Разрешаем уже установленные соединения
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# SSH (порт 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# HTTP (порт 80)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# HTTPS (порт 443)
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

### Сохранение правил iptables после перезагрузки

Правила, добавленные вручную, не сохраняются после перезагрузки. Для сохранения правил используется:

- **Debian/Ubuntu**:

```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

- **CentOS/RHEL**:

```bash
sudo yum install iptables-services
sudo service iptables save
sudo systemctl enable --now iptables
```

### Примеры полезных правил iptables

**Блокировка конкретного IP-адреса:**

```css
sudo iptables -A INPUT -s 203.0.113.45 -j DROP
```

**Разрешить ICMP (ping):**

```css
sudo iptables -A INPUT -p icmp -j ACCEPT
```

**Ограничение числа подключений для защиты от DDoS:**

```css
# Разрешать не более 10 новых подключений в минуту с одного IP
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m recent --set
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -m recent --update --seconds 60 --hitcount 10 -j DROP
```

### Отладка и мониторинг iptables

Просмотр статистики попаданий по правилам:

```bash
sudo iptables -L -n -v
```

Просмотр пакетов в реальном времени (полезно при отладке):

```css
sudo iptables -A INPUT -p tcp --dport 22 -j LOG --log-prefix "SSH connection: "
```

Логи будут в `/var/log/syslog` (Debian/Ubuntu) или `/var/log/messages` (CentOS).

## Итоги

- `iptables` — основной инструмент для настройки firewall в Linux.
- Правила организуются в цепочки (INPUT, OUTPUT, FORWARD) и таблицы (filter, nat, mangle).
- Стандартные политики по умолчанию важны для базовой защиты системы.
- Правила iptables требуют сохранения для применения после перезагрузки.
- Используйте логирование и статистику для диагностики и мониторинга.

Знание iptables позволяет надёжно защищать Linux-серверы и управлять сетевым трафиком на профессиональном уровне.