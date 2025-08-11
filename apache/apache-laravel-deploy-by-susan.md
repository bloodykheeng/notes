Here’s a **well-organized Markdown guide** for deploying Laravel on Apache, based on your provided resources.

---

# How to Deploy Laravel on Apache (Ubuntu) by susan

## 📋 Prerequisites

* Ubuntu server with Apache installed.
* PHP installed (with required extensions for Laravel).
* MySQL or another supported database installed.
* Git installed.
* Composer installed (or ready to be installed).
* Laravel project in a Git repository.
* Sudo/root privileges.

---

## 1️⃣ Install PHP & Required Extensions

First, add the official PHP package repository:

```bash
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```

Install PHP extensions needed for Laravel:

```bash
sudo apt install php8.2-curl php8.2-dom php8.2-mbstring php8.2-xml php8.2-mysql zip unzip
```

---

## 2️⃣ Enable Apache Rewrite Module

Laravel needs URL rewriting for its routing system:

```bash
sudo a2enmod rewrite
sudo service apache2 restart
```

---

## 3️⃣ Install Composer

If Composer is not already installed:

```bash
cd /usr/bin
curl -sS https://getcomposer.org/installer | sudo php
sudo mv composer.phar composer
composer
```

---

## 4️⃣ Clone Your Laravel Project

Go to your web directory and clone your repository:

```bash
cd /var/www/
git clone git@github.com:username/project.git
cd project
```

---

## 5️⃣ Install Dependencies

**Production:**

```bash
composer install --optimize-autoloader --no-dev
```

**Development:**

```bash
composer update
```

---

## 6️⃣ Configure `.env`

Copy the example environment file and configure it:

```bash
cp .env.example .env
php artisan key:generate
```

Update `.env` with:

* Database credentials
* App URL
* Any other environment-specific settings

---

## 7️⃣ Set Permissions

Make `storage` and `bootstrap/cache` writable by Apache’s user:

```bash
ps aux | grep "apache" | awk '{print $1}' | grep -v root | head -n 1
# Usually it's www-data
sudo chown -R www-data storage
sudo chown -R www-data bootstrap/cache
```

---

## 8️⃣ Configure Apache Virtual Host

Create a config file:

```bash
sudo nano /etc/apache2/sites-available/project.conf
```

Example configuration:

```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/project/public

    <Directory /var/www/project/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/project-error.log
    CustomLog ${APACHE_LOG_DIR}/project-access.log combined
</VirtualHost>
```

---

## 9️⃣ Enable Site & Restart Apache

```bash
sudo a2ensite project.conf
apache2ctl -t   # Check for syntax errors
sudo systemctl restart apache2
```

---

## 🔟 Test in Browser

Visit:

```
http://example.com
```

If it fails:

* Check Apache error logs:

```bash
tail -f /var/log/apache2/project-error.log
```

* Review `.env` settings.
* Check file permissions.

---

## ✅ Quick Troubleshooting Tips

* Make sure `mod_rewrite` is enabled.
* Ensure `public` is the document root.
* Check that your `.env` file has the correct `APP_KEY` and DB settings.
* Use `php artisan config:clear` and `php artisan cache:clear` if configs seem outdated.

---

## 📚 References

* [Susan Buck’s Deployment Guide](https://codewithsusan.com/notes/deploy-laravel-on-apache)
* [Laravel Official Docs](https://laravel.com/docs/deployment)

---

If you want, I can also make you a **more advanced version** with SSL (Let’s Encrypt) and `.htaccess` hardening so the Laravel app is secure in production.
