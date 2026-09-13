#!/bin/bash
set -e

echo "=== 1. Timezone & Hardware Clock Sync ==="
ln -sf /usr/share/zoneinfo/Asia/Ho_Chi_Minh /etc/localtime
hwclock --systohc

echo "=== 2. Enable Multilib Repository ==="
sed -i '/^#\[multilib\]/{s/^#//;n;s/^#//}' /etc/pacman.conf
pacman -Sy

echo "=== 3. Add User 'nghia' & Set Password ==="
useradd -m -G wheel -s /bin/bash nghia
echo "Set password for user nghia:"
passwd nghia

echo "=== 4. Configure Sudo for Wheel Group ==="
echo "%wheel ALL=(ALL:ALL) ALL" > /etc/sudoers.d/wheel
chmod 0440 /etc/sudoers.d/wheel

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
cat << 'EOF' >> /home/nghia/.bash_profile
# Export DBUS and XDG Variables
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
export XDG_CONFIG_HOME="$HOME/.config"
export XDG_DATA_HOME="$HOME/.local/share"
export XDG_CACHE_HOME="$HOME/.cache"
EOF

chown nghia:nghia /home/nghia/.bash_profile

echo "=== 7. Install AUR Helper (yay) as User nghia ==="
su - nghia << 'YAY_EOF'
cd /tmp
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si --noconfirm
cd /tmp && rm -rf /tmp/yay
YAY_EOF

echo "=== 8. Enable Global System Services ==="
systemctl enable NetworkManager
systemctl enable firewalld
systemctl enable nftables
systemctl enable bluetooth
systemctl enable acpid
systemctl enable tlp

echo "=== 9. User Environment Verification & Local Services ==="
su - nghia << 'USER_EOF'
echo "Checking Environment Variables:"
echo "XDG_RUNTIME_DIR: $XDG_RUNTIME_DIR"
echo "DBUS_SESSION_BUS_ADDRESS: $DBUS_SESSION_BUS_ADDRESS"

# Enable and start pipewire audio stack
systemctl --user daemon-reload
systemctl --user enable --now pipewire pipewire-pulse wireplumber
USER_EOF

echo "=== 10. Check ACPI and Brightness Control ==="
# Verify ACPI event listening capability
acpi -b || true
brightnessctl info || true

echo "Setup script completed successfully."
