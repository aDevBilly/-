#!/usr/bin/env bash
# Arch Linux: install packages and enable services.
# Safe to rerun. Missing or failed optional items are skipped.
# Run as root.

set -euo pipefail

[[ ${EUID} -eq 0 ]] || { echo "Run as root." >&2; exit 1; }
command -v pacman >/dev/null || { echo "pacman not found." >&2; exit 1; }

log()  { printf '\n==> %s\n' "$*"; }
warn() { printf '==> WARNING: %s\n' "$*" >&2; }

SKIPPED_PKGS=()
LIB32_STATUS="skipped (multilib off)"

install_official() {
    local -a pkgs=("$@")
    local -a available=()
    local pkg
    for pkg in "${pkgs[@]}"; do
        if pacman -Si "$pkg" >/dev/null 2>&1 || pacman -Q "$pkg" >/dev/null 2>&1; then
            available+=("$pkg")
        else
            warn "skip missing repo package: $pkg"
            SKIPPED_PKGS+=("$pkg")
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
    unzip 7zip tar gzip xz
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
    mesa
    vulkan-radeon
    vulkan-icd-loader
    xf86-video-amdgpu
    libva libva-utils
)
LIB32_PKGS=(
    lib32-mesa
    lib32-vulkan-radeon
    lib32-vulkan-icd-loader
)

multilib_enabled() {
    pacman-conf --repo-list 2>/dev/null | grep -qx multilib
}

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

if multilib_enabled; then
    log "multilib enabled; install 32-bit GPU libs"
    install_official "${LIB32_PKGS[@]}"
    LIB32_STATUS="attempted"
else
    warn "multilib is not enabled; skipping lib32 packages"
    LIB32_STATUS="skipped (multilib off)"
    SKIPPED_PKGS+=("${LIB32_PKGS[@]}")
fi

log "Install optional extras"
install_official "${OPTIONAL_PKGS[@]}"

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

# --- final report ---
hr() { printf '%s\n' "------------------------------------------------------------"; }
pkg_line() {
    local pkg=$1
    local ver size
    if pacman -Q "$pkg" >/dev/null 2>&1; then
        ver=$(pacman -Q "$pkg" | awk '{print $2}')
        size=$(pacman -Qi "$pkg" 2>/dev/null | awk -F': ' '/^Installed Size/ {print $2}')
        printf '  OK    %-28s %-20s %s\n' "$pkg" "$ver" "${size:-}"
        return 0
    fi
    local skipped=0 s
    for s in "${SKIPPED_PKGS[@]+"${SKIPPED_PKGS[@]}"}"; do
        [[ $s == "$pkg" ]] && skipped=1 && break
    done
    if ((skipped)); then
        printf '  SKIP  %s\n' "$pkg"
        return 0
    fi
    printf '  MISS  %s\n' "$pkg"
    return 1
}
report_group() {
    local title=$1 count_as_error=$2
    shift 2
    local miss=0 pkg
    printf '\n%s\n' "$title"
    for pkg in "$@"; do
        pkg_line "$pkg" || miss=$((miss + 1))
    done
    if [[ ${count_as_error} == 1 ]]; then
        GROUP_MISS=$((GROUP_MISS + miss))
    fi
}

log "Setup report"
hr
echo "Host"
printf '  hostname:     %s\n' "$(hostname 2>/dev/null || echo unknown)"
printf '  kernel:       %s\n' "$(uname -r)"
printf '  arch:         %s\n' "$(uname -m)"
if [[ -f /etc/os-release ]]; then
    # shellcheck disable=SC1091
    . /etc/os-release
    printf '  os:           %s\n' "${PRETTY_NAME:-Arch Linux}"
fi
if command -v timedatectl >/dev/null; then
    printf '  timezone:     %s\n' "$(timedatectl show -p Timezone --value 2>/dev/null || true)"
    printf '  NTP:          %s\n' "$(timedatectl show -p NTP --value 2>/dev/null || true)"
    printf '  synchronized: %s\n' "$(timedatectl show -p NTPSynchronized --value 2>/dev/null || true)"
fi
printf '  pacman repos: %s\n' "$(pacman-conf --repo-list 2>/dev/null | tr '\n' ' ')"
if multilib_enabled; then
    echo "  multilib:     enabled"
else
    echo "  multilib:     disabled"
fi
echo "  lib32 pkgs:   ${LIB32_STATUS}"
echo "  yay:          skipped by design"

if command -v lspci >/dev/null; then
    echo
    echo "Graphics"
    lspci -nn | grep -Ei 'vga|3d|display' | sed 's/^/  /' || echo "  (none detected)"
fi

GROUP_MISS=0
report_group "Core" 1 "${CORE_PKGS[@]}"
report_group "Audio" 1 "${AUDIO_PKGS[@]}"
report_group "Network / bluetooth" 1 "${NET_PKGS[@]}"
report_group "Hardware / power" 1 "${HW_PKGS[@]}"
report_group "GPU (64-bit)" 1 "${GPU_PKGS[@]}"
report_group "GPU (32-bit)" 0 "${LIB32_PKGS[@]}"
report_group "Extra CLI" 1 "${EXTRA_PKGS[@]}"
report_group "Optional" 0 "${OPTIONAL_PKGS[@]}"

echo
echo "Services"
for svc in "${SERVICES[@]}"; do
    if ! systemctl cat "$svc" >/dev/null 2>&1; then
        printf '  MISSING  %s\n' "$svc"
        continue
    fi
    en=$(systemctl is-enabled "$svc" 2>/dev/null || true)
    ac=$(systemctl is-active "$svc" 2>/dev/null || true)
    printf '  %-9s %-8s %s\n' "$en" "$ac" "$svc"
done
echo
echo "Conflicts / masks"
printf '  nftables.service:          %s / %s\n' \
    "$(systemctl is-enabled nftables.service 2>/dev/null || echo disabled)" \
    "$(systemctl is-active nftables.service 2>/dev/null || echo inactive)"
printf '  systemd-rfkill.service:    %s\n' \
    "$(systemctl is-enabled systemd-rfkill.service 2>/dev/null || echo masked)"
printf '  systemd-rfkill.socket:     %s\n' \
    "$(systemctl is-enabled systemd-rfkill.socket 2>/dev/null || echo masked)"

echo
echo "Live checks"
if command -v nmcli >/dev/null; then
    printf '  NetworkManager: %s\n' "$(nmcli -t -f STATE general 2>/dev/null || echo unknown)"
fi
if command -v firewall-cmd >/dev/null; then
    printf '  firewalld:      %s  zone=%s\n' \
        "$(firewall-cmd --state 2>/dev/null || echo unknown)" \
        "$(firewall-cmd --get-default-zone 2>/dev/null || echo n/a)"
fi
if command -v bluetoothctl >/dev/null; then
    printf '  bluetooth:      %s\n' "$(systemctl is-active bluetooth.service 2>/dev/null || true)"
fi
if command -v tlp-stat >/dev/null; then
    tlp-stat -s 2>/dev/null | awk '/^System/{p=1} p && NF{print "  "$0} /^$/{if(p) exit}' || echo "  TLP installed"
fi
if command -v acpi >/dev/null; then
    acpi -b 2>/dev/null | sed 's/^/  battery: /' || true
fi

echo
echo "Skipped packages"
if ((${#SKIPPED_PKGS[@]})); then
    printf '  %s\n' "${SKIPPED_PKGS[@]}"
else
    echo "  (none)"
fi

echo
echo "Notes"
echo "  - yay is not installed."
echo "  - lib32 packages require an enabled [multilib] repo."
echo "  - PipeWire user units start at graphical/login session."
echo "  - No desktop environment was installed."
hr
if ((GROUP_MISS)); then
    warn "Report: ${GROUP_MISS} listed package(s) are not installed."
    exit 1
fi
echo "Report: required listed packages are installed and services were processed."

