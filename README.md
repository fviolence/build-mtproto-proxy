# Build MTProto proxy for Telegram (Debian 13, ufw)

### Install essentials
```bash
sudo apt update && sudo apt install -y git curl build-essential libssl-dev zlib1g-dev xxd vim ufw
```

### Configure firewall, choose <EXTERNAL_PORT> to be exposed (eg. 8443)
```bash
sudo ufw allow <EXTERNAL_PORT>/tcp
sudo ufw status numbered
sudo ufw enable 
sudo ufw reload
```

### Build MTProto
```bash
sudo git clone https://github.com/TelegramMessenger/MTProxy /opt/MTProxy
cd /opt/MTProxy
sudo make
cd /opt/MTProxy/objs/bin
sudo curl -fsS https://core.telegram.org/getProxySecret -o proxy-secret
sudo curl -fsS https://core.telegram.org/getProxyConfig -o proxy-multi.conf
```

### Create a mtproto and refresh services
First the script to update Telegram's secret and config
```bash
sudo vim /usr/local/sbin/mtproxy-refresh
```
```bash
#!/bin/bash

sudo curl -fsS https://core.telegram.org/getProxySecret -o /opt/MTProxy/objs/bin/proxy-secret
sudo curl -fsS https://core.telegram.org/getProxyConfig -o /opt/MTProxy/objs/bin/proxy-multi.conf
sudo systemctl daemon-reload
sudo systemctl restart mtproxy.service
```

Then refresh service, that will call the script, and a timer
```bash
sudo vim /etc/systemd/system/mtproxy-refresh.service
```
```conf
[Unit]
Description=Refresh MTProxy config

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/mtproxy-refresh
```

```bash
sudo vim /etc/systemd/system/mtproxy-refresh.timer
```
```conf
[Unit]
Description=Daily refresh of MTProxy config

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

Finally the proxy service itself
```bash
sudo vim /etc/systemd/system/mtproxy.service
```
```conf
[Unit]
Description=Telegram MTProxy
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/MTProxy/objs/bin
ExecStart=/opt/MTProxy/objs/bin/mtproto-proxy -u nobody -p 8888 -H <EXTERNAL_PORT> -S <CLIENT1_PASSWORD> --aes-pwd /opt/MTProxy/objs/bin/proxy-secret /opt/MTProxy/objs/bin/proxy-multi.conf -M 1
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Replace `<CLIENT1_PASSWORD>` with some random password, eg.:
```bash
head -c 16 /dev/urandom | xxd -p -c 16
```

> NOTE: To add more clients simply generate more passwords and pass `-S` option several times.

### Run daemons and final check
```bash
sudo chmod +x /usr/local/sbin/mtproxy-refresh

sudo systemctl daemon-reload

sudo systemctl enable --now mtproxy.service 
sudo systemctl enable --now mtproxy-refresh.timer
sudo systemctl start        mtproxy-refresh.service   # run once manually

sudo systemctl status mtproxy.service 
sudo systemctl status mtproxy-refresh.timer
sudo systemctl status mtproxy-refresh.service
```

# Resulting client link
```
https://t.me/proxy?server=<EXTERNAL_IP>&port=<EXTERNAL_PORT>&secret=<CLIENT1_PASSWORD>
```

### WA: avoid PID greater then 65535 (known issue with proxy server)

```bash
sudo sysctl -w kernel.pid_max=65535
cat /proc/sys/kernel/pid_max
printf 'kernel.pid_max = 65535\n' | sudo tee /etc/sysctl.d/99-mtproxy-pid-max.conf
sudo sysctl --system
```

Then check:
```bash
cat /proc/sys/kernel/pid_max
echo $$
```

And restart the server:
```bash
sudo systemctl restart mtproxy.service
sudo systemctl status mtproxy.service --no-pager
```
