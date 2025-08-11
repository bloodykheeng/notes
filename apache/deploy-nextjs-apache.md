Got it — here’s your updated **Next.js + Apache deployment notes** with **nvm for Node installation** included so you’re not stuck with outdated Node from Ubuntu’s apt repo.

---

# **Deploying Next.js on Apache (AWS EC2 – Ubuntu)**

---

## **1. Preparing the Server**

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install apache2 curl git -y
```

Set correct folder permissions (fixes `npm install` permission errors):

```bash
sudo chown -R ubuntu:ubuntu /var/www/html/anime_vault-main
```

---

## **2. Install Node.js with NVM (Recommended)**

Install **nvm**:

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc  # reload shell
```

Verify:

```bash
nvm --version
```

Install Node LTS (or a specific version):

```bash
nvm install --lts
# or
nvm install 20
```

Set default Node version:

```bash
nvm alias default 20
```

Check:

```bash
node -v
npm -v
```

---

## **3. Fixing npm Install Issues**

### **A) Signature / Lock File Issues**

📎 [GitHub Discussion](https://github.com/vercel/next.js/discussions/48192)

```bash
rm -f package-lock.json
rm -rf node_modules .next
npm install
npm run build
```

### **B) npm Killed Due to Low Memory (Swap Space)**

📎 [StackOverflow Solution](https://stackoverflow.com/questions/38127667/npm-install-ends-with-killed)

```bash
sudo /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=1024
sudo /sbin/mkswap /var/swap.1
sudo /sbin/swapon /var/swap.1
```

---

## **4. Apache Reverse Proxy for Next.js**

Enable necessary Apache modules:

```bash
sudo a2enmod proxy proxy_http rewrite headers
```

Create virtual host config:

```bash
sudo nano /etc/apache2/sites-available/cfp.conf
```

Paste:

```apache
<VirtualHost *:80>
    ServerName 213.52.130.213
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/cfp

    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/

    ErrorLog ${APACHE_LOG_DIR}/hibreton-frontend-error.log
    CustomLog ${APACHE_LOG_DIR}/hibreton-frontend-access.log combined
</VirtualHost>
```

Enable the site & disable default:

```bash
sudo a2ensite cfp.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

---

## **5. Running Next.js with PM2**

Install PM2 globally:

```bash
npm install pm2 -g
```

Run Next.js app:

```bash
cd /var/www/cfp
pm2 start npm --name "cfp-nextjs" -- start
pm2 startup
pm2 save
```

Manage app:

```bash
pm2 list
pm2 restart cfp-nextjs
pm2 delete cfp-nextjs
```

---

## **6. Laravel Backend on Port 8000 (Optional)**

Allow Apache to listen on port 8000:

```bash
sudo apt install net-tools
sudo nano /etc/apache2/ports.conf
# Add:
Listen 8000
sudo systemctl restart apache2
```

Check if port 8000 is in use:

```bash
sudo netstat -plnt | grep 8000
```

Create Laravel site config:

```bash
sudo nano /etc/apache2/sites-available/oagapibeta.conf
```

```apache
<VirtualHost *:8000>
    ServerName 213.52.130.213
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/oagapibeta/public

    <Directory /var/www/oagapibeta/public>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/demo-error.log
    CustomLog ${APACHE_LOG_DIR}/demo-access.log combined
</VirtualHost>
```

Enable & reload Apache:

```bash
sudo a2ensite oagapibeta.conf
sudo systemctl reload apache2
```

---

## **7. UFW Firewall Fix for SSH**

Check firewall:

```bash
sudo ufw status
```

Allow SSH if missing:

```bash
sudo ufw allow 22/tcp
sudo ufw reload
sudo ufw status
```

---

## **References**

* [Mastering Next.js Deployment on EC2 – dev.to](https://dev.to/techwithhari/mastering-nextjs-deployment-on-ec2-direct-vs-apache-setup-1h3i)
* [Next.js EC2 Apache Gist](https://gist.github.com/makaubenson/35c4c5f0e3c39d09d6bc33f8ac0bd6ec)
* [Linode: Hosting a Website on Ubuntu](https://www.linode.com/docs/guides/hosting-a-website-ubuntu-18-04/#configure-name-based-virtual-hosts)

---

Do you want me to also **add the steps for starting from a fresh AWS EC2 instance** so this becomes a true *zero-to-live* Next.js + Apache deployment manual? That way someone could follow it from scratch without guessing any setup parts.
