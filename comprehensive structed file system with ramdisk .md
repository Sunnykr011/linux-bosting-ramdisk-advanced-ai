#!/bin/bash
set -euo pipefail

### CONFIG ###
RAMDISK_DIR="/mnt/ramdisk"
SCRIPT_DIR="/opt/ramdisk-script"
SCRIPT_NAME="myscript.sh"
RAM_SIZE="256M"
SERVICE_NAME="ramdisk-runner"

echo "[*] Starting optimized setup..."

# ✅ 1. Create persistent storage for the script
echo "[*] Creating script directory..."
sudo mkdir -p "$SCRIPT_DIR"

# ✅ 2. Create the user script to run from RAM
cat <<'EOL' | sudo tee "$SCRIPT_DIR/$SCRIPT_NAME" > /dev/null
#!/bin/bash
set -euo pipefail
echo "[+] Running from RAM disk at $(date)"
# Add your custom logic here
dd if=/dev/zero of=/mnt/ramdisk/speedtest bs=1M count=100 conv=fdatasync
echo "[+] Done with high-speed operation."
EOL

sudo chmod +x "$SCRIPT_DIR/$SCRIPT_NAME"

# ✅ 3. Create launcher that sets up RAM disk and runs the script
cat <<EOF | sudo tee "$SCRIPT_DIR/ramdisk-launcher.sh" > /dev/null
#!/bin/bash
set -euo pipefail

RAMDISK_DIR="$RAMDISK_DIR"
SCRIPT_NAME="$SCRIPT_NAME"

echo "[*] Preparing RAM disk..."
mkdir -p "\$RAMDISK_DIR"

# 🧼 Clean up old contents
echo "[*] Cleaning RAM disk..."
rm -rf "\$RAMDISK_DIR"/*

# ✅ Mount with performance flags
if ! mountpoint -q "\$RAMDISK_DIR"; then
    mount -t tmpfs -o size=$RAM_SIZE,noatime,nodiratime tmpfs "\$RAMDISK_DIR"
fi

echo "[*] Copying script to RAM disk..."
cp "$SCRIPT_DIR/\$SCRIPT_NAME" "\$RAMDISK_DIR/\$SCRIPT_NAME"
chmod +x "\$RAMDISK_DIR/\$SCRIPT_NAME"

echo "[*] Running script from RAM..."
"\$RAMDISK_DIR/\$SCRIPT_NAME"
EOF

sudo chmod +x "$SCRIPT_DIR/ramdisk-launcher.sh"

# ✅ 4. Create systemd service to run at boot
echo "[*] Creating systemd service..."
cat <<EOF | sudo tee "/etc/systemd/system/${SERVICE_NAME}.service" > /dev/null
[Unit]
Description=Run script from RAM disk at boot
After=network-online.target local-fs.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=$SCRIPT_DIR/ramdisk-launcher.sh
RemainAfterExit=false

[Install]
WantedBy=multi-user.target
EOF

# ✅ 5. Enable the service
echo "[*] Enabling service..."
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable "${SERVICE_NAME}.service"

echo "[✔] Setup complete. Reboot to test!"
