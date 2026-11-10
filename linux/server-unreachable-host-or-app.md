# Server unreachable: is it the host or the app?

Every site on a VPS stops loading. Before touching the app, find out which layer is down. This came from a real
case: a Hostinger VPS running a Laravel API + Next.js (prod and test) went silent for 1 h 49 min and came back by
itself (10 Nov 2026).

## 1. From your own PC, while it is down

Replace the IP and hosts with yours.

```bash
# Every site on the box: code 000 + curl exit 28 = timeout
for h in app.example.com api.example.com; do
  curl -s -o /dev/null -m 15 -w "$h code=%{http_code} t=%{time_total}\n" https://$h/ || echo "$h failed ($?)"
done
```

```powershell
# Windows: ping, the ports, and a known-good site to rule out your own line
ping -n 3 203.0.113.10
foreach ($p in 22,80,443) { "port $p open=" + (Test-NetConnection 203.0.113.10 -Port $p -WarningAction SilentlyContinue).TcpTestSucceeded }
(Test-NetConnection google.com -Port 443 -WarningAction SilentlyContinue).TcpTestSucceeded
tracert -d -w 1500 -h 20 203.0.113.10
```

## 2. Read the result

| What you see | Where the fault is |
| --- | --- |
| Sites give **502 / 504**, SSH works | The app: Nginx is up, the upstream (pm2, PHP-FPM) is not. Fix on the server. |
| Sites give **500** | The app itself (Laravel log, permissions, `.env`). |
| **Everything times out**, SSH included, google.com works | The VPS or its host node is off or frozen. Not your code. |
| Google fails too | Your own line. |
| `tracert` reaches the provider's network and dies at the last hop | Confirms the VPS itself is silent; the route is fine. |

Ping alone proves little (some firewalls drop it). SSH timing out together with 80 and 443 is the strong signal.

## 3. While it is down

- The provider panel (Hostinger hPanel: VPS, the server, Overview): status (Running, Stopped, Suspended), the CPU
  and RAM graphs. Stopped: Start. Suspended: the banner says why (billing, abuse). Running but silent: Restart, or
  the browser terminal.
- The account holder's inbox: a maintenance or incident notice.
- The provider's status page. Note that per-node "emergency maintenance" often never reaches outage trackers, so a
  quiet status page proves nothing.

## 4. Once it is back: how long, and how it ended

Read-only. Paste as one block.

```bash
uptime
journalctl --list-boots --no-pager | tail -2
timeout 20 journalctl -b -1 -n 25 --no-pager
```

- `uptime` small = it rebooted. Days = the VPS stayed up and the network dropped.
- `--list-boots`: the end of boot `-1` to the start of boot `0` is how long it was off.
- The last lines of boot `-1`:
  - shutdown lines ("Stopping ...", "Reached target ... Reboot", "Journal stopped") = a clean restart, someone
    (the provider) shut it down;
  - ordinary activity (cron, firewall lines) that just stops = **hard stop**: host node failure or forced
    power-off. Nothing inside the VPS leads up to it.
- Look for `Out of memory` / `oom-kill` in boot `-1`: then it was your box after all.
- `last` is gone on Ubuntu 26.04 (wtmp replaced by wtmpdb); the journal is the record.

## 5. Once it is back: did anything break

```bash
grep -iE "crash|recover|corrupt|error" /var/log/mysql/error.log | tail -8
cd /var/www/<api folder> && timeout 30 sudo -u www-data php artisan queue:failed | tail -6
supervisorctl status | awk '{print $2}' | sort | uniq -c
pm2 ls
```

- MySQL "XA crash recovery" started and finished is normal after a power cut. "corrupt" is not.
- Failed jobs around the stop time: retry or re-run them.
- Every Supervisor program RUNNING and every pm2 app online (both come back on boot if `pm2 save` and the
  `pm2-<user>` unit were set up).
- Compare the outage window with the scheduler (`routes/console.php`): a daily job whose time fell inside the gap
  did not run and may need running by hand. Hourly jobs catch up by themselves.

## 6. Who asks the provider

Only the account holder can open a case. If the client owns the hosting, send them the evidence and a message to
paste:

- server name and IP, the window in UTC (providers work in UTC), "the journal ends with no shutdown sequence", "no
  notice received";
- ask for the cause, why no notice, what prevents a repeat, and the uptime credit (Hostinger advertises 99.9%,
  about 43 min a month; the credit is on request, not automatic);
- ask for the answer in writing.

Hostinger: support chat in hPanel starts with an AI assistant; ask for a human agent (repeat if needed). Email and
WhatsApp details are on hostinger.com/support ("How to contact Hostinger support"). No phone line.

## 7. Leave a record

Write `INCIDENT-<date>-<what>.md` in the project (timeline in UTC and local time, diagnosis, effect on scheduled
jobs, follow-up) and a troubleshooting row in its `docs/deployment.md`, so the next outage is a five-minute check.
