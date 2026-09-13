#/!usr/bin/env bash
# Arch Linux post-install bootstrap
# Review + hardened rewrite of the original setup script.
#
# Usage (as root):
#   NEW_USER=nghia TZ=Asia/Ho_Chi_Minh bash arch-post-install.sh
#
# Optional environment:
#   NEW_USER          default: nghia
#   TZ                default: Asia/Ho_Chi_Minh
#   NEW_USER_PASSWORD set this to skip the interactive passwd prompt
#   SKIP_AUR=1        skip yay install
#   SKIP_AMDGPU=1     skip AMD-specific graphics packages

set -euo pipefail

NEW_USER="${NEW_USER:-nghia}"
TZ="${TZ:-Asia/Ho_Chi_Minh}"
SKIP_AUR="${SKIP_AUR:-0}"
SKIP_AMDGPU="${SKIP_AMDGPU:-0}"

log()  { printf '\n\033[1;32m==>\033[0m %s\n' "$*"; }
warn() { printf '\033[1;33m==> WARNING:\033[0m %s\n' "$*" >&2; }
die()  { printf '\033[1;31m==> ERROR:\033[0m %s\n' "$*" >&2; exit 1; }

require_root() {
    [[ ${EUID} -eq 0 ]] || die "Run this script as root."
}

require_arch() {
    [[ -f /etc/os-release ]] || die "/etc/os-release is missing."
    # shellcheck disable=SC1091
    . /etc/os-release
    [[ ${ID:-} == arch ]] || die "This script is for Arch Linux (found ID=${ID:-unknown})."
    command -v pacman >/dev/null || die "pacman not found."
}

backup_file() {
    local file=$1
    [[ -f ${file} ]] || return 0
    local stamp
    stamp=$(date +%Y%m%d-%H%M%S)
    cp -a "${file}" "${file}.bak.${stamp}"
}

enable_unit() {
    local unit=$1
    if systemctl cat "${unit}" >/dev/null 2>&1; then
        systemctl enable --now "${unit}"
    else
        warn "Unit ${unit} is not available; skipping."
    fi
}

require_root
require_arch

# ---------------------------------------------------------------------------
log "1. Timezone, NTP, hardware clock"
# Prefer timedatectl over a raw symlink. Keep RTC in UTC (Arch default).
if timedatectl list-timezones | grep -qx "${TZ}"; then
    timedatectl set-timezone "${TZ}"
else
    die "Timezone '${TZ}' is not available on this system."
fi
timedatectl set-ntp true || warn "Could not enable NTP (systemd-timesyncd may be missing)."
# Only write the RTC if hwclock exists; do not switch the RTC to localtime.
if command -v hwclock >/dev/null; then
    hwclock --systohc --utc || warn "hwclock --systohc failed."
fi

# ---------------------------------------------------------------------------
log "2. Pacman quality-of-life + multilib"
backup_file /etc/pacman.conf

# Color + parallel downloads (idempotent).
if grep -q '^#Color' /etc/pacman.conf; then
    sed -i 's/^#Color/Color/' /etc/pacman.conf
fi
if ! grep -q '^ParallelDownloads' /etc/pacman.conf; then
    sed -i '/^#ParallelDownloads/s/^#//' /etc/pacman.conf
    if ! grep -q '^ParallelDownloads' /etc/pacman.conf; then
        sed -i '/^\[options\]/a ParallelDownloads = 5' /etc/pacman.conf
    fi
fi

# Enable [multilib] and the following Include line, even if only one is commented.
if grep -q '^\[multilib\]' /etc/pacman.conf; then
    log "multilib already enabled."
else
    # Handles the stock commented block:
    #   #[multilib]
    #   #Include = /etc/pacman.d/mirrorlist
    python - <<'PY' || die "Failed to enable multilib in /etc/pacman.conf"
from pathlib import Path
p = Path("/etc/pacman.conf")
text = p.read_text()
old = "#[multilib]\n#Include = /etc/pacman.d/mirrorlist"
new = "[multilib]\nInclude = /etc/pacman.d/mirrorlist"
if old not in text:
    old = "#[multilib]\r\n#Include = /etc/pacman.d/mirrorlist"
    new = "[multilib]\r\nInclude = /etc/pacman.d/mirrorlist"
if old not in text:
    raise SystemExit("Could not find the stock commented [multilib] block")
p.write_text(text.replace(old, new, 1))
PY
fi

# Never use pacman -Sy without -u: that is a partial upgrade.
pacman -Syu --noconfirm

# ---------------------------------------------------------------------------
log "3. Create user '${NEW_USER}'"
if ! id "${NEW_USER}" >/dev/null 2>&1; then
    useradd -m -G wheel -s /bin/bash "${NEW_USER}"
else
    log "User '${NEW_USER}' already exists; ensuring wheel membership."
    usermod -aG wheel "${NEW_USER}"
fi

if [[ -n ${NEW_USER_PASSWORD:-} ]]; then
    printf '%s:%s\n' "${NEW_USER}" "${NEW_USER_PASSWORD}" | chpasswd
    unset NEW_USER_PASSWORD
else
    if [[ -t 0 ]]; then
        log "Set a password for ${NEW_USER}"
        passwd "${NEW_USER}"
    else
        warn "No TTY and NEW_USER_PASSWORD is unset; user may have no password."
    fi
fi

# ---------------------------------------------------------------------------
log "4. Sudo for the wheel group"
mkdir -p /etc/sudoers.d
chmod 0750 /etc/sudoers.d
SUDOERS_DROPIN=/etc/sudoers.d/10-wheel
cat > "${SUDOERS_DROPIN}" <<'EOF'
%wheel ALL=(ALL:ALL) ALL
EOF
chmod 0440 "${SUDOERS_DROPIN}"
visudo -cf "${SUDOERS_DROPIN}" || {
    rm -f "${SUDOERS_DROPIN}"
    die "sudoers drop-in failed visudo validation."
}

# ---------------------------------------------------------------------------
log "5. Official packages"
# libva-mesadriver is not a package. libva-mesa-driver was merged into mesa
# (mesa now Provides/Replaces libva-mesa-driver).
# xf86-video-amdgpu is an Xorg DDX; Wayland/KMS use the kernel amdgpu module + mesa.
# firewalld already uses nftables as its backend — do not also enable nftables.service.
AMD_PKGS=()
if [[ ${SKIP_AMDGPU} != 1 ]]; then
    AMD_PKGS=(
        mesa
        lib32-mesa
        vulkan-radeon
        lib32-vulkan-radeon
        vulkan-icd-loader
        lib32-vulkan-icd-loader
        xf86-video-amdgpu
        libva
        libva-utils
    )
fi

pacman -S --needed --noconfirm \
    base base-devel sudo \
    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber alsa-utils \
    bluez bluez-utils \
    networkmanager firewalld \
    v4l-utils acpi acpid brightnessctl tlp tlp-rdw \
    eza yazi ripgrep fd fzf bat btop neovim nano git \
    xdg-user-dirs xdg-utils man-db man-pages \
    "${AMD_PKGS[@]}"

# Optional extra from the original list. Keep it optional so a missing package
# cannot abort the whole run after the core set is installed.
if ! pacman -S --needed --noconfirm speedtest-cli; then
    warn "speedtest-cli is unavailable; skipping."
fi

# ---------------------------------------------------------------------------
log "6. User XDG directories (do NOT hardcode the session bus)"
# systemd-logind already exports XDG_RUNTIME_DIR and starts the session bus.
# Hardcoding those two variables in ~/.bash_profile breaks su/ssh/cron and
# fights a real graphical login. Only set the optional XDG_*_HOME overrides.
USER_HOME=$(getent passwd "${NEW_USER}" | cut -d: -f6)
[[ -n ${USER_HOME} && -d ${USER_HOME} ]] || die "Home directory for ${NEW_USER} not found."

install -d -o "${NEW_USER}" -g "${NEW_USER}" -m 0755 \
    "${USER_HOME}/.config" \
    "${USER_HOME}/.local/share" \
    "${USER_HOME}/.cache"

BASH_PROFILE="${USER_HOME}/.bash_profile"
if [[ ! -f ${BASH_PROFILE} ]]; then
    install -o "${NEW_USER}" -g "${NEW_USER}" -m 0644 /dev/null "${BASH_PROFILE}"
fi
if ! grep -q 'XDG_CONFIG_HOME' "${BASH_PROFILE}"; then
    cat >> "${BASH_PROFILE}" <<'EOF'

# XDG base directories (session bus / runtime dir come from logind)
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_CACHE_HOME="$HOME/.cache"
EOF
    chown "${NEW_USER}:${NEW_USER}" "${BASH_PROFILE}"
fi

sudo -u "${NEW_USER}" xdg-user-dirs-update || true

# ---------------------------------------------------------------------------
log "7. AUR helper (yay)"
if [[ ${SKIP_AUR} == 1 ]]; then
    log "SKIP_AUR=1; not installing yay."
elif command -v yay >/dev/null; then
    log "yay is already installed."
else
    # Build as the unprivileged user, install as root.
    # makepkg -si would prompt for a sudo password and is a poor fit here.
    YAY_SRC=$(mktemp -d /tmp/yay.XXXXXX)
    chown "${NEW_USER}:${NEW_USER}" "${YAY_SRC}"
    sudo -u "${NEW_USER}" git clone https://aur.archlinux.org/yay.git "${YAY_SRC}"
    ( cd "${YAY_SRC}" && sudo -u "${NEW_USER}" makepkg -s --noconfirm )
    pacman -U --noconfirm "${YAY_SRC}"/yay-*.pkg.tar.zst
    rm -rf "${YAY_SRC}"
fi

# ---------------------------------------------------------------------------
log "8. System services"
# Pick ONE firewall manager. firewalld drives nftables internally.
systemctl disable --now nftables.service 2>/dev/null || true

enable_unit NetworkManager.service
enable_unit NetworkManager-dispatcher.service   # required by tlp-rdw
enable_unit firewalld.service
enable_unit bluetooth.service
enable_unit acpid.service
enable_unit tlp.service

# TLP and systemd-rfkill fight over radio devices.
systemctl mask systemd-rfkill.service systemd-rfkill.socket || true

# Let user units start without a graphical login (needed for pipewire enable).
loginctl enable-linger "${NEW_USER}" || warn "Could not enable linger for ${NEW_USER}."

# ---------------------------------------------------------------------------
log "9. User services (PipeWire)"
# systemctl --user only works when the user manager is running.
if sudo -u "${NEW_USER}" XDG_RUNTIME_DIR="/run/user/$(id -u "${NEW_USER}")" \
        systemctl --user daemon-reload 2>/dev/null; then
    sudo -u "${NEW_USER}" XDG_RUNTIME_DIR="/run/user/$(id -u "${NEW_USER}")" \
        systemctl --user enable --now pipewire.socket pipewire-pulse.socket wireplumber.service \
        || warn "Could not start PipeWire user units now; they will start at next login."
else
    warn "User systemd instance is not running yet."
    warn "PipeWire sockets are usually enabled by package preset and will start at login."
fi

# ---------------------------------------------------------------------------
log "10. Hardware sanity checks (non-fatal)"
acpi -b || true
brightnessctl info || true
if command -v lspci >/dev/null; then
    lspci -nn | grep -Ei 'vga|3d|display' || true
fi

log "Done."
printf '%s\n' \
    "" \
    "Next steps:" \
    "  - Reboot so linger, firmware, and user units settle." \
    "  - Confirm audio after login: systemctl --user status pipewire pipewire-pulse wireplumber" \
    "  - Confirm firewall:          firewall-cmd --state" \
    "  - Confirm time:              timedatectl" \
    "  - Confirm TLP:               tlp-stat -s"
