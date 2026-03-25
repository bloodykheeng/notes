````md id="fullnextcicd001"
# Next.js CI/CD Setup with VPS Deployment (GitHub Actions + SSH)

## 🚀 Overview

This guide walks you through:

- Creating a Next.js app
- Setting up CI/CD with GitHub Actions
- Testing locally with `act`
- Deploying automatically to a VPS using SSH
- Managing secrets securely

---

## 🧱 1. Create a Next.js App

```bash
npx create-next-app@latest my-app
cd my-app
npm install
````

Run locally:

```bash
npm run dev
```

---

## 📁 2. Setup GitHub Actions

Create the workflow file:

```bash
mkdir -p .github/workflows
touch .github/workflows/ci.yml
```

---

## ⚙️ 3. Add CI/CD Pipeline

Edit `.github/workflows/ci.yml`:

```yaml
# ----------------------------------------
# This workflow:
# 1. Runs on every push to main
# 2. Installs dependencies
# 3. Builds the Next.js app
# 4. SSH into VPS
# 5. Pulls latest code
# 6. Rebuilds and restarts app using PM2
# ----------------------------------------

name: Next.js CI/CD

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Build application
        run: npm run build

      - name: Deploy to VPS via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd my-app
            git pull origin main
            npm install
            npm run build
            pm2 restart nextjs-app
```

---

## 🧪 4. Test Locally with `act`

Install:

```bash
brew install act
# OR
choco install act-cli
```

Run:

```bash
act
```
sometimes u hav to run `docker login`
`act -l` - lists all available actions
`act -j <jobname>` - runs a specific job
`act` -  runs all jobs

If you get image issues try:

```bash
act -P ubuntu-latest=nektos/act-environments-ubuntu:18.04
```

---

## 🖥️ 5. Setup Your VPS (One-Time)

SSH into your server:

```bash
ssh ubuntu@your-server-ip
```

Install Node & PM2:

```bash
sudo apt update
sudo apt install -y nodejs npm
npm install -g pm2
```

Clone your project:

```bash
git clone https://github.com/your-username/your-repo.git
cd my-app
npm install
npm run build
```

Start app:

```bash
pm2 start npm --name "nextjs-app" -- start
pm2 save
pm2 startup
```

---

## 🔐 6. Generate SSH Keys

```bash
ssh-keygen -t rsa -b 4096 -C "github-actions"
```

Files created:

* `~/.ssh/id_rsa` (PRIVATE KEY)
* `~/.ssh/id_rsa.pub` (PUBLIC KEY)

---

## 📤 7. Add Public Key to VPS

```bash
ssh-copy-id ubuntu@your-server-ip
```

OR:

```bash
cat ~/.ssh/id_rsa.pub
```

Paste into:

```bash
nano ~/.ssh/authorized_keys
```

---

## 📥 8. Add Secrets to GitHub

Go to:

**GitHub → Settings → Secrets and variables → Actions**

Add:

### VPS_HOST

```
your-server-ip
```

### VPS_USER

```
ubuntu
```

### VPS_SSH_KEY

```
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
```

---

## ✅ 9. Test SSH Access

```bash
ssh ubuntu@your-server-ip
```

Should login without password ✅

---

## 🔄 10. Auto Deployment Flow

Every push:

```bash
git push origin main
```

Triggers:

1. GitHub Actions runs
2. App builds
3. SSH connects to VPS
4. Code updates (`git pull`)
5. App rebuilds
6. PM2 restarts app

---

## 🌐 11. (Optional) Setup Nginx

```bash
sudo apt install nginx
```

Basic config:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
    }
}
```

Restart:

```bash
sudo systemctl restart nginx
```

---

## 🔒 12. (Optional) Enable HTTPS

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx
```

---

## ⚠️ Common Errors

### Permission Denied (SSH)

```bash
chmod 600 ~/.ssh/id_rsa
```

---

### PM2 Not Found

```bash
npm install -g pm2
```

---

### App Not Updating

```bash
git pull origin main
```

---

## 💡 Best Practices

* Use `npm ci` in CI
* Never expose private keys
* Use environment variables
* Prefer `pm2 reload` for zero downtime

---

## 🎯 Conclusion

You now have:

* ✅ Next.js app
* ✅ CI/CD pipeline
* ✅ VPS auto deployment
* ✅ Secure SSH setup

🚀 Your app now auto-deploys on every push!

```
