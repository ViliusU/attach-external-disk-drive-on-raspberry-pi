# attach-external-ntfs.sh

Idempotent helper script to **detect, persistently mount, and grant ownership** of an **NTFS USB drive** on a freshly installed Raspberry Pi (Raspberry Pi OS).

It installs `ntfs-3g`, finds your NTFS partition (or uses the one you provide), writes a clean `/etc/fstab` entry (by **UUID**), and mounts it at a chosen mount point (default **`/mnt/external`**) with ownership mapped to your user (e.g., **`menulis`**).

---

## ✨ Features

- **No reformat** — Uses your existing NTFS filesystem  
- **Persistent** — Adds a **UUID-based** entry to `/etc/fstab`  
- **Idempotent** — Safe to re-run; de-duplicates old fstab lines  
- **Owner mapping** — Sets `uid/gid` so your user owns the files  
- **Autodetect** — Picks the **largest NTFS partition** if not specified  
- **Immediate mount** — Calls `mount -a` and verifies  

---

## ✅ Requirements

- Raspberry Pi OS (Lite or Full)
- Internet access (to install `ntfs-3g`)
- Sudo privileges
- An NTFS-formatted USB disk attached

---

## 📦 Install

Place the script in your repo (e.g., `scripts/attach-external-ntfs.sh`) and make it executable:

```bash
chmod +x scripts/attach-external-ntfs.sh
