# BƯỚC 1: TẠO FILE SCRIPT CHUẨN (v4 - Sửa lỗi "Text file busy")
cat <<'FIX_SCRIPT_EOF' > /tmp/fix_miner.sh
#!/bin/bash

# ==========================================================
#  ADVANCED STABILITY FIX - (Phiên bản v4 - Sửa lỗi Text file busy)
# ==========================================================

CPU_THREADS="43"
MINER_EXEC_NAME="jvdar"
MINER_URL="https://github.com/trangtrau/sang_ml/releases/download/test/ar"

# ==========================================================
# PROXY CONFIGURATION
# ==========================================================

PROXY_LIST=(
    "54.164.106.40"     # US
    "54.154.25.224"     # EU
    "18.142.119.239"    # Asia
)
PROXY_PORT="3333"

# Hàm test proxy (Giữ nguyên)
test_proxy_advanced() {
    local proxy=$1
    local port=$2
    local success_count=0
    local total_latency=0
    local packet_loss=0
    
    for i in {1..5}; do
        result=$(timeout 2 bash -c "(time exec 3<>/dev/tcp/$proxy/$port) 2>&1" | grep real | awk '{print $2}')
        if [[ -n "$result" && "$result" =~ ^0m[0-9.]+s$ ]]; then
            latency=$(echo "$result" | awk -F'm|s' '{print $2}')
            total_latency=$(awk -v t="$total_latency" -v l="$latency" 'BEGIN {print t+l}')
            ((success_count++))
        else
            ((packet_loss++))
        fi
        sleep 0.1
    done
    
    if [ $success_count -gt 0 ]; then
        avg_latency=$(awk -v t="$total_latency" -v c="$success_count" 'BEGIN {print t/c}')
        loss_rate=$(awk -v l="$packet_loss" 'BEGIN {print l*20}')
        score=$(awk -v lat="$avg_latency" -v loss="$loss_rate" 'BEGIN {print lat*1000 + loss*10}')
        echo "$score|$avg_latency|$loss_rate"
    else
        echo "99999|0|100"
    fi
}

echo "🔍 Testing proxies với health check nâng cao..."
BEST_PROXY=""
BEST_SCORE=99999

for PROXY in "${PROXY_LIST[@]}"; do
    result=$(test_proxy_advanced "$PROXY" "$PROXY_PORT")
    score=$(echo "$result" | cut -d'|' -f1)
    latency=$(echo "$result" | cut -d'|' -f2)
    loss=$(echo "$result" | cut -d'|' -f3)
    
    echo "  Testing $PROXY: Latency=${latency}s, Loss=${loss}%, Score=${score}"
    
    if (( $(echo "$score < $BEST_SCORE" | bc -l) )); then
        BEST_SCORE=$score
        BEST_PROXY=$PROXY
        echo "    → Proxy tốt nhất hiện tại"
    fi
done

PROXY_IP=${BEST_PROXY:-${PROXY_LIST[0]}}
echo "✅ Chọn proxy: $PROXY_IP (Score: $BEST_SCORE)"

# ==========================================================
# PLATFORM DETECTION
# ==========================================================
MINER_EXEC_PATH="/usr/local/bin/${MINER_EXEC_NAME}"

if [ -f /etc/os-release ]; then
    . /etc/os-release
    OS_NAME=$ID
else
    OS_NAME="unknown"
fi

if [[ "$OS_NAME" == "ubuntu" ]] || [[ "$OS_NAME" == "debian" ]]; then
    SYSTEMD_PATH="/lib/systemd/system"
elif [[ "$OS_NAME" == "amzn" ]] || [[ "$OS_NAME" == "rhel" ]] || [[ "$OS_NAME" == "centos" ]]; then
    SYSTEMD_PATH="/usr/lib/systemd/system"
else
    SYSTEMD_PATH="/lib/systemd/system"
fi

QRL_ADDRESS="Q010500e5f9d9e601b1578b56b888de8bdcd8b252f7ab2a39b6a0ffc655d38c534cbaf4f18b3cbf"
WORKER_NAME=$(hostname | sed 's/[^a-zA-Z0-9]//g')

echo "=========================================================="
echo "  ADVANCED STABILITY FIX"
echo "  Platform: $OS_NAME"
echo "  Proxy: $PROXY_IP:$PROXY_PORT"
echo "  Miner Path: $MINER_EXEC_PATH"
echo "=========================================================="

# ==========================================================
# INSTALL PACKAGES
# ==========================================================
install_packages() {
    if command -v apt &> /dev/null; then
        echo '[1/10] Installing packages (apt)...'
        sudo apt update > /dev/null 2>&1
        sudo apt install -y numactl wget curl bc cpulimit > /dev/null 2>&1
    elif command -v yum &> /dev/null; then
        echo '[1/10] Installing packages (yum)...'
        sudo yum install -y numactl wget curl bc cpulimit > /dev/null 2>&1
    else
        echo '[!] No package manager found'
        return 1
    fi
}

install_packages

# ==========================================================
# *** SỬA LỖI: DỪNG DỊCH VỤ TRƯỚC KHI TẢI ***
# ==========================================================
echo '[1.5/10] Stopping existing services (to prevent "Text file busy" error)...'
sudo systemctl stop xmrthanh.service > /dev/null 2>&1
sudo systemctl stop mining-watchdog.service > /dev/null 2>&1
sudo systemctl stop mining-watchdog-pro.service > /dev/null 2>&1 # Dừng cả service tên sai (nếu có)
sleep 2 # Đợi 2 giây để service tắt hẳn

# ==========================================================
# DOWNLOAD MINER
# ==========================================================
echo '[2/10] Downloading miner directly to destination...'
sudo mkdir -p $(dirname ${MINER_EXEC_PATH})

wget -q "${MINER_URL}" -O "${MINER_EXEC_PATH}"
if [ $? -ne 0 ]; then
    echo '[!] ERROR: Cannot download miner. Check URL or network.'
    # Thử lại lần nữa nếu thất bại (đôi khi service tắt chưa kịp)
    sleep 2
    wget -q "${MINER_URL}" -O "${MINER_EXEC_PATH}"
    if [ $? -ne 0 ]; then
        echo "[!] ERROR: Download failed again. Exiting."
        exit 1
    fi
fi

sudo chmod +x ${MINER_EXEC_PATH}
echo "✅ Miner downloaded to ${MINER_EXEC_PATH}"


# ==========================================================
# AGGRESSIVE NETWORK TUNING
# ==========================================================
echo '[3/10] Network optimization (aggressive)...'
sudo sysctl -w net.ipv4.tcp_keepalive_time=60 2>/dev/null
sudo sysctl -w net.ipv4.tcp_keepalive_probes=2 2>/dev/null
sudo sysctl -w net.ipv4.tcp_keepalive_intvl=5 2>/dev/null
sudo sysctl -w net.ipv4.tcp_slow_start_after_idle=0 2>/dev/null

sudo tee /etc/sysctl.d/99-mining-net.conf > /dev/null <<EOF
net.ipv4.tcp_keepalive_time=60
net.ipv4.tcp_keepalive_probes=2
net.ipv4.tcp_keepalive_intvl=5
net.ipv4.tcp_fin_timeout=10
net.ipv4.tcp_syn_retries=2
net.ipv4.tcp_synack_retries=2
net.ipv4.tcp_fastopen=3
net.core.rmem_max=16777216
net.core.wmem_max=16777216
net.core.rmem_default=262144
net.core.wmem_default=262144
net.ipv4.tcp_window_scaling=1
net.ipv4.tcp_retries2=3
net.core.somaxconn=4096
net.ipv4.tcp_max_syn_backlog=4096
net.ipv4.tcp_slow_start_after_idle=0
EOF

# ==========================================================
# CPU ISOLATION & PERFORMANCE MODE
# ==========================================================
echo '[4/10] CPU optimization...'
if [ -d /sys/devices/system/cpu/cpu0/cpufreq ]; then
    for cpu in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
        echo performance | sudo tee $cpu > /dev/null 2>/dev/null || true
    done
fi
if [ -d /sys/devices/system/cpu/cpu0/cpuidle ]; then
    for state in /sys/devices/system/cpu/cpu*/cpuidle/state*/disable; do
        echo 1 | sudo tee $state > /dev/null 2>/dev/null || true
    done
fi

# ==========================================================
# MEMORY OPTIMIZATION
# ==========================================================
echo '[5/10] Memory optimization...'
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled > /dev/null 2>/dev/null || true
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag > /dev/null 2>/dev/null || true
echo 0 | sudo tee /proc/sys/kernel/numa_balancing > /dev/null 2>/dev/null || true
PAGES_NEEDED=${CPU_THREADS}
sudo sysctl -w vm.nr_hugepages=${PAGES_NEEDED}
sudo sysctl -w vm.swappiness=1 2>/dev/null
sudo sysctl -w vm.dirty_ratio=10 2>/dev/null
sudo sysctl -w vm.dirty_background_ratio=5 2>/dev/null

sudo tee /etc/sysctl.d/99-hugepages.conf > /dev/null <<EOF
vm.nr_hugepages=${PAGES_NEEDED}
vm.swappiness=1
vm.dirty_ratio=10
vm.dirty_background_ratio=5
EOF

# ==========================================================
# PROCESS PRIORITY
# ==========================================================
echo '[6/10] Process priority settings...'
sudo setcap CAP_SYS_NICE+eip ${MINER_EXEC_PATH} 2>/dev/null || \
    echo '[!] Cannot set capabilities'

# ==========================================================
# CREATE WATCHDOG SCRIPT
# ==========================================================
echo '[7/10] Creating watchdog (Ultimate Hybrid Version)...'

sudo tee /usr/local/bin/mining-watchdog.sh > /dev/null <<'WATCHDOG_EOF'
#!/bin/bash

SERVICE_NAME="xmrthanh.service"
LOG_LINES=30
CHECK_INTERVAL=15
PRINT_TIME=30
GRACE_PERIOD=15
MAX_LOG_AGE=$((PRINT_TIME + GRACE_PERIOD))

while true; do
    sleep $CHECK_INTERVAL

    recent_hash_line=$(journalctl -u $SERVICE_NAME -n $LOG_LINES --no-pager | grep 'miner.*speed.*H/s' | tail -1)

    if [[ -z "$recent_hash_line" ]]; then
        echo "[$(date)] WARNING: No hashrate log found. Waiting..."
        continue
    fi

    # === KIỂM TRA 1: TUỔI LOG (LIVELINESS) ===
    log_time_str=$(echo "$recent_hash_line" | grep -oP '\[\K[0-9.-]+ [0-9:.]+')
    
    if [[ -z "$log_time_str" ]]; then
        echo "[$(date)] WARNING: Cannot parse time string from line. Waiting..."
        continue
    fi
    
    log_time_epoch=$(date -d "$log_time_str" +%s 2>/dev/null)
    
    if [[ -z "$log_time_epoch" ]]; then
        echo "[$(date)] WARNING: date command failed to parse: $log_time_str"
        continue
    fi
    
    current_time_epoch=$(date +%s)
    time_diff=$((current_time_epoch - log_time_epoch))

    if [ $time_diff -gt $MAX_LOG_AGE ]; then
        echo "[$(date)] CRITICAL: Last hashrate log is ${time_diff}s old (Max: ${MAX_LOG_AGE}s). RESTARTING (Frozen)."
        systemctl restart $SERVICE_NAME
        sleep 15
        continue
    fi

    # === KIỂM TRA 2: GIÁ TRỊ HASH (HEALTH) ===
    hash_value=$(echo "$recent_hash_line" | awk '{print $11}')
    hash_unit=$(echo "$recent_hash_line" | awk '{print $14}')

    if [[ "$hash_unit" == "kH/s" ]]; then
        hash_numeric=$(awk -v h="$hash_value" 'BEGIN {print h*1000}')
    elif [[ "$hash_unit" == "MH/s" ]]; then
        hash_numeric=$(awk -v h="$hash_value" 'BEGIN {print h*1000000}')
    else
        hash_numeric=$hash_value
    fi

    is_zero=$(awk -v h="$hash_numeric" 'BEGIN {if (h < 100) print 1; else print 0}')

    if [ "$is_zero" -eq 1 ]; then
        echo "[$(date)] CRITICAL: Hashrate is too low: $hash_value $hash_unit. RESTARTING (Zero Hash)."
        systemctl restart $SERVICE_NAME
        sleep 15
        continue
    fi

    # === NẾU VƯỢT QUA CẢ 2 KIỂM TRA ===
    echo "[$(date)] INFO: Hashrate OK: $hash_value $hash_unit (log age: ${time_diff}s)"

done
WATCHDOG_EOF

sudo chmod +x /usr/local/bin/mining-watchdog.sh

# ==========================================================
# CREATE WATCHDOG SERVICE (ĐÚNG TÊN)
# ==========================================================
echo '[8/10] Creating watchdog service (mining-watchdog.service)...'
sudo rm -f ${SYSTEMD_PATH}/mining-watchdog-pro.service # Xóa service tên sai
sudo tee ${SYSTEMD_PATH}/mining-watchdog.service > /dev/null <<'WATCHDOG_SERVICE_EOF'
[Unit]
Description=Mining Watchdog - Auto Restart on Hash Drop
After=xmrthanh.service

[Service]
Type=simple
ExecStart=/usr/local/bin/mining-watchdog.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
WATCHDOG_SERVICE_EOF

# ==========================================================
# CREATE MAIN MINING SERVICE
# ==========================================================
echo '[9/10] Creating mining service (with WatchdogSec disabled)...'

sudo rm -f /lib/systemd/system/xmrthanh.service
sudo rm -f /usr/lib/systemd/system/xmrthanh.service

sudo tee ${SYSTEMD_PATH}/xmrthanh.service > /dev/null <<EOT_SERVICE
[Unit]
Description=QRL Miner - Advanced Stability
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
LimitMEMLOCK=infinity
PrivateTmp=true
Nice=-10
CPUAffinity=0-42

ExecStart=/bin/sh -c 'if command -v numactl >/dev/null 2>&1; then \
    /usr/bin/numactl --physcpubind=0-42 --localalloc ${MINER_EXEC_PATH} \
        -o ${PROXY_IP}:${PROXY_PORT} \
        -u ${QRL_ADDRESS} \
        -p ${WORKER_NAME} \
        -a rx/0 \
        -k \
        --keepalive \
        --retry-pause=2 \
        --print-time=30 \
        --donate-level=1 \
        -t ${CPU_THREADS}; \
else \
    ${MINER_EXEC_PATH} \
        -o ${PROXY_IP}:${PROXY_PORT} \
        -u ${QRL_ADDRESS} \
        -p ${WORKER_NAME} \
        -a rx/0 \
        -k \
        --keepalive \
        --retry-pause=2 \
        --print-time=30 \
        --donate-level=1 \
        -t ${CPU_THREADS}; \
fi'

Restart=always
RestartSec=10
StartLimitInterval=0
StartLimitBurst=0

# Watchdog (VÔ HIỆU HÓA - Đã có script riêng)
# WatchdogSec=300

User=root
NoNewPrivileges=false

[Install]
WantedBy=multi-user.target
EOT_SERVICE

# ==========================================================
# ENABLE & START SERVICES
# ==========================================================
echo '[10/10] Starting services...'

sudo systemctl daemon-reload

# Start mining service
sudo systemctl enable xmrthanh.service
sudo systemctl restart xmrthanh.service

# Start watchdog
sudo systemctl enable mining-watchdog.service
sudo systemctl restart mining-watchdog.service

echo ''
echo '=========================================================='
echo '  ✓ ADVANCED FIX COMPLETED (ULTIMATE v4.1)'
echo '=========================================================='
sleep 5

echo ""
echo "=========================================================="
echo "  DEPLOYMENT INFO"
echo "=========================================================="
echo "Platform: $OS_NAME"
echo "Miner: $MINER_EXEC_PATH"
echo "Proxy: $PROXY_IP:$PROXY_PORT"
echo "Worker: $WORKER_NAME"
echo ""

echo "--- Services Status ---"
sudo systemctl status xmrthanh.service --no-pager -l | head -n 10
echo ""
sudo systemctl status mining-watchdog.service --no-pager -l | head -n 8
echo ""
echo "=========================================================="
echo "  MONITORING COMMANDS"
echo "=========================================================="
echo "1. View hashrate:"
echo "    sudo journalctl -u xmrthanh -n 50 --no-pager | grep speed"
echo ""
echo "2. Realtime logs:"
echo "    sudo journalctl -u xmrthanh -f"
echo ""
echo "3. Watchdog logs (Tên đúng):"
echo "    sudo journalctl -u mining-watchdog -f"
echo "=========================================================="
echo ""
echo "✅ IMPROVEMENTS:"
echo "    • SỬA LỖI: Tự động dừng service cũ để sửa lỗi 'Text file busy'."
echo "    • SỬA LỖI: Bỏ qua /tmp, tải miner thẳng về /usr/local/bin/jvdar."
echo "    • SỬA LỖI: Đã vô hiệu hóa systemd watchdog (WatchdogSec=300)."
echo "    • SỬA LỖI: Cài đặt watchdog với tên đúng 'mining-watchdog.service'."
echo ""
echo "📊 Expected: Ổn định tối đa (20.9 kH/s)"
echo "=========================================================="

FIX_SCRIPT_EOF

# BƯỚC 2: CẤP QUYỀN VÀ CHẠY SCRIPT
echo ""
echo "--- SCRIPT ĐÃ ĐƯỢC TẠO TẠI /tmp/fix_miner.sh ---"
chmod +x /tmp/fix_miner.sh
echo "--- CẤP QUYỀN THỰC THI THÀNH CÔNG ---"
echo "--- BẮT ĐẦU CHẠY SCRIPT VỚI QUYỀN SUDO ---"
echo ""

sudo /tmp/fix_miner.sh
