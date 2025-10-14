#!/usr/bin/env bash
set -euo pipefail

# Defaults
MOUNTPOINT="/mnt/external"
OWNER_USER="${SUDO_USER:-${USER}}"
DEVICE=""
UUID=""

usage() {
  cat <<EOF
Usage: sudo $(basename "$0") [--device /dev/sdX1 | --uuid <UUID>] [--mountpoint /mnt/external] [--owner <username>]

Examples:
  sudo $(basename "$0")
  sudo $(basename "$0") --device /dev/sda1
  sudo $(basename "$0") --uuid 1234-ABCD --mountpoint /mnt/media --owner menulis
EOF
}

# Parse args
while [[ $# -gt 0 ]]; do
  case "$1" in
    --device) DEVICE="${2:-}"; shift 2 ;;
    --uuid)   UUID="${2:-}";   shift 2 ;;
    --mountpoint) MOUNTPOINT="${2:-}"; shift 2 ;;
    --owner)  OWNER_USER="${2:-}"; shift 2 ;;
    -h|--help) usage; exit 0 ;;
    *) echo "Unknown arg: $1"; usage; exit 1 ;;
  esac
done

# Root check
if [[ $EUID -ne 0 ]]; then
  echo "Please run as root (use: sudo $0 ...)"
  exit 1
fi

# Resolve owner UID/GID (use the invoking user if run with sudo)
if ! id "$OWNER_USER" >/dev/null 2>&1; then
  echo "Owner user '$OWNER_USER' does not exist." >&2
  exit 1
fi
OWNER_UID="$(id -u "$OWNER_USER")"
OWNER_GID="$(id -g "$OWNER_USER")"

echo "[1/6] Installing ntfs-3g (if needed)..."
apt-get update -y
apt-get install -y ntfs-3g

echo "[2/6] Detecting NTFS partition..."
if [[ -z "$UUID" ]]; then
  if [[ -n "$DEVICE" ]]; then
    if [[ ! -b "$DEVICE" ]]; then
      echo "Device '$DEVICE' not found." >&2
      exit 1
    fi
    FSTYPE="$(lsblk -no FSTYPE "$DEVICE" | tr -d ' ')"
    if [[ "$FSTYPE" != "ntfs" && "$FSTYPE" != "ntfs3" ]]; then
      echo "Warning: '$DEVICE' is fstype '$FSTYPE' (expected ntfs/ntfs3)." >&2
    fi
    UUID="$(blkid -s UUID -o value "$DEVICE" || true)"
    if [[ -z "$UUID" ]]; then
      echo "Could not read UUID from $DEVICE. Is it partitioned and formatted?" >&2
      exit 1
    fi
  else
    # Auto-pick the largest NTFS partition
    read -r DEVICE UUID < <(
      lsblk -b -rno NAME,FSTYPE,UUID,SIZE,TYPE \
      | awk '$2 ~ /^ntfs|ntfs3$/ && $5=="part"{print $1, $3, $4}' \
      | sort -k3,3nr \
      | awk 'NR==1{print "/dev/"$1, $2}'
    )
    if [[ -z "${DEVICE:-}" || -z "${UUID:-}" ]]; then
      echo "No NTFS partition found. Plug the drive and try again, or pass --device /dev/sdX1." >&2
      exit 1
    fi
  fi
else
  # UUID provided; find its device (optional)
  DEVICE="$(blkid -U "$UUID" 2>/dev/null || true)"
  if [[ -z "$DEVICE" ]]; then
    echo "Note: could not resolve device for UUID=$UUID (that’s OK)."
  fi
fi

echo "Selected partition:"
echo "  DEVICE : ${DEVICE:-unknown}"
echo "  UUID   : $UUID"
echo "  MOUNT  : $MOUNTPOINT"
echo "  OWNER  : $OWNER_USER (uid=$OWNER_UID gid=$OWNER_GID)"

echo "[3/6] Creating mount point..."
mkdir -p "$MOUNTPOINT"

echo "[4/6] Backing up and updating /etc/fstab..."
cp -an /etc/fstab "/etc/fstab.backup.$(date +%Y%m%d-%H%M%S)"

# Remove any existing lines for this UUID or this mountpoint to keep it idempotent
tmpfile="$(mktemp)"
awk -v mp="$MOUNTPOINT" -v id="UUID=$UUID" '
  BEGIN{OFS="\t"}
  $0 ~ /^[[:space:]]*#/ {print; next}
  $1 ~ id || $2 == mp {next}
  {print}
' /etc/fstab > "$tmpfile"
cat "$tmpfile" > /etc/fstab
rm -f "$tmpfile"

# Append our entry
echo -e "UUID=$UUID\t$MOUNTPOINT\tntfs-3g\tdefaults,nofail,uid=$OWNER_UID,gid=$OWNER_GID,umask=022\t0\t0" >> /etc/fstab

echo "[5/6] Mounting..."
# If mounted elsewhere, try to unmount first (best effort)
mountpoint -q "$MOUNTPOINT" && umount -f "$MOUNTPOINT" || true
mount -a

echo "[6/6] Verifying..."
if mountpoint -q "$MOUNTPOINT"; then
  df -h | awk 'NR==1 || $6=="'"$MOUNTPOINT"'"'
  echo "Success: $MOUNTPOINT is mounted with ownership uid=$OWNER_UID gid=$OWNER_GID."
  echo "Tip: try 'ls -la $MOUNTPOINT' to see your files."
else
  echo "Failed to mount at $MOUNTPOINT. Check 'dmesg | tail -n 50' for details." >&2
  exit 1
fi
