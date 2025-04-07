#!/bin/bash

# ================================
# Enhanced BlackArch System Optimizer
# Optimized for: 4.5GB RAM, 40GB Storage
# ================================

# Configuration
MONITOR_INTERVAL=60          # Reduced for faster response to resource pressure
LOG_FILE="/var/log/sys_monitor.log"
CLEANUP_INTERVAL=300        # System cleanup every 5 minutes

# Optimized resource thresholds
CPU_MAX=75                  # Reduced to prevent system freeze
MEM_MAX=80                  # Adjusted for ZRAM usage
DISK_MAX=85                 # Storage monitoring threshold

# Timestamp function
timestamp() {
    date '+%Y-%m-%d %H:%M:%S'
}

# Enhanced system cleanup function
cleanup_system() {
    sync; echo 1 > /proc/sys/vm/drop_caches
    swapoff -a && swapon -a
    journalctl --vacuum-size=50M
    echo "$(timestamp) - System cleanup performed" >> "$LOG_FILE"
}

# ZRAM setup function (40% of RAM = ~1.8GB)
setup_zram() {
    modprobe zram
    echo lz4 > /sys/block/zram0/comp_algorithm
    echo 1843M > /sys/block/zram0/disksize
    mkswap /dev/zram0
    swapon -p 100 /dev/zram0
}

# Temporary filesystem optimization
optimize_tmp() {
    mount -o remount,noexec,nosuid,size=1152M /tmp
    mkdir -p /tmp/blackarch_scans
    mkdir -p /tmp/blackarch_enum
    chmod 1777 /tmp/blackarch_*
}

# Main system optimization function
optimize_system() {
    sysctl -w vm.swappiness=100
    sysctl -w vm.vfs_cache_pressure=500
    sysctl -w vm.dirty_background_ratio=1
    sysctl -w vm.dirty_ratio=50
    sysctl -w vm.dirty_expire_centisecs=1000
    sysctl -w vm.dirty_writeback_centisecs=500
    sysctl -w vm.min_free_kbytes=943104

    for disk in /sys/block/sd*/queue/scheduler; do
        echo "deadline" > "$disk"
    done

    for disk in /sys/block/sd*/queue/read_ahead_kb; do
        echo "128" > "$disk"
    done

    setup_zram
    optimize_tmp

    for tool in nmap masscan gobuster nikto dirb; do
        if pgrep -f "$tool" > /dev/null; then
            renice -10 -p $(pgrep -f "$tool")
        fi
    done

    for service in cups bluetooth avahi-daemon; do
        systemctl stop "$service"
        systemctl disable "$service"
    done

    echo "$(timestamp) - System optimization completed" >> "$LOG_FILE"
}

# Enhanced system monitoring
monitor_system() {
    local cleanup_counter=0
    while true; do
        CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2 + $4}')
        MEM_USAGE=$(free | awk '/Mem:/ {print $3/$2 * 100}')
        SWAP_USAGE=$(free | awk '/Swap:/ {print $3/$2 * 100}')
        DISK_USAGE=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')

        echo "$(timestamp) - CPU: ${CPU_USAGE}% MEM: ${MEM_USAGE}% SWAP: ${SWAP_USAGE}% DISK: ${DISK_USAGE}%" >> "$LOG_FILE"

        if [ "${CPU_USAGE%.*}" -gt "$CPU_MAX" ] || [ "${MEM_USAGE%.*}" -gt "$MEM_MAX" ]; then
            cleanup_system
        fi

        cleanup_counter=$((cleanup_counter + 1))
        if [ $cleanup_counter -ge $(($CLEANUP_INTERVAL / $MONITOR_INTERVAL)) ]; then
            cleanup_system
            cleanup_counter=0
        fi

        sleep $MONITOR_INTERVAL
    done
}

# Main execution
echo "$(timestamp) - Starting BlackArch System Optimizer" >> "$LOG_FILE"
optimize_system
monitor_system
