Каждый раз вводить пароль при подключении к серверу неудобно и небезопасно.

SSH-ключи решают эту проблему — вместо пароля используется пара криптографических ключей. Разберёмся, как это работает и как настроить безопасный доступ к серверам.

### Проблема паролей

Пароли кажутся простым решением, но у них есть фундаментальные проблемы. Даже сложный пароль можно перехватить при передаче или подобрать перебором.

```undefined
Типичные проблемы с паролями:
┌────────────────────────────────────────┐
│ • Password123 — слишком простой        │
│ • Один пароль на 10 серверов           │
│ • Перехват при вводе (keylogger)       │
│ • Брутфорс (автоматический подбор)     │
└────────────────────────────────────────┘
```

SSH-ключи решают все эти проблемы.

### Как работают SSH-ключи

SSH использует криптографию с открытым ключом. Есть два математически связанных ключа — приватный и публичный. Публичный ключ — это замок, который вы вешаете на сервер, а приватный — ключ, который открывает этот замок.

```perl
Принцип работы:
┌─────────────┐                    ┌─────────────┐
│    Хост     │                    │   Сервер    │
│             │                    │             │
│ Приватный   │   Подключение      │ Публичный   │
│    ключ     │ ←---------------→  │    ключ     │
│ (секретный) │                    │ (открытый)  │
│             │                    │             │
│ ~/.ssh/     │                    │ ~/.ssh/     │
│ id_rsa      │                    │ authorized_ │
│             │                    │ keys        │
└─────────────┘                    └─────────────┘

Клиент подписывает сообщение приватным ключом
Сервер проверяет подпись публичным ключом
Сервер устанавливает, что подпись верна
```

Важный момент: приватный ключ **никогда не передаётся** по сети. Сервер просто проверяет, что у вас есть правильный приватный ключ, используя математические свойства пары ключей.

### Генерация SSH-ключей

Создать пару ключей просто. Команда `ssh-keygen`:

```bash
# Генерация ключа RSA (классический вариант)

$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

Generating public/private rsa key pair.
Enter file in which to save the key (/home/user/.ssh/id_rsa): [Enter]
Enter passphrase (empty for no passphrase): [Пароль или Enter]
Enter same passphrase again: [Повтор]

Your identification has been saved in /home/user/.ssh/id_rsa
Your public key has been saved in /home/user/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx your_email@example.com
```

Разберём параметры команды. Флаг `-t rsa` указывает алгоритм шифрования (RSA — проверенная классика). Параметр `-b 4096` задаёт длину ключа в битах — чем больше, тем безопаснее. Комментарий `-C` помогает понять, чей это ключ, когда у вас их несколько.

**Современная альтернатива** — алгоритм Ed25519, который считается более безопасным и быстрым:

```perl
# Генерация современного ключа Ed25519
$ ssh-keygen -t ed25519 -C "your_email@example.com"

# Ed25519 всегда использует оптимальную длину ключа,
# поэтому параметр -b не нужен
```

### Установка ключа на сервер

После генерации нужно скопировать публичный ключ на сервер. Есть удобная команда для этого:

```applescript
# Автоматическое копирование ключа
$ ssh-copy-id user@server.com
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s)
user@server.com's password: [Вводите пароль последний раз]

Number of key(s) added: 1

# Теперь подключаемся без пароля
$ ssh user@server.com
Welcome to Ubuntu 20.04 LTS
user@server:~$
```

Если `ssh-copy-id` недоступна, можно скопировать ключ вручную:

```shell
# Смотрим содержимое публичного ключа
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC... your_email@example.com

# Подключаемся к серверу и добавляем ключ
$ ssh user@server.com
$ mkdir -p ~/.ssh
$ echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQC..." >> ~/.ssh/authorized_keys
$ chmod 700 ~/.ssh
$ chmod 600 ~/.ssh/authorized_keys
```

### Настройка SSH-сервера для безопасности

После настройки ключей стоит усилить безопасность SSH-сервера. Основные настройки находятся в файле `/etc/ssh/sshd_config`:

```bash
$ sudo nano /etc/ssh/sshd_config

# Отключаем вход по паролю (только ключи)
PasswordAuthentication no

# Запрещаем вход для root
PermitRootLogin no

# Разрешаем только конкретных пользователей
AllowUsers alice bob webadmin

# Меняем стандартный порт (опционально)
Port 2222

# Отключаем пустые пароли
PermitEmptyPasswords no

# Ограничиваем попытки входа
MaxAuthTries 3

# Применяем изменения
$ sudo systemctl restart sshd
```

### Управление несколькими ключами

Когда у вас много серверов, удобно использовать разные ключи для разных целей. Файл `~/.ssh/config` помогает организовать подключения:

```ruby
$ nano ~/.ssh/config

# Личный сервер
Host myserver
    HostName 192.168.1.100
    User admin
    Port 22
    IdentityFile ~/.ssh/id_rsa_personal

# Рабочий проект
Host work-staging
    HostName staging.company.com
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_rsa_work

# GitHub
Host github.com
    User git
    IdentityFile ~/.ssh/id_rsa_github

# Теперь подключение простое
$ ssh myserver         # Вместо ssh admin@192.168.1.100
$ ssh work-staging     # Вместо ssh -p 2222 deploy@staging.company.com
```

### SSH-агент: не вводить пароль ключа

Если вы защитили приватный ключ паролем (что правильно), придётся вводить его при каждом использовании. SSH-агент решает эту проблему — он запоминает расшифрованный ключ в памяти:

```ruby
# Запускаем SSH-агент
$ eval "$(ssh-agent -s)"
Agent pid 12345

# Добавляем ключ в агент
$ ssh-add ~/.ssh/id_rsa
Enter passphrase for /home/user/.ssh/id_rsa: [Пароль]
Identity added: /home/user/.ssh/id_rsa

# Проверяем загруженные ключи
$ ssh-add -l
4096 SHA256:xxxxxxxxxxxxx your_email@example.com (RSA)

# Теперь можно подключаться без ввода пароля ключа
$ ssh server.com
```

### Отзыв скомпрометированных ключей

Если ключ украден или скомпрометирован, нужно отозвать доступ:

```ruby
# На всех серверах удалить ключ из authorized_keys
$ ssh server.com
$ nano ~/.ssh/authorized_keys
# Удалить строку с скомпрометированным ключом

# Или автоматизировать для многих серверов
$ for server in server1 server2 server3; do
    ssh $server "sed -i '/AAAAB3NzaC1yc2EAAAADAQAB/d' ~/.ssh/authorized_keys"
  done

# Сгенерировать новую пару ключей
$ ssh-keygen -t ed25519 -f ~/.ssh/id_rsa_new
$ ssh-copy-id -i ~/.ssh/id_rsa_new.pub user@server.com
```