
По сути в данной задаче достаточно просто создать юзера `testuser` — у него уже будут права на чтение этого файла, т.к. по умолчанию права на `/opt` — 755, а на создаваемые в ней файлы — 644. То есть пользователь уже может читать этот файл, но не писать в него.
**Проверка:**
```bash
ls -la /opt/secret.txt && \
su - testuser -c "cat /opt/secret.txt" && \
su - testuser -c "echo 'hacked' >> /opt/secret.txt"

-rw-r--r-- 1 root root 12 Jun 18 22:12 /opt/secret.txt
secret data
-bash: line 1: /opt/secret.txt: Permission denied
```

**Но для проверки:**

```
groupadd secretreaders
usermod -aG secretreaders testuser
chown root:secretreaders /opt && chmod 640 /opt/secret.txt
```

Теперь у нас есть группа, которой доступно только чтение этого файла, и в неё входит `testuser`.