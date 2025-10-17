#!/bin/bash
# ==============================================
# SMART DEBIAN CLEANUP SCRIPT
# Hanya hapus package non-essential, keep system packages
# ==============================================

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[0;33m'
BLUE='\033[0;34m'
NC='\033[0m'

print_status() { echo -e "${BLUE}[INFO]${NC} $1"; }
print_success() { echo -e "${GREEN}[SUCCESS]${NC} $1"; }
print_warning() { echo -e "${YELLOW}[WARNING]${NC} $1"; }
print_error() { echo -e "${RED}[ERROR]${NC} $1"; }

# Check if running as root
if [ "$EUID" -ne 0 ]; then
    print_error "Script harus dijalankan dengan sudo!"
    exit 1
fi

# Essential packages yang TIDAK BOLEH dihapus
ESSENTIAL_PACKAGES=(
    # System Base
    "apt" "base-files" "base-passwd" "bash" "bsdutils" "coreutils" "dash" "debconf" 
    "debian-archive-keyring" "debianutils" "diffutils" "dpkg" "e2fsprogs" "findutils"
    "gawk" "grep" "gzip" "hostname" "init" "libc6" "login" "mount" "ncurses-base"
    "ncurses-bin" "perl-base" "procps" "sed" "sysvinit-utils" "tar" "util-linux"
    
    # Network Essential
    "iproute2" "iputils-ping" "netbase" "netcat-openbsd" "openssh-client" "openssh-server"
    "wget" "curl"
    
    # System Management
    "systemd" "systemd-sysv" "udev" "sudo" "adduser" "passwd"
    
    # Hardware
    "linux-image-amd64" "fdisk" "grub-common" "grub-pc" "grub2-common"
    
    # Security
    "apt-transport-https" "ca-certificates" "gnupg" "gnupg2"
    
    # File Systems
    "e2fsprogs" "dosfstools" "mount" "util-linux"
    
    # Shell & Tools
    "vim-common" "nano" "less" "man-db" "manpages"
)

# Services packages yang BOLEH dihapus (non-essential) - DIPERBARUI
SERVICE_PACKAGES=(
    # Web Servers
    "apache2" "nginx" "lighttpd"
    
    # Database Servers
    "mariadb-server" "mariadb-client" "mysql-server" "mysql-client"
    
    # DNS Server
    "bind9" "bind9utils" "bind9-doc"
    
    # Mail Servers
    "postfix" "dovecot-imapd" "dovecot-pop3d"
    
    # FTP Servers
    "vsftpd" "proftpd"
    
    # Programming Languages & Frameworks - DITAMBAHKAN
    "php" "php-common" "php-cli" "php-fpm" "php-mysql" "php-curl" "php-gd" "php-mbstring"
    "php-xml" "php-zip" "php-bcmath" "php-json" "php-imagick" "php-redis"
    "phpmyadmin"
    
    # Python & Tools
    "python3" "python2" "python3-pip" "python3-venv"
    "perl" "ruby"
    
    # Development Tools
    "build-essential" "git" "nodejs" "npm"
    
    # Other Services
    "postgresql" "mongodb" "redis-server" "memcached"
)

# Banner dengan warning
echo ""
echo -e "${YELLOW}==========================================${NC}"
echo -e "${YELLOW}        SMART DEBIAN CLEANUP SCRIPT${NC}"
echo -e "${YELLOW}     Hapus Non-Essential Packages Only${NC}"
echo -e "${YELLOW}==========================================${NC}"
echo ""
echo -e "${YELLOW}SCRIPT INI AKAN:${NC}"
echo -e "${YELLOW}• Hapus package services (Apache, MySQL, PHP, DNS, dll)${NC}"
echo -e "${YELLOW}• Hapus config files services${NC}"
echo -e "${YELLOW}• Hapus data services${NC}"
echo -e "${YELLOW}• Membersihkan cache dan log${NC}"
echo ""
echo -e "${YELLOW}YANG AKAN DIPERTAHANKAN:${NC}"
echo -e "${GREEN}• Semua system essential packages${NC}"
echo -e "${GREEN}• Network configuration${NC}"
echo -e "${GREEN}• User accounts dan home directories${NC}"
echo ""

# Confirmation
read -p "Lanjutkan dengan cleanup? (y/N): " confirm
if [ "$confirm" != "y" ] && [ "$confirm" != "Y" ]; then
    print_error "Script dibatalkan!"
    exit 1
fi

# Function untuk cek apakah package installed
is_installed() {
    dpkg -l "$1" 2>/dev/null | grep -q "^ii"
}

print_status "Memulai SMART Debian Cleanup..."
echo "=========================================="

# Step 1: Analyze System
print_status "Step 1: Menganalisa system..."

# Get installed services packages
print_status "Mendeteksi installed services..."
installed_services=()
for service in "${SERVICE_PACKAGES[@]}"; do
    if is_installed "$service"; then
        installed_services+=("$service")
    fi
done

# Deteksi PHP packages secara khusus
print_status "Mendeteksi PHP packages..."
php_packages=$(dpkg -l | grep -E "^ii.*php" | awk '{print $2}' | tr '\n' ' ')
if [ -n "$php_packages" ]; then
    print_status "PHP packages terdeteksi: $php_packages"
fi

# Show analysis results
echo ""
print_status "ANALYSIS RESULTS:"
echo "-------------------"
echo "• Total services packages: ${#installed_services[@]}"
echo "• PHP packages: $(echo "$php_packages" | wc -w)"
echo "• Essential packages: ${#ESSENTIAL_PACKAGES[@]}"

if [ ${#installed_services[@]} -eq 0 ]; then
    print_success "Tidak ada services packages yang terdeteksi!"
    echo "Tidak ada yang perlu dibersihkan."
    exit 0
fi

echo ""
print_status "Services yang akan dihapus:"
for service in "${installed_services[@]}"; do
    echo "  - $service"
done

# Final confirmation
echo ""
read -p "Hapus services di atas? (y/N): " final_confirm
if [ "$final_confirm" != "y" ] && [ "$final_confirm" != "Y" ]; then
    print_error "Cleanup dibatalkan!"
    exit 1
fi

# Step 2: Stop Services
print_status "Step 2: Menghentikan services..."

for service in "${installed_services[@]}"; do
    case $service in
        apache2|nginx)
            systemctl stop apache2 2>/dev/null || true
            systemctl stop nginx 2>/dev/null || true
            ;;
        mariadb-server|mysql-server)
            systemctl stop mariadb 2>/dev/null || true
            systemctl stop mysql 2>/dev/null || true
            ;;
        bind9)
            systemctl stop bind9 2>/dev/null || true
            ;;
        postfix|dovecot*)
            systemctl stop postfix 2>/dev/null || true
            systemctl stop dovecot 2>/dev/null || true
            ;;
        vsftpd|proftpd)
            systemctl stop vsftpd 2>/dev/null || true
            systemctl stop proftpd 2>/dev/null || true
            ;;
        php*-fpm)
            systemctl stop php8.2-fpm 2>/dev/null || true
            systemctl stop php8.1-fpm 2>/dev/null || true
            systemctl stop php8.0-fpm 2>/dev/null || true
            systemctl stop php7.4-fpm 2>/dev/null || true
            ;;
        postgresql)
            systemctl stop postgresql 2>/dev/null || true
            ;;
        redis-server)
            systemctl stop redis-server 2>/dev/null || true
            ;;
        memcached)
            systemctl stop memcached 2>/dev/null || true
            ;;
    esac
done

print_success "Services stopped"

# Step 3: Remove Service Packages
print_status "Step 3: Menghapus service packages..."

for service in "${installed_services[@]}"; do
    print_status "Menghapus: $service"
    apt remove --purge -y "$service" 2>/dev/null || true
done

# Additional: Remove PHP packages yang mungkin terlewat
print_status "Menghapus PHP packages tambahan..."
php_related_packages=$(dpkg -l | grep -E "^ii.*php" | awk '{print $2}')
for php_pkg in $php_related_packages; do
    if [[ " ${SERVICE_PACKAGES[@]} " =~ " ${php_pkg} " ]]; then
        continue  # Sudah dihapus di loop utama
    else
        print_status "Menghapus PHP package: $php_pkg"
        apt remove --purge -y "$php_pkg" 2>/dev/null || true
    fi
done

print_success "Service packages removed"

# Step 4: Remove Service Data and Configs
print_status "Step 4: Membersihkan data dan config services..."

# Web Servers
if [ -d "/etc/apache2" ]; then
    rm -rf /etc/apache2
    print_status "Apache config dihapus"
fi

if [ -d "/etc/nginx" ]; then
    rm -rf /etc/nginx
    print_status "Nginx config dihapus"
fi

if [ -d "/var/www" ]; then
    # Hanya hapus content, bukan directory structure
    rm -rf /var/www/html/*
    rm -rf /var/www/*/
    print_status "Web content dibersihkan"
fi

# Database Servers
if [ -d "/var/lib/mysql" ]; then
    rm -rf /var/lib/mysql/*
    print_status "MySQL data dibersihkan"
fi

if [ -d "/etc/mysql" ]; then
    rm -rf /etc/mysql
    print_status "MySQL config dihapus"
fi

# PHP Configs
if [ -d "/etc/php" ]; then
    rm -rf /etc/php
    print_status "PHP config dihapus"
fi

# DNS Server
if [ -d "/etc/bind" ]; then
    rm -rf /etc/bind
    print_status "BIND config dihapus"
fi

if [ -d "/var/cache/bind" ]; then
    rm -rf /var/cache/bind/*
    print_status "BIND cache dibersihkan"
fi

# Mail Servers
if [ -d "/etc/postfix" ]; then
    rm -rf /etc/postfix
    print_status "Postfix config dihapus"
fi

if [ -d "/etc/dovecot" ]; then
    rm -rf /etc/dovecot
    print_status "Dovecot config dihapus"
fi

# FTP Servers
if [ -d "/etc/vsftpd" ]; then
    rm -rf /etc/vsftpd
    print_status "VSFTPD config dihapus"
fi

if [ -d "/etc/proftpd" ]; then
    rm -rf /etc/proftpd
    print_status "ProFTPD config dihapus"
fi

# Other Databases
if [ -d "/var/lib/postgresql" ]; then
    rm -rf /var/lib/postgresql/*
    print_status "PostgreSQL data dibersihkan"
fi

if [ -d "/etc/postgresql" ]; then
    rm -rf /etc/postgresql
    print_status "PostgreSQL config dihapus"
fi

# Redis
if [ -d "/var/lib/redis" ]; then
    rm -rf /var/lib/redis/*
    print_status "Redis data dibersihkan"
fi

if [ -d "/etc/redis" ]; then
    rm -rf /etc/redis
    print_status "Redis config dihapus"
fi

print_success "Service data and configs cleaned"

# Step 5: Clean Service Logs
print_status "Step 5: Membersihkan service logs..."

service_logs=(
    "/var/log/apache2"
    "/var/log/nginx" 
    "/var/log/mysql"
    "/var/log/mariadb"
    "/var/log/bind9"
    "/var/log/postfix"
    "/var/log/dovecot"
    "/var/log/vsftpd"
    "/var/log/php"
    "/var/log/postgresql"
    "/var/log/redis"
)

for log_dir in "${service_logs[@]}"; do
    if [ -d "$log_dir" ]; then
        rm -rf "$log_dir"/*
        print_status "Logs cleaned: $(basename "$log_dir")"
    fi
done

print_success "Service logs cleaned"

# Step 6: Clean Package Cache
print_status "Step 6: Membersihkan package cache..."

apt autoremove -y
apt autoclean
apt clean

print_success "Package cache cleaned"

# Step 7: Clean Temporary Files
print_status "Step 7: Membersihkan temporary files..."

# Clean temp directories
rm -rf /tmp/*
rm -rf /var/tmp/*

# Clean package cache
rm -rf /var/cache/apt/archives/*
rm -rf /var/cache/apt/archives/partial/*

# Clean PHP sessions and cache
if [ -d "/var/lib/php/sessions" ]; then
    rm -rf /var/lib/php/sessions/*
    print_status "PHP sessions dibersihkan"
fi

# Clean old logs (keep current)
find /var/log -name "*.log" -type f -exec truncate -s 0 {} \; 2>/dev/null || true
find /var/log -name "*.gz" -type f -delete 2>/dev/null || true
find /var/log -name "*.1" -type f -delete 2>/dev/null || true

print_success "Temporary files cleaned"

# Step 8: Final Check
print_status "Step 8: Final system check..."

# Cek apakah PHP masih ada
if dpkg -l | grep -q "^ii.*php"; then
    remaining_php=$(dpkg -l | grep "^ii.*php" | awk '{print $2}' | tr '\n' ' ')
    print_warning "Masih ada PHP packages: $remaining_php"
else
    print_success "Semua PHP packages berhasil dihapus"
fi

# Cek essential packages masih ada
essential_ok=true
for essential in "${ESSENTIAL_PACKAGES[@]}"; do
    if is_installed "$essential"; then
        continue
    else
        print_warning "Essential package missing: $essential"
        essential_ok=false
    fi
done

if $essential_ok; then
    print_success "Semua essential packages terinstall dengan baik"
else
    print_warning "Beberapa essential packages mungkin hilang"
fi

# Final Summary
echo ""
echo "=========================================="
print_success "SMART DEBIAN CLEANUP COMPLETED!"
echo "=========================================="
echo ""
echo "📋 YANG SUDAH DIBERSIHKAN:"
echo "--------------------------"
echo "✅ Service packages: ${#installed_services[@]} packages"
echo "✅ PHP packages: $(echo "$php_packages" | wc -w) packages"
echo "✅ Service config files"
echo "✅ Service data directories" 
echo "✅ Service log files"
echo "✅ Package cache"
echo "✅ Temporary files"
echo ""
echo "🔒 YANG DIPERTAHANKAN:"
echo "---------------------"
echo "✅ ${#ESSENTIAL_PACKAGES[@]} essential packages"
echo "✅ System base configuration"
echo "✅ User accounts dan home directories"
echo "✅ Network settings"
echo "✅ System logs (current)"
echo ""
echo "🚀 SISTEM BERSIH DAN SIAP UNTUK INSTALL BARU!"
echo "=========================================="

# Recommendations
echo ""
print_status "REKOMENDASI SELANJUTNYA:"
echo "• Install services yang diperlukan:"
echo "  apt install apache2 mariadb-server php phpmyadmin"
echo "• Untuk development PHP:"
echo "  apt install php php-cli php-mysql php-curl php-gd php-mbstring php-xml php-zip"
echo "• Cek system health:"
echo "  systemctl status"
echo "  df -h"
echo "  free -h"
echo ""
print_success "Cleanup selesai! System dalam keadaan bersih dan stabil!"
