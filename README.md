#!/usr/bin/env bash
# Arch Linux: install packages and enable services.
# Safe to rerun. Missing or failed optional items are skipped.
# Run as root.

set -euo pipefail

[[ ${EUID} -eq 0 ]] || { echo "Run as root." >&2; exit 1; }
command -v pacman >/dev/null || { echo "pacman not found." >&2; exit 1; }

log()  { printf '\n==> %s\n' "$*"; }
warn() { printf '==> WARNING: %s\n' "$*" >&2; }

install_official() {
    local -a pkgs=("$@")
    local -a available=()
    local pkg
    for pkg in "${pkgs[@]}"; do
        if pacman -Si "$pkg" >/dev/null 2>&1 || pacman -Q "$pkg" >/dev/null 2>&1; then
            available+=("$pkg")
        else
            warn "skip missing repo package: $pkg"
        fi
    done
    ((${#available[@]})) || return 0
    pacman -S --needed --noconfirm "${available[@]}"
}

# --- sync (rerun-safe) ---
log "Sync and upgrade"
pacman -Syu --noconfirm

# --- official packages ---
# Essential core
CORE_PKGS=(
    base base-devel linux linux-headers linux-firmware
    sudo pacman-contrib
    man-db man-pages texinfo
    bash bash-completion which less
    coreutils util-linux procps-ng psmisc
    pciutils usbutils lsof
    iproute2 iputils inetutils
    wget curl rsync openssh
    unzip p7zip tar gzip xz
    dosfstools e2fsprogs ntfs-3g
    smartmontools plocate
    xdg-utils xdg-user-dirs
)

# Audio / desktop plumbing
AUDIO_PKGS=(
    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber
    gst-plugin-pipewire alsa-utils
)

# Network, firewall, bluetooth
NET_PKGS=(
    networkmanager firewalld
    bluez bluez-utils
)

# Laptop power / hardware
HW_PKGS=(
    v4l-utils acpi acpid brightnessctl tlp tlp-rdw
    sof-firmware
)

# AMD graphics (mesa now provides libva-mesa-driver)
GPU_PKGS=(
    mesa lib32-mesa
    vulkan-radeon lib32-vulkan-radeon
    vulkan-icd-loader lib32-vulkan-icd-loader
    xf86-video-amdgpu
    libva libva-utils
)

# Extra CLI / quality of life
EXTRA_PKGS=(
    git neovim nano
    eza yazi ripgrep fd fzf bat btop htop
    tmux jq tree fastfetch
    reflector
)

# Optional extras: skipped if the repo does not have them
OPTIONAL_PKGS=(
    speedtest-cli
)

log "Install essential core"
install_official "${CORE_PKGS[@]}"

log "Install audio, network, hardware, GPU, extra"
install_official "${AUDIO_PKGS[@]}" "${NET_PKGS[@]}" "${HW_PKGS[@]}" "${GPU_PKGS[@]}" "${EXTRA_PKGS[@]}"

log "Install optional extras"
install_official "${OPTIONAL_PKGS[@]}"

# --- yay (optional; never fail the script) ---
install_yay() {
    if command -v yay >/dev/null; then
        echo "yay already installed."
        return 0
    fi
    command -v git >/dev/null || { warn "git missing; skip yay"; return 0; }
    command -v makepkg >/dev/null || { warn "makepkg missing; skip yay"; return 0; }

    local build_user src
    build_user="$(loginctl list-users --no-legend 2>/dev/null | awk '$2 != "root" {print $2; exit}')"
    [[ -n ${build_user:-} ]] || build_user="$(getent passwd 1000 | cut -d: -f1 || true)"

    src=$(mktemp -d /tmp/yay.XXXXXX)
    if [[ -n ${build_user:-} ]] && id "$build_user" >/dev/null 2>&1; then
        chown "$build_user:$build_user" "$src"
        sudo -u "$build_user" git clone https://aur.archlinux.org/yay.git "$src" || { rm -rf "$src"; return 1; }
        ( cd "$src" && sudo -u "$build_user" makepkg -s --noconfirm ) || { rm -rf "$src"; return 1; }
    else
        warn "no unprivileged user for makepkg; skip yay"
        rm -rf "$src"
        return 0
    fi
    pacman -U --noconfirm "$src"/yay-*.pkg.tar.zst || { rm -rf "$src"; return 1; }
    rm -rf "$src"
}

log "Install yay (skip on error)"
install_yay || warn "yay install failed; continuing without it."

# --- services (idempotent) ---
log "Enable services"
systemctl disable --now nftables.service 2>/dev/null || true

SERVICES=(
    NetworkManager.service
    NetworkManager-dispatcher.service
    firewalld.service
    bluetooth.service
    acpid.service
    tlp.service
    systemd-timesyncd.service
    fstrim.timer
    plocate-updatedb.timer
)

for svc in "${SERVICES[@]}"; do
    if systemctl cat "$svc" >/dev/null 2>&1; then
        systemctl enable --now "$svc" >/dev/null
        echo "enabled: $svc"
    else
        warn "skip missing unit: $svc"
    fi
done

systemctl mask systemd-rfkill.service systemd-rfkill.socket >/dev/null 2>&1 || true

# --- rerun check ---
log "Dependency check"
CHECK_PKGS=(
    base base-devel sudo git
    networkmanager firewalld
    pipewire wireplumber pipewire-pulse
    bluez tlp mesa
)
missing=0
for pkg in "${CHECK_PKGS[@]}"; do
    if pacman -Q "$pkg" >/dev/null 2>&1; then
        printf '  OK   %s\n' "$pkg"
    else
        printf '  MISS %s\n' "$pkg"
        missing=$((missing + 1))
    fi
done

echo
echo "services:"
for svc in "${SERVICES[@]}"; do
    if systemctl is-enabled "$svc" >/dev/null 2>&1; then
        printf '  OK   %s (%s)\n' "$svc" "$(systemctl is-active "$svc" 2>/dev/null || true)"
    else
        printf '  OFF  %s\n' "$svc"
    fi
done

if command -v yay >/dev/null; then
    echo "yay: $(command -v yay)"
else
    echo "yay: not installed"
fi

echo
if ((missing)); then
    warn "$missing required package(s) still missing."
    exit 1
fi
echo "Packages installed, services enabled, basic dependencies OK."
echo "PipeWire user units start at login."
