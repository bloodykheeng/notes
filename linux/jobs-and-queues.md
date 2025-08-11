Here’s your **expanded Markdown notes** — now covering **Laravel scheduler, cron jobs, queue management, and Supervisor setup for production** — all in one well-organized guide.

---

# How to Run Laravel Scheduler & Queue Workers on Ubuntu

## 📋 Prerequisites

* Ubuntu server with SSH access.
* Laravel project deployed and working.
* PHP installed and available in the system path.
* Composer installed.
* Database queue driver configured (e.g., MySQL, PostgreSQL, Redis).
* Sudo/root privileges.

---

## 1️⃣ Running the Laravel Scheduler with Cron

Laravel’s scheduler runs tasks defined in `app/Console/Kernel.php`.
To activate it, you need a cron job that triggers Laravel’s scheduler every minute.

### Add Cron Entry

Open the crontab for the current user:

```bash
crontab -e
```

Add:

```bash
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```

**Explanation:**

* `* * * * *` → Runs every minute.
* `cd /path-to-your-project` → Go to Laravel project folder.
* `php artisan schedule:run` → Executes Laravel scheduler.
* `>> /dev/null 2>&1` → Discards output to keep logs clean.

Save and exit.
Verify:

```bash
crontab -l
```

---

## 2️⃣ Testing the Scheduler

Example scheduled task in `app/Console/Kernel.php`:

```php
protected function schedule(Schedule $schedule)
{
    $schedule->command('inspire')->everyMinute();
}
```

Monitor logs:

```bash
tail -f storage/logs/laravel.log
```

---

## 3️⃣ Queue Management in Laravel

### Common Artisan Commands

| Command                        | Description                                    |
| ------------------------------ | ---------------------------------------------- |
| `php artisan queue:work`       | Process new jobs as they come in.              |
| `php artisan queue:listen`     | Like `queue:work`, but reloads after each job. |
| `php artisan queue:retry {id}` | Retry a failed job by ID.                      |
| `php artisan queue:failed`     | List failed jobs.                              |
| `php artisan queue:flush`      | Delete all failed jobs.                        |

💡 **Tip:** You can configure **separate queues** for different job types or reports, allowing different rate limits.

---

## 4️⃣ Running Queue Workers in Production with Supervisor

Using `php artisan queue:work` directly is fine for development, but in production you need a **process monitor** like **Supervisor** to keep workers running.

### Install Supervisor

```bash
sudo apt-get install supervisor
```

### Configure Laravel Worker

Create config:

```bash
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Example configuration:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /path/to/your/project/artisan queue:work database --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/path/to/your/project/worker.log
stopwaitsecs=3600
```

**Explanation:**

* `--sleep=3` → Wait 3 seconds between jobs.
* `--tries=3` → Retry failed jobs up to 3 times.
* `--max-time=3600` → Restart the worker every hour to prevent memory leaks.
* `numprocs=8` → Run 8 parallel workers.
* `stdout_logfile` → Log worker output.

---

## 5️⃣ Start and Manage Supervisor

```bash
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl start laravel-worker:*
```

**Useful commands:**

```bash
sudo supervisorctl status
sudo supervisorctl restart laravel-worker:*
sudo supervisorctl stop laravel-worker:*
```

---

## 6️⃣ Conclusion

With this setup:

* Laravel **scheduler** runs automatically every minute via cron.
* Laravel **queue workers** are managed by Supervisor, ensuring they stay alive after reboots or failures.
* You can process jobs reliably in production with minimal downtime.

---

## 📚 References


* [https://neon.com/guides/laravel-queue-workers-job-processing](https://neon.com/guides/laravel-queue-workers-job-processing)
* [Laravel Task Scheduling](https://laravel.com/docs/scheduling)
* [Laravel Queues](https://laravel.com/docs/queues)
* [Supervisor Docs](http://supervisord.org/)
* [Ubuntu Cron Howto](https://help.ubuntu.com/community/CronHowto)

---

I can also make a **combined deployment note** that includes:

1. Deploy Laravel on Apache.
2. Set up SSL with Let’s Encrypt.
3. Configure cron for scheduler.
4. Install Supervisor for queues.

That way, it’s **one full Laravel production setup guide**. Would you like me to create that next?
