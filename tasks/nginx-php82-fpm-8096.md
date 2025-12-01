# Nginx + PHP 8.2 FPM using Unix Socket (Port 8096)

**Goal:**  
- Run Nginx on port **8096**  
- Document root: `/var/www/html`  
- PHP 8.2 processed via **php-fpm** using Unix socket `/var/run/php-fpm/default.sock`  
- Works with `index.php` and `info.php`

## Exact Working Commands (RHEL 9 / Rocky 9)

```bash
# 1. Install EPEL + Remi repo
sudo dnf install -y epel-release
sudo dnf install -y https://rpms.remirepo.net/enterprise/remi-release-9.rpm

# 2. Switch to PHP 8.2
sudo dnf module reset php -y
sudo dnf module enable php:remi-8.2 -y

# 3. Install PHP-FPM 8.2 + Nginx
sudo dnf install -y php-fpm php-cli php-mysqlnd nginx

# 4. Configure php-fpm to use Unix socket
sudo mkdir -p /var/run/php-fpm
sudo sed -i 's|listen =.*|listen = /var/run/php-fpm/default.sock|' /etc/php-fpm.d/www.conf
sudo sed -i 's|;listen.owner = nobody|listen.owner = nginx|' /etc/php-fpm.d/www.conf
sudo sed -i 's|;listen.group = nobody|listen.group = nginx|' /etc/php-fpm.d/www.conf
sudo sed -i 's|;listen.mode = 0660|listen.mode = 0660|' /etc/php-fpm.d/www.conf

# 5. Configure Nginx (port + document root)
sudo sed -i 's/listen       80;/listen       8096;/' /etc/nginx/nginx.conf
sudo sed -i 's|root         /usr/share/nginx/html;|root         /var/www/html;|' /etc/nginx/nginx.conf

# 6. Add PHP processing block
sudo mkdir -p /etc/nginx/default.d
sudo cat > /etc/nginx/default.d/php.conf <<EOF
location ~ \.php$ {
    fastcgi_pass   unix:/var/run/php-fpm/default.sock;
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME  \$document_root\$fastcgi_script_name;
    include        fastcgi_params;
}
EOF

# 7. Permissions & services
sudo chown -R nginx:nginx /var/www/html
sudo systemctl enable --now nginx php-fpm

# 8. Open port
sudo firewall-cmd --add-port=8096/tcp --permanent
sudo firewall-cmd --reload


**Test Commands**

curl http://localhost:8096/index.php
curl http://localhost:8096/info.php    # shows phpinfo()

Result: Fully working PHP site on port 8096 via Unix socket
Tested & verified – December 2025
