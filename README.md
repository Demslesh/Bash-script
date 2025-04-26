#!/bin/bash

set -e

echo "Посмотрим что подойдет..."
if command -v apt > /dev/null; then
    PKG_MANAGER="apt"
    DHCP_PACKAGE="isc-dhcp-server"
    DNS_PACKAGE="bind9"
    SSH_PACKAGE="openssh-server"
elif command -v apt-get > /dev/null; then
    PKG_MANAGER="apt-get"
    DHCP_PACKAGE="isc-dhcp-server"
    DNS_PACKAGE="bind9"
    SSH_PACKAGE="openssh-server"
elif command -v dnf > /dev/null; then
    PKG_MANAGER="dnf"
    DHCP_PACKAGE="dhcp-server"
    DNS_PACKAGE="bind"
    SSH_PACKAGE="openssh-server"
elif command -v yum > /dev/null; then
    PKG_MANAGER="yum"
    DHCP_PACKAGE="dhcp-server"
    DNS_PACKAGE="bind"
    SSH_PACKAGE="openssh-server"
elif command -v pacman > /dev/null; then
    PKG_MANAGER="pacman"
    DHCP_PACKAGE="dhcp"
    DNS_PACKAGE="bind"
    SSH_PACKAGE="openssh"
elif command -v zypper > /dev/null; then
    PKG_MANAGER="zypper"
    DHCP_PACKAGE="dhcp-server"
    DNS_PACKAGE="bind"
    SSH_PACKAGE="openssh"
elif command -v apk > /dev/null; then
    PKG_MANAGER="apk"
    DHCP_PACKAGE="dhcp"
    DNS_PACKAGE="bind"
    SSH_PACKAGE="openssh"
else
    echo "Ошибка: Не удалось определить пакетный менеджер!"
    exit 1
fi

echo "Попался такой: $PKG_MANAGER"


echo "Обновим пакеты..."
if [[ "$PKG_MANAGER" == "apt" || "$PKG_MANAGER" == "apt-get" ]]; then
    sudo $PKG_MANAGER update && sudo $PKG_MANAGER upgrade -y && sudo $PKG_MANAGER autoremove -y
elif [[ "$PKG_MANAGER" == "dnf" || "$PKG_MANAGER" == "yum" ]]; then
    sudo $PKG_MANAGER update -y
elif [[ "$PKG_MANAGER" == "pacman" ]]; then
    sudo pacman -Syu --noconfirm
elif [[ "$PKG_MANAGER" == "zypper" ]]; then
    sudo zypper refresh && sudo zypper update -y
elif [[ "$PKG_MANAGER" == "apk" ]]; then
    sudo apk update && sudo apk upgrade
fi


echo "Устанавливаем SSH, DHCP, DNS и iptables..."
PACKAGES=("$SSH_PACKAGE" "$DHCP_PACKAGE" "$DNS_PACKAGE" "iptables")

for package in "${PACKAGES[@]}"; do
    if command -v rpm > /dev/null; then
        CHECK_CMD="rpm -qa | grep -q $package"
    else
        CHECK_CMD="dpkg -l | grep -q $package"
    fi

    if ! eval $CHECK_CMD; then
        if sudo $PKG_MANAGER install -y "$package" > /dev/null 2>&1; then
            echo "$package успешно установлен."
        else
            echo "Ошибка установки $package. Возможно, он недоступен для $PKG_MANAGER."
        fi
    else
        echo "$package уже установлен."
    fi
done


echo "Определим доступные нам службы..."
if systemctl list-units --type=service | grep -q "dhcpd"; then
    DHCP_SERVICE="dhcpd"
elif systemctl list-units --type=service | grep -q "isc-dhcp-server"; then
    DHCP_SERVICE="isc-dhcp-server"
fi

if systemctl list-units --type=service | grep -q "named"; then
    DNS_SERVICE="named"
elif systemctl list-units --type=service | grep -q "bind9"; then
    DNS_SERVICE="bind9"
fi


echo "В рестарт службы..."
SERVICES=("sshd" "$DHCP_SERVICE" "$DNS_SERVICE")

for service in "${SERVICES[@]}"; do
    if [[ -n "$service" ]]; then
        sudo systemctl restart "$service"
        echo "Служба $service перезапущена."
    else
        echo "Служба $service не найдена в системе."
    fi
done

echo "Настройка завершена!"
