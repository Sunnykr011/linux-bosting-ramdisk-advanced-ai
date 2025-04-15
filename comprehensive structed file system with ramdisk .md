#!/bin/bash
set -euo pipefail

### CONFIG ###
RAMDISK_DIR="/mnt/ramdisk"
SCRIPT_DIR="/opt/ramdisk-script"
SCRIPT_NAME="myscript.sh"
RAM_SIZE="256M"  # Safe for 4GB RAM
SERVICE_NAME="ramdisk-runner"

echo "[*] Starting one-time setup..."

# ✅ 1. Create persistent storage for the script
echo "[*] Creating persistent script location..."
sudo mkdir -p "$SCRIPT_DIR"

# ✅ 2. Create the user script to be run from RAM
cat <<'EOL' | sudo tee "$SCRIPT_DIR/$SCRIPT_NAME" > /dev/null
#!/bin/bash
set -euo pipefail
echo "[+] Running from RAM disk at $(date)"
# Add your logic below (example)
echo "[+] Performing high-speed operation..."
sleep 2
echo "[+] Done. This was fast and clean."
EOL

sudo chmod +x "$SCRIPT_DIR/$SCRIPT_NAME"

# ✅ 3. Create launcher that sets up RAM disk and runs your script
cat <<EOF | sudo tee "$SCRIPT_DIR/ramdisk-launcher.sh" > /dev/null
#!/bin/bash
set -euo pipefail

RAMDISK_DIR="$RAMDISK_DIR"
SCRIPT_NAME="$SCRIPT_NAME"

echo "[*] Mounting RAM disk at \$RAMDISK_DIR..."
mkdir -p "\$RAMDISK_DIR"
if ! mountpoint -q "\$RAMDISK_DIR"; then
    mount -t tmpfs -o size=$RAM_SIZE tmpfs "\$RAMDISK_DIR"
fi

echo "[*] Copying script to RAM disk..."
cp "$SCRIPT_DIR/\$SCRIPT_NAME" "\$RAMDISK_DIR/\$SCRIPT_NAME"
chmod +x "\$RAMDISK_DIR/\$SCRIPT_NAME"

echo "[*] Executing script from RAM..."
"\$RAMDISK_DIR/\$SCRIPT_NAME"
EOF

sudo chmod +x "$SCRIPT_DIR/ramdisk-launcher.sh"

# ✅ 4. Create a systemd service that runs the launcher at boot
echo "[*] Setting up systemd service..."
cat <<EOF | sudo tee "/etc/systemd/system/${SERVICE_NAME}.service" > /dev/null
[Unit]
Description=Run script from RAM disk at boot
After=multi-user.target

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

echo "[✔] All done. Reboot to test RAM disk boot script."
