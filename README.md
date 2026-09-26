ash

PACKAGE=(
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

    pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber
    gst-plugin-pipewire alsa-utils

    networkmanager firewalld
    bluez bluez-utils

    v4l-utils acpi acpid brightnessctl tlp tlp-rdw
    sof-firmware

    mesa
    vulkan-radeon
    vulkan-icd-loader
    xf86-video-amdgpu
    libva libva-utils

    lib32-mesa
    lib32-vulkan-radeon
    lib32-vulkan-icd-loader

    git neovim nano
    eza yazi ripgrep fd fzf bat btop htop
    tmux jq tree fastfetch
    reflector tree-sitter-cli fish


)

cnt=0
cnt_de=0
if [[ "$1"  == "-d" ]]; then
    for i in ${PACKAGE[@]}; do

        if (! pacman -Q "$i" &>/dev/null); then

            if (pacman -Si "$i" &>/dev/null); then
                sudo pacman -S $i
            fi
        fi
    done
    exit 0
fi
for i in ${PACKAGE[@]}; do
    if (pacman -Q "$i" &>/dev/null); then
        echo "$i: Installed"
        cnt=$((cnt+1));
    else
        if (pacman -Si "$i" &>/dev/null); then
            echo "$i: Uninstalled"
        else
            echo "$i: Doesnt exist"
            cnt_de=$((cnt_de +1))
        fi
    fi
done
echo "Doesnt exist: $cnt_de"
echo "Installed: $cnt of $((${#PACKAGE[@]} - $cnt_de))"
