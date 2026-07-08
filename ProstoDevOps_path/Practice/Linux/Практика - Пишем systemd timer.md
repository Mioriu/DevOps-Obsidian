Сервис:
[Unit]
Description=simple cleanup Service

[Services]
Type=oneshot
ExecStart=/bin/sh -c /opt/scripts/cleanup.sh
ReadWritePath=/var/log

Таймер:
[Unit]
Description=cleanup service timer
After=time-sync.target systemd-time-wait-sync.target

[Timer]
OnCalendar=*:0/2
Persistent=true
Unit=cleanup.service

[Install]
WantedBy=timers.target

Проверка:
systemctl list-timers | grep cleanup && cat /var/log/cleanup.log
Wed 2026-07-08 22:38:00 MSK 1min 14s Wed 2026-07-08 22:36:26 MSK      18s ago cleanup.timer                  cleanup.service
[2026-07-08 22:34:11] Cleanup running
[2026-07-08 22:36:26] Cleanup running
