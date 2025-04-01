# How to Secure Nginx with Let's Encrypt SSL on Ubuntu

## Prerequisites

- A server running Ubuntu 20.04 or later.
- Nginx installed and configured.
- A domain name pointed to your server's IP address.

## Step 1: Install Certbot and the Nginx Plugin

```sh
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

## Step 2: Verify Nginx Configuration

Before installing SSL, ensure that your Nginx configuration file is properly set up.

- Set the `server_name` directive only in the configuration that listens on port **80**.
- This is required before installing SSL. After installation, you can modify `server_name` in other configurations.

Example:

```nginx
server {
    listen 80;
    server_name ppdacms.net www.ppdacms.net;
    location / {
        proxy_pass http://localhost:3000;
    }
}
```

Test Nginx configuration:

```sh
sudo nginx -t
```

If there are no errors, reload Nginx:

```sh
sudo systemctl reload nginx
```

## Step 3: Obtain an SSL Certificate

Run the following command to install SSL for your domain:

```sh
sudo certbot --nginx -d ppdacms.net -d www.ppdacms.net
```

Follow the prompts to complete the installation. Certbot will automatically configure Nginx to use SSL.

### Installing SSL for Additional Domains

If you have multiple domains, install SSL certificates separately:

```sh
sudo certbot --nginx -d ppdacms.nwtdemos.com -d www.ppdacms.nwtdemos.com
sudo certbot --nginx -d apippdacms.nwtdemos.com -d www.apippdacms.nwtdemos.com
sudo certbot --nginx -d apicms.nwtdemos.com -d www.apicms.nwtdemos.com
```

## Step 4: Verify SSL Certificate

After installation, check if SSL is working by visiting your domain via `https://yourdomain.com`.

## Step 5: Renew SSL Certificates Automatically

Certbot automatically renews certificates, but you can manually test renewal with:

```sh
sudo certbot renew --dry-run
```

## Step 6: Revoke an SSL Certificate (If Needed)

If you need to revoke an SSL certificate, use the following command:

```sh
sudo certbot revoke --cert-name ppdacms.net
sudo certbot revoke --cert-name www.ppdacms.net
```

## Step 7: Locate and Edit Nginx Configurations

If you need to modify configurations, navigate to:

```sh
cd /etc/nginx/sites-available/
```

Edit the corresponding file:

```sh
sudo nano your-config-file
```

## Final Step: Restart Nginx

```sh
sudo systemctl restart nginx
```

Your SSL certificates are now installed and configured successfully! 🚀

helpfull resources
https://docs.chaicode.com/home-for-programmers/

https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-20-04
