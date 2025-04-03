# Updating Code Deployment Guide for Laravel and Next.js on Nginx Server

## Server Access

- **IP Address:** `172.105.24.194`
- **SSH Access:**
  ```sh
  ssh root@172.105.24.194
  ```
- **Password:** <server password>

## Project URLs

- **Frontend:** [https://ppdacms.nwtdemos.com](https://ppdacms.nwtdemos.com)
- **API Backend:** [https://apippdacms.nwtdemos.com](https://apippdacms.nwtdemos.com)

---

## Laravel Backend Deployment

### Navigate to Laravel Project Directory

```sh
cd /var/www/ppda_laravel_api
```

### Pull Latest Code from Git

```sh
git pull
```

**Git Credentials:**

- **Username:** `bloodykheeng`
- **Password:** `<github password>`

---

These other steps are optional

### Install Dependencies (If Necessary)

```sh
composer install --no-dev --optimize-autoloader
```

### Run Migrations (If Needed)

```sh
php artisan migrate --force
```

### Clear Cache and Configurations

```sh
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear
```

### Restart PHP-FPM & Nginx

```sh
systemctl restart php8.1-fpm
systemctl restart nginx
```

---

## Next.js Frontend Deployment

### Navigate to Next.js Project Directory

```sh
cd /var/www/ppda_nextjs_front
```

### Pull Latest Code from Git

```sh
git pull
```

**Git Credentials:**

- **Username:** `bloodykheeng`
- **Password:** `<git hub password>`

### Install Dependencies (If Necessary)

```sh
npm install
```

### Build the Next.js Project

```sh
npm run build
```

### Restart PM2 Process Manager

```sh
pm2 status
pm2 restart ppda_nextjs_front
```

### Delete & Restart PM2 (If Needed)

```sh
pm2 delete ppda_nextjs_front
pm2 start npm --name "ppda_nextjs_front" -- run start
```

---

## Common Issues & Fixes

### Nginx Not Serving Changes

- Restart Nginx:
  ```sh
  systemctl restart nginx
  ```
- Check Nginx Logs:
  ```sh
  tail -f /var/log/nginx/error.log
  ```

### Next.js Not Updating After Deployment

- Run `npm run build` again.
- Restart PM2 process.
- Check logs:
  ```sh
  pm2 logs ppda_nextjs_front
  ```

### Laravel Errors

- Check Laravel logs:
  ```sh
  tail -f /var/www/ppda_laravel_api/storage/logs/laravel.log
  ```
- Check PHP-FPM status:
  ```sh
  systemctl status php8.1-fpm
  ```

---

## Notes

- Always **pull latest code** before deployment.
- Make sure **database migrations** are applied if changes were made.
- **PM2 must be restarted** after deploying Next.js updates.
- **Nginx must be restarted** if there are any Laravel or configuration changes.

### Done! 🚀
