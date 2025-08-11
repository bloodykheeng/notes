Alright — here’s a **clean, well-structured Markdown guide** for installing and managing a **Let’s Encrypt SSL certificate on Apache (Ubuntu)**, combining the steps from all your resources into one reference.

---

# How to Install & Manage Let’s Encrypt SSL on Apache (Ubuntu)

## 📋 Prerequisites

* Ubuntu server with Apache installed and running.
* A registered domain pointing to your server’s IP.
* Port **80** (HTTP) and **443** (HTTPS) open in the firewall.
* Sudo/root privileges.

---

## 1️⃣ Open Firewall Ports

Allow HTTPS (443) and HTTP (80) traffic:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw reload
sudo ufw status
```

---

## 2️⃣ Install Certbot & Apache Plugin

Certbot is the tool for requesting and managing Let’s Encrypt certificates:

```bash
sudo apt update
sudo apt install certbot python3-certbot-apache
```

Check installation:

```bash
certbot --version
```

---

## 3️⃣ Get an SSL Certificate

Replace `example.com` and `www.example.com` with your actual domain(s):

```bash
sudo certbot --apache -d example.com -d www.example.com
```

**During the process, Certbot will:**

* Ask for an email address (for renewal/security notices).
* Ask if you agree to the Let’s Encrypt Terms of Service.
* Optionally redirect all HTTP traffic to HTTPS.

When done, your site will be accessible at:

```
https://example.com
```

You should see a padlock 🔒 in your browser.

---

## 4️⃣ Test Your SSL

Run an SSL Labs test:

🔗 [https://www.ssllabs.com/](https://www.ssllabs.com/)

---

## 5️⃣ Automatic Renewal

Let’s Encrypt certificates expire every **90 days**, but Certbot sets up automatic renewal via a **systemd timer** or cron job.

Check if the renewal timer is active:

```bash
sudo systemctl status certbot.timer
```

Simulate renewal:

```bash
sudo certbot renew --dry-run
```

---

## 6️⃣ Removing a Certificate

If you need to remove a certificate:

```bash
sudo certbot delete --cert-name yourdomain.com
```

Or list all certificates first:

```bash
sudo certbot certificates
```

---

## 7️⃣ Manual Cleanup After Removal

After deleting a certificate:

1. Remove the `https` site config (usually `/etc/apache2/sites-available/<your-domain>-le-ssl.conf`).
2. Remove `Rewrite` rules in your HTTP config that redirect to HTTPS.
3. Remove SSL certificate lines from the `<VirtualHost *:443>` block:

```apache
SSLCertificateFile /etc/letsencrypt/live/yourdomain.com/fullchain.pem
SSLCertificateKeyFile /etc/letsencrypt/live/yourdomain.com/privkey.pem
Include /etc/letsencrypt/options-ssl-apache.conf
```

4. Restart Apache:

```bash
sudo systemctl restart apache2
```

---

## ✅ Quick Troubleshooting Tips

* **Port blocked?** Make sure your firewall or hosting provider allows ports **80** and **443**.
* **Renewal fails?** Check `/var/log/letsencrypt/letsencrypt.log`.
* **Mixed content warning?** Update your app URLs to use `https://`.

---

## 📚 References

* [Let’s Encrypt Docs](https://letsencrypt.org/getting-started/)
* [Certbot Apache Guide](https://certbot.eff.org/)
* [SSL Labs Test](https://www.ssllabs.com/)

---

I can also merge this with your **React on Apache** notes so that it becomes a single “Deploy React with SSL” guide.
That way you’ll have one document covering **both deployment & HTTPS setup** in one flow. Would you like me to make that combined version?
