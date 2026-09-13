#!/bin/bash
set -euo pipefail

echo "=== 1. Timezone & Hardware Clock Sync ==="
if [ -f /usr/share/zoneinfo/Asia/Ho_Chi_Minh ]; then
    ln -sf /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime
    hwclock --systohc
fi

echo "=== 2. Enable Multilib Repository ==="
if grep -q "^#\[multilib\]" /etc/pacman.conf; then
    sed -i '/^#\[multilib\]/{s/^#//;n;s/^#//}' /etc/pacman.conf
fi
pacman -Sy

echo "=== 3. Add User 'nghia' & Set Password ==="
if ! id "nghia" &>/dev/null; then
    useradd -m -G wheel -s /bin/bash nghia
    echo "Set password for user nghia:"
    passwd nghia
else
    echo "User 'nghia' already exists. Skipping creation."
    # Ensure user is in wheel group
    usermod -aG wheel nghia
fi

echo "=== 4. Configure Sudo for Wheel Group ==="
# Fix for "No such file or directory": Ensure directory exists
mkdir -p /etc/sudoers.d
if [ ! -f /etc/sudoers.d/wheel ]; then
    echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
    chmod 0440 /etc/sudoers.d/wheel
fi

echo "=== 5. Install System & Graphics Packages ==="
pacman -S --needed --noconfirm \
    base base-devel sudo \
    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
    bluez bluez-utils \
    networkmanager firewalld nftables speedtest-cli \
    v4l-utils acpi acpid brightnessctl tlp tlp-rdw \
    mesa lib32-mesa vulkan-radeon lib32-vulkan-radeon xf86-video-amdgpu libva-mesadriver \
    eza yazi ripgrep fd fzf bat btop neovim nano git

echo "=== 6. Export DBUS & XDG Variables to .bash_profile ==="
BASH_PROFILE="/home/nghia/.bash_profile"
touch "$BASH_PROFILE"

if ! grep -q "XDG_RUNTIME_DIR" "$BASH_PROFILE"; then
    cat << 'EOF' >> "$BASH_PROFILE"

# Export DBUS and XDG Variables
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_CACHE_HOME="$HOME/.cache"
EOF
    chown nghia:nghia "$BASH_PROFILE"
fi

echo "=== 7. Install AUR Helper (yay) as User nghia ==="
if ! command -v yay &>/dev/null; then
    su - nghia << 'YAY_EOF'
    rm -rf /tmp/yay
    git clone https://aur.archlinux.org/yay.git /tmp/yay
    cd /tmp/yay
    makepkg -si --noconfirm
    rm -rf /tmp/yay
YAY_EOF
else
    echo "AUR helper 'yay' is already installed."
fi

echo "=== 8. Enable Global System Services ==="
for svc in NetworkManager firewalld nftables bluetooth acpid tlp; do
    systemctl is-enabled --quiet "$svc" || systemctl enable "$svc"
done

echo "=== 9. User Environment Verification & Local Services ==="
su - nghia << 'USER_EOF'
echo "Checking Environment Variables:"
echo "XDG_RUNTIME_DIR: $XDG_RUNTIME_DIR"
echo "DBUS_SESSION_BUS_ADDRESS: $DBUS_SESSION_BUS_ADDRESS"

# Enable and start pipewire audio stack if running inside an active session
systemctl --user daemon-reload || true
systemctl --user enable --now pipewire pipewire-pulse wireplumber || true
USER_EOF

echo "=== 10. Check ACPI and Brightness Control ==="
acpi -b || true
brightnessctl info || true

echo "Setup script completed successfully."
