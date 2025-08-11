Got it ✅
Here’s your **well-organized Markdown guide** — same style as your Laravel one — for deploying a React app on Apache without needing to add a `.htaccess` file inside the `build` folder.

---

# How to Deploy React on Apache Without `.htaccess` in Build Folder

## 📋 Prerequisites

* Ubuntu server with Apache installed.
* Node.js & npm installed (for building React).
* Git installed.
* React project in a Git repository.
* Sudo/root privileges.

---

## 1️⃣ Install Apache

If Apache is not already installed:

```bash
sudo apt update
sudo apt install apache2
```

---

## 2️⃣ Build Your React App

On your local machine or server, run:

```bash
cd your-react-project
npm install
npm run build
```

This creates a `build` folder containing your production-ready React files.

---

## 3️⃣ Copy Build Files to Server

Upload your `build` folder to your server’s web root.
Example: move it to `/var/www/oagdash`:

```bash
scp -r build/* user@server:/var/www/oagdash/
```

Or if building directly on the server:

```bash
sudo mkdir -p /var/www/oagdash
sudo cp -r build/* /var/www/oagdash/
```

---

## 4️⃣ Create Apache Virtual Host

Open a new config file:

```bash
sudo nano /etc/apache2/sites-available/oagdash.conf
```

Add this configuration (includes React routing rules, so **no `.htaccess` needed**):

```apache
<VirtualHost *:80>
    ServerName cfpdemo.oag.go.ug
    DocumentRoot /var/www/oagdash

    <Directory /var/www/oagdash>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted

        # Redirect 404s to index.html for React routing
        RewriteEngine On
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteCond %{REQUEST_FILENAME} !-d
        RewriteRule ^ /index.html [L]
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/react-frontend-error.log
    CustomLog ${APACHE_LOG_DIR}/react-frontend-access.log combined
</VirtualHost>
```

---

## 5️⃣ Enable the Site & Required Modules

```bash
sudo a2enmod rewrite
sudo a2ensite oagdash.conf
sudo apache2ctl -t   # Check syntax
sudo systemctl restart apache2
```

---

## 6️⃣ Test in Browser

Visit:

```
http://cfpdemo.oag.go.ug
```

If React routing works (even after refreshing deep links like `/about`), the configuration is correct.

---

## ✅ Troubleshooting Tips

* **Blank page?** Make sure the `index.html` file is in `/var/www/oagdash`.
* **404 errors on refresh?** Ensure the rewrite rules are in your Apache config.
* **Permission denied?** Run:

  ```bash
  sudo chown -R www-data:www-data /var/www/oagdash
  sudo chmod -R 755 /var/www/oagdash
  ```
* **Still seeing old files?** Clear your browser cache or hard reload.

---

## 📚 References

* [React Deployment Docs](https://create-react-app.dev/docs/deployment/)
* [Apache mod\_rewrite Guide](https://httpd.apache.org/docs/current/mod/mod_rewrite.html)

---

If you want, I can also make a **version with HTTPS and Let’s Encrypt** so your React app is secure in production.
