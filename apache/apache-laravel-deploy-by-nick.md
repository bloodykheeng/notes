---

# 🚀 Setting up Laravel + MySQL + Apache on AWS Ubuntu (Development Environment) by Nick

This guide covers setting up a Laravel development environment with **MySQL** on an **AWS EC2 Ubuntu instance**, connecting via **VSCode Remote SSH**, and configuring **FileZilla** for file transfer.

---

## 1️⃣ Update Packages

```bash
sudo apt update
```

---

## 2️⃣ Install Apache

```bash
sudo apt install apache2
```

---

## 3️⃣ Install PHP & Extensions

```bash
sudo apt install php libapache2-mod-php php-mbstring php-xmlrpc php-soap php-gd php-xml php-cli php-zip php-bcmath php-tokenizer php-json php-pear
```

---

## 4️⃣ Install MySQL Server

```bash
sudo apt install mysql-server
```

---

## 5️⃣ Install phpMyAdmin

```bash
sudo apt install phpmyadmin
# When prompted, press SPACE to select Apache2, then ENTER
```

**If you ever need to remove phpMyAdmin:**

```bash
sudo apt-get purge phpmyadmin
```

When reinstalling and prompted again, press **No** if you don’t want it to auto-configure.

---

## 6️⃣ Configure MySQL Root User

```bash
sudo mysql
USE mysql;
UPDATE user SET plugin='mysql_native_password' WHERE User='root';
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'forge@123';
```

Test login:

```bash
mysql -u root -p
```

Create a database:

```sql
CREATE DATABASE nickexample;
```

Access phpMyAdmin:

```
http://<your-public-ip>/phpmyadmin/
```

---

## 7️⃣ Install Composer

```bash
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
sudo chmod +x /usr/local/bin/composer
composer
```

---

## 8️⃣ Install Laravel App

Move into the Apache web root:

```bash
cd /var/www/html
```

Set ownership and permissions:

```bash
whoami  # Usually 'ubuntu'
sudo chown -R www-data:ubuntu /var/www/html/
sudo find /var/www/html/ -type d -exec chmod 775 {} \;
sudo find /var/www/html/ -type f -exec chmod 664 {} \;
```

> Folders 775 and files 664, not `chmod -R 775`, which puts the executable bit on every file.
> Git tracks that bit, so a project cloned into `/var/www` would show every file as modified
> and `git pull` would later refuse. See
> [../git/git-pull-refuses-after-chmod-file-mode.md](../git/git-pull-refuses-after-chmod-file-mode.md).

Create Laravel project:

```bash
composer create-project --prefer-dist laravel/laravel nickexample
cd nickexample
```

Serve Laravel (development mode):

```bash
php artisan serve --host=0.0.0.0
```

---

## 9️⃣ Connect via VSCode Remote SSH

```bash
ssh ubuntu@<ip-address> -p 22 -i "path/to/laravel-react.pem"
```

Example:

```bash
ssh ubuntu@44.205.93.93 -p 22 -i "C:\Users\BK\Downloads\laravel-react.pem"
```

**Saving files as root in VSCode Remote SSH:**

* Press `Ctrl + Shift + P` → “Save as root” (see [StackOverflow guide](https://stackoverflow.com/questions/56291492/how-to-save-a-file-in-vscode-remote-ssh-with-a-non-root-user-privileges)).

---

## 🔟 Migrate Database Tables

```bash
php artisan migrate
```

---

## 🔑 AWS EC2 – Set a Password for Ubuntu User (Optional)

Switch to root:

```bash
sudo su -
```

Set password:

```bash
passwd ubuntu
```

---

## 📂 Connect with FileZilla

1. **File → Site Manager → New Site**
2. Protocol: **SFTP**
3. Host: `<public-ip>`
4. Port: `22`
5. Login type: **Key file**
6. User: `ubuntu`
7. Key file: Select your `.pem` file (change file filter to show `.pem`)

---

## 📜 Apache Virtual Host Examples

### React App Config

```apache
<VirtualHost 172.31.47.186:80>
    ServerAdmin webmaster@nicedashboard.com
    DocumentRoot /var/www/nicedashboard

    <Directory /var/www/nicedashboard>
      AllowOverride All
      Allow from all
      Options -MultiViews
      Require all granted
    </Directory>
</VirtualHost>
```

### Laravel App Config

```apache
<VirtualHost 172.31.47.187:80>
    ServerAdmin webmaster@niceapi.com
    DocumentRoot /var/www/niceapi/public

    <Directory /var/www/niceapi/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/demo-error.log
    CustomLog ${APACHE_LOG_DIR}/demo-access.log combined
</VirtualHost>
```

Enable and test configs:

```bash
sudo a2ensite react.conf
sudo a2ensite laravel.conf
apache2ctl configtest
sudo service apache2 restart
```

---

## ⚠️ React Routing Issue Fix

If routes like `/component/othercomponent` return 404, ensure:

* Apache config has:

```apache
AllowOverride All
```

* Add `<base href="/">` in your HTML `<head>`.

---

## 📌 Extra File Permissions Command

Allow `whoami` user to write in `/var/www`:

```bash
sudo chown -R www-data:ubuntu /var/www/
sudo find /var/www/ -type d -exec chmod 775 {} \;
sudo find /var/www/ -type f -exec chmod 664 {} \;
```

If a project under `/var/www` already had `chmod -R 775` and `git pull` now refuses, run
`git config core.fileMode false` in it. See
[../git/git-pull-refuses-after-chmod-file-mode.md](../git/git-pull-refuses-after-chmod-file-mode.md).

---

## ✅ Summary

You now have:

* **Apache** + **PHP** + **MySQL** installed.
* **phpMyAdmin** accessible.
* **Composer** installed.
* **Laravel** running in `/var/www/html`.
* **VSCode Remote SSH** & **FileZilla** connections set up.
* **Virtual Hosts** configured for Laravel and React.

---
https://youtu.be/JUV1vcjkgTc?si=se0TUJu_W7FskmRt

Do you want me to also make a **production-ready version** of this AWS Laravel setup with SSL, `.env` optimization, and Apache security hardening? That way it’s safe for public launch.
