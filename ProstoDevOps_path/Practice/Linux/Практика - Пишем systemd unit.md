Сам юнит файл

cat /etc/systemd/system/simple-web
[Unit]
Description=test python server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/simple-web/server.py
Restart=on-failure
RestartSec=5s
User=nobody
Group=nobody

[Install]
WantedBy=multi-user.target

Проверка:
systemctl daemon-reload
systemctl enable --now simple-web
sytemctl status simple-web
simple-web.service - test python server
     Loaded: loaded (/etc/systemd/system/simple-web.service; enabled; preset: enabled)
     Active: active (running) since Wed 2026-07-08 22:04:05 MSK; 3s ago
   Main PID: 16481 (python3)
      Tasks: 1 (limit: 19082)
     Memory: 8.9M (peak: 9.4M)
        CPU: 100ms
     CGroup: /system.slice/simple-web.service
             └─16481 /usr/bin/python3 /opt/simple-web/server.py

curl http://localhost:8080
Hello from systemd service!
