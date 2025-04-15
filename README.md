

## ✅ HOW TO USE

1. Save it:
```bash
nano fast_ram_bootstrap.sh
```

2. Paste the script above and save (`CTRL + O`, `CTRL + X`)

3. Run it:
```bash
chmod +x fast_ram_bootstrap.sh
./fast_ram_bootstrap.sh
```

4. Then reboot:
```bash
sudo reboot
```

---

## 🚀 What Happens After Reboot?

1. On every boot:
   - RAM disk (`/mnt/ramdisk`) is mounted
   - Script from `/opt/ramdisk-script` is copied into RAM
   - Executed from RAM directly (super fast)

2. Output:
```bash
[+] Running from RAM disk at ...
[+] Performing high-speed operation...
[+] Done. This was fast and clean.
