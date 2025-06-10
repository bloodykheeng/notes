# Deploy Laravel on Ubuntu with Nginx, MySQL, and PHPMyAdmin

## Step 1: Setting Up the Firewall

Before we begin, configure the firewall to allow essential services:

```bash
sudo ufw app list  # Shows available applications
sudo ufw allow OpenSSH  # Allow SSH access
sudo ufw enable  # Enable the firewall
sudo ufw status  # Check firewall status
```

---

## Step 2: Installing Nginx, MySQL, and PHP

First, install the required dependencies:

```bash
sudo apt update && sudo apt install -y nginx mysql-server php php-fpm php-mbstring php-xml php-bcmath php-curl zip unzip
```

### Verify PHP Version

Ensure your server meets Laravel's PHP requirements:

```bash
php --version  # Check the installed PHP version
```

For Laravel's latest requirements, visit:
[Laravel Deployment Requirements](https://laravel.com/docs/11.x/deployment)

To check PHP used by Nginx:

```bash
cd /var/www/html
sudo nano info.php
```

Add the following:

```php
<?php
phpinfo();
```

Save and visit `http://your-domain.com/info.php` to check the PHP version in use.

---

## Step 3: Install PHP Extensions

Server Requirements
The Laravel framework has a few system requirements. You should ensure that your web server has the following minimum PHP version and extensions:

PHP >= 8.2
Ctype PHP Extension
cURL PHP Extension
DOM PHP Extension
Fileinfo PHP Extension
Filter PHP Extension
Hash PHP Extension
Mbstring PHP Extension
OpenSSL PHP Extension
PCRE PHP Extension
PDO PHP Extension
Session PHP Extension
Tokenizer PHP Extension
XML PHP Extension

To ensure all required PHP extensions are installed:

```bash
sudo add-apt-repository ppa:ondrej/php -y
sudo apt-get update
sudo apt-get install -y php8.2-xml php8.2-dom php8.2-mysql zip unzip
```

or u can use

```bash
apt install nginx mysql-server php php-fpm php-mbstring php-xml php-bcmath php-curl zip unzip php-zip php8.3-gd -y
```

-y will answer yes to all
with this method u dont specify the version they will automaticlly be set

Confirm the installed extensions:

```bash
php -m
```

---

## Step 4: Install Composer

Laravel uses Composer for dependency management:

```bash
cd /usr/bin
curl -sS https://getcomposer.org/installer | sudo php
sudo mv composer.phar composer
```

Verify installation:

```bash
composer
```

---

## Step 5: Clone Laravel Project

Navigate to the web root directory and clone your Laravel project:

```bash
cd /var/www/
git clone your-repository-url demo
cd demo
```

Install dependencies:

if working on a production server , use the following command so any development specific dependacies are excluded and the version number of your dependacies match whatever was used in development and written to composer.lock

```bash
composer install --optimize-autoloader --no-dev  # For production
```

if working on a development server use the following command so all dependencies are included and you get the latest versions within your version constraits listed in your composer.json file

```bash
composer update  # For development
```

Copy environment settings and generate an app key:

```bash
cp .env.example .env
php artisan key:generate
```

set
APP_ENV=production
APP_DEBUG=false

---

## Step 6: Set Permissions

Set the correct permissions for Laravel:

There are two directories within a Laravel application that need to be writable by the server : storage and bootstrap/cache. within these directories, the server will write application-specific files such as cache info, session data, error logs etc.

To allow this to happen you need to update the permissions of storage and bootstrap/cache so it is owned by the system user your web server is running as
run the following command to see which user your nginx web server runs as

```bash
ps aux | grep "nginx: worker process" | awk '{print $1}' | grep -v root
```

on my server the above command outputs the user www-data so i will update storage and bootstrap/cache to be owned by www-data with the following commands

```bash
chown -R www-data storage
chown -R www-data bootstrap/cache
```

#### 🗂 `public/`

This is the web-accessible root. To allow uploads or public file creation:

```bash
sudo chown -R www-data:www-data /var/www/ppda_laravel_api/public
sudo chmod -R 775 /var/www/ppda_laravel_api/public
```

#### 🗂 `public/file_share_attachments/`

If you have a subfolder specifically for file sharing:

```bash
sudo chown -R www-data:www-data /var/www/ppda_laravel_api/public/file_share_attachments
sudo chmod -R 775 /var/www/ppda_laravel_api/public/file_share_attachments
```

---

### 💡 Notes:

- `chown` changes the owner to `www-data`, which allows the web server to write to those directories.
- `chmod 775` gives:

  - Owner (www-data): read, write, execute
  - Group: read, write, execute
  - Others: read and execute only

---

## Step 7: Configure Nginx for Laravel

Create an Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/demo
```

within the content update the following
server_name - should point to the domain youre using for this application
root - should point to the path to your Laravel applications public
the refernce to php8.2-fm.sock should match whatever version of php your server is running

go here to confirm the php ur using

```bash
cd /etc/php
cd /var/run/php
```

ensure that the php version of fastcgi_pass matches the php version of ur nginx server

```bash
location ~ \.php${
fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
}
```

Add the following:

```nginx
server {
    listen 80;
    server_name demo.yourdomain.com;
    root /var/www/demo/public;

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Enable the site and restart Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/demo /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## Step 8: Install and Configure PHPMyAdmin

```bash
sudo apt install -y phpmyadmin
```

when prompt comes up to choose webserver
ie apache2 , lighttpd

dont select any
just press tab and ok
then the rest say yes and enter password

then it will install but its not accessible in the browser
we can solve it by creating symbolic link

sudo ln -s /usr/share/phpMyAdmin/ /var/www/html

this will clone folder phpMyAdmin into html then we can point to it using our confs either the default
conf or any of ur choice

Nb its not really nesssary to create the symbolic link u can just forgo it and u add the conf
with this as ur root root /usr/share/;
not this root /var/www/html/;

If PHPMyAdmin is not accessible, configure Nginx:

```bash
sudo nano /etc/nginx/sites-available/default
```

Add:

```nginx
location /phpmyadmin {
    root /usr/share/;
    index index.php;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

but if u dont have sites default u can use this in ur conf
/etc/nginx/sites-available/ppda_angular_front
ie

```nginx

server {
    listen 80;
    server_name 127.0.0.1;  # Change this to your actual domain or IP if needed

    root /var/www/ppda_angular_front/;
    index index.html index.php;

    location / {
        try_files $uri /index.html;
    }

    # Pass PHP scripts to FastCGI server
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock; # Ensure PHP-FPM is running
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }

    # phpMyAdmin configuration
    location /phpmyadmin {
        root /var/www/html/;
        index index.php;

        location ~ \.php$ {
            include snippets/fastcgi-php.conf;
            fastcgi_pass unix:/run/php/php8.3-fpm.sock;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include fastcgi_params;
        }
    }
}


```

Restart Nginx:

```bash
sudo systemctl restart nginx
```

---

### 👤 Create a New MySQL User

To create a new user (e.g., `mingo`):

```sql
CREATE USER 'mingo'@'localhost' IDENTIFIED BY 'Tr3y@1234567';
```

---

### ✅ Grant All Privileges to the New User

Give full privileges to `mingo` on all databases and tables:

```sql
GRANT ALL PRIVILEGES ON *.* TO 'mingo'@'localhost';
```

Apply those privileges:

```sql
FLUSH PRIVILEGES;
```

---

this step is not required
If you still face MySQL login issues:
enter mysql

```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password';
FLUSH PRIVILEGES;
```

for help
https://youtu.be/crfAj9yay5w?si=7rqge3yzXdXJBsPK

---

## Final Step: Test the Application

Visit your domain:

```
http://yourdomain.com
```

For PHPMyAdmin:

```
http://yourdomain.com/phpmyadmin
```

Your Laravel application should now be fully deployed on Nginx with MySQL and PHPMyAdmin!

For further troubleshooting, check:

```bash
sudo journalctl -xe
sudo tail -f /var/log/nginx/error.log
```
