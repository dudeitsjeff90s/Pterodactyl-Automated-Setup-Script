# Pterodactyl-Automated-Setup-Script
This repository contains a script created with the help of AI, using my notes it was able to designed to simplify the installation and setup of the open-source Game Server Management platform called (Pterodactyl)

This is the Script

#!/bin/bash
set -e

echo "=== Pterodactyl Automated Installer ==="

# -------- VARIABLES --------
read -p "Enter domain or server IP: " DOMAIN
read -p "Enter MySQL password for pterodactyl user: " DB_PASS
read -p "Enter admin email for panel: " ADMIN_EMAIL

PANEL_DIR="/var/www/pterodactyl"

# -------- STEP 1: SYSTEM PREP --------
apt update && apt upgrade -y
apt install -y software-properties-common curl apt-transport-https ca-certificates gnupg redis-server mysql-server apache2

# -------- STEP 2: PHP 8.3 --------
add-apt-repository -y ppa:ondrej/php
apt update

apt install -y \
php8.3 php8.3-cli php8.3-common php8.3-mysql php8.3-xml \
php8.3-curl php8.3-gd php8.3-mbstring php8.3-bcmath \
php8.3-zip php8.3-fpm libapache2-mod-php8.3

update-alternatives --set php /usr/bin/php8.3

# -------- STEP 3: COMPOSER --------
curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# -------- STEP 4: DOWNLOAD PANEL --------
mkdir -p $PANEL_DIR
cd $PANEL_DIR

curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
tar -xzvf panel.tar.gz
chmod -R 755 storage bootstrap/cache

cp .env.example .env

COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader

php artisan key:generate --force

APP_KEY=$(grep APP_KEY .env)
echo "==== SAVE THIS APP KEY ===="
echo "$APP_KEY"
echo "==========================="

# -------- STEP 5: DATABASE --------
mysql <<EOF
CREATE DATABASE panel;
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY '$DB_PASS';
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1';
FLUSH PRIVILEGES;
EOF

# -------- STEP 6: ENV SETUP --------
sed -i 's/REDIS_PASSWORD=.*/REDIS_PASSWORD=null/' .env

php artisan p:environment:setup --author="$ADMIN_EMAIL" --url="http://$DOMAIN"
php artisan p:environment:database --host=127.0.0.1 --port=3306 --database=panel --username=pterodactyl --password="$DB_PASS"

# -------- STEP 7: MIGRATE --------
php artisan migrate --seed --force

# -------- STEP 8: CREATE USER --------
php artisan p:user:make

# -------- STEP 9: PERMISSIONS --------
chown -R www-data:www-data $PANEL_DIR

# -------- STEP 10: CRON --------
(crontab -l 2>/dev/null; echo "* * * * * php $PANEL_DIR/artisan schedule:run >> /dev/null 2>&1") | crontab -

# -------- QUEUE WORKER --------
cat > /etc/systemd/system/pteroq.service <<EOF
[Unit]
Description=Pterodactyl Queue Worker
After=redis-server.service

[Service]
User=www-data
Group=www-data
Restart=always
ExecStart=/usr/bin/php $PANEL_DIR/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now redis-server pteroq.service

# -------- STEP 11: APACHE --------
a2dissite 000-default.conf

cat > /etc/apache2/sites-available/pterodactyl.conf <<EOF
<VirtualHost *:80>
    ServerName $DOMAIN
    DocumentRoot "$PANEL_DIR/public"

    AllowEncodedSlashes On

    php_value upload_max_filesize 100M
    php_value post_max_size 100M

    <Directory "$PANEL_DIR/public">
        AllowOverride all
        Require all granted
    </Directory>
</VirtualHost>
EOF

ln -s /etc/apache2/sites-available/pterodactyl.conf /etc/apache2/sites-enabled/pterodactyl.conf
a2enmod rewrite
systemctl restart apache2

# -------- STEP 12: DOCKER --------
curl -sSL https://get.docker.com/ | CHANNEL=stable bash
systemctl enable --now docker

# -------- STEP 13: WINGS --------
mkdir -p /etc/pterodactyl

ARCH=$(uname -m)
if [[ "$ARCH" == "x86_64" ]]; then
  WINGS_ARCH="amd64"
else
  WINGS_ARCH="arm64"
fi

curl -L -o /usr/local/bin/wings https://github.com/pterodactyl/wings/releases/latest/download/wings_linux_$WINGS_ARCH
chmod +x /usr/local/bin/wings

cat > /etc/systemd/system/wings.service <<EOF
[Unit]
Description=Pterodactyl Wings Daemon
After=docker.service
Requires=docker.service

[Service]
User=root
WorkingDirectory=/etc/pterodactyl
ExecStart=/usr/local/bin/wings
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload

echo "=== INSTALL COMPLETE ==="
echo "Panel URL: http://$DOMAIN"
echo "IMPORTANT: Add config.yml to /etc/pterodactyl before starting Wings"
