# Deploying Next.js App on Nginx (Digital Ocean)

## Prerequisites

- A Digital Ocean server (or any Ubuntu server) with at least **2GB RAM** (recommended for `npm run build`)
- Domain name (optional, but recommended)

---

## Step 1: Upgrade the Server

```bash
sudo apt update
sudo apt upgrade -y
```

---

## Step 2: Install Nginx and Certbot (for SSL)

```bash
sudo apt install nginx certbot python3-certbot-nginx -y
```

### Allow Firewall Access

```bash
sudo ufw allow "Nginx Full"
sudo ufw allow OpenSSH
sudo ufw enable
```

---

## Step 3: Install Node.js and NPM

### Option 1: Install Node.js via NVM (Recommended)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash
exec $SHELL
nvm install --lts
nvm install 20 # Or your preferred version
nvm use 20
nvm alias default 20
```

### Option 2: Install Node.js Directly

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs
```

---

## Step 4: Set Up the Next.js Application

```bash
cd /var/www
mkdir ppda_nextjs_front
cd ppda_nextjs_front
git clone https://github.com/YOUR_GITHUB_REPO.git .
npm install
```

### Add `.env` file and build the project

```bash
nano .env # Add environment variables
npm run build
```

---

## Step 5: Configure Nginx

### Navigate to the sites-available directory

```bash
cd /etc/nginx/sites-available
```

### Create a new Nginx configuration file

```bash
sudo nano ppda_nextjs_front
```

Paste the following configuration:

```nginx
server {
    listen 80;
    server_name YOUR_SERVER_IP_OR_DOMAIN;

    gzip on;
    gzip_proxied any;
    gzip_types application/javascript application/x-javascript text/css text/javascript;
    gzip_comp_level 5;
    gzip_buffers 16 8k;
    gzip_min_length 256;

    location /_next/static/ {
        alias /var/www/ppda_nextjs_front/.next/static/;
        expires 365d;
        access_log off;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Save and exit the file (`CTRL+X`, `Y`, `ENTER`).

### Create a symbolic link for Nginx

```bash
sudo ln -s /etc/nginx/sites-available/ppda_nextjs_front /etc/nginx/sites-enabled/
```

### Restart Nginx and check configuration

```bash
sudo systemctl restart nginx
nginx -t
```

---

## Step 6: Set Up PM2 to Manage the Next.js Application

```bash
cd /var/www/ppda_nextjs_front
npm install -g pm2
pm2 start npm --name ppda_nextjs_front -- start
```

### Optionally start on a different port

```bash
pm2 start npm --name ppda_nextjs_front -- start -- --port=3001
```

### Save PM2 processes

```bash
pm2 save
pm2 startup
```

### Manage PM2 processes

```bash
pm2 restart ppda_nextjs_front
pm2 delete ppda_nextjs_front
```

---

## Step 7: Set Up SSL (Optional but Recommended)

```bash
sudo certbot --nginx -d YOUR_DOMAIN
```

This will automatically configure SSL for your Next.js app.

---

## Final Notes

- Your Next.js application should now be running on Nginx and managed by PM2.
- If deploying multiple Next.js apps, change the `proxy_pass` port in the Nginx configuration and PM2 startup command accordingly.
- Ensure you monitor memory usage, as Next.js builds can consume significant RAM.

---

## References

- [YouTube Guide](https://youtu.be/Dzajd46nyWo?si=_NZJTytay1vwn2r5)
- [Gist](https://gist.github.com/oelbaga/5019647715e68815c602ff05cff2416e)
- [medium guide](https://ilgaz.medium.com/deploy-multiple-next-js-apps-on-ubuntu-with-nginx-e8081c9bb080)
