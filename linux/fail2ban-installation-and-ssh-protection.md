# Installing Fail2Ban on a VPS (SSH Protection Guide)

## Overview

Fail2Ban is a security tool that protects Linux servers by automatically banning IP addresses that repeatedly fail authentication attempts (e.g., SSH brute-force attacks).

This guide configures Fail2Ban to protect SSH access.

---

# Prerequisites

* Ubuntu / Debian VPS
* Root or sudo access
* SSH already working

---

# Step 1: Update Server Packages

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Step 2: Install Fail2Ban

```bash
sudo apt install fail2ban -y
```

---

# Step 3: Enable and Start Fail2Ban

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Verify status:

```bash
sudo systemctl status fail2ban
```

Expected output:

```
Active: active (running)
```

---

# Step 4: Configure SSH Protection Jail

Create the Fail2Ban local configuration file:

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste the following:

```bash
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
```

Save and exit.

---

# Step 5: Restart Fail2Ban

```bash
sudo systemctl restart fail2ban
```

---

# Step 6: Verify SSH Protection Is Active

Check jail status:

```bash
sudo fail2ban-client status sshd
```

Expected output example:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed: X
|  `- File list: /var/log/auth.log
`- Actions
   |- Currently banned: 0
   |- Total banned: X
```

---

# Step 7: Monitor Fail2Ban Logs

View real-time logs:

```bash
sudo tail -f /var/log/fail2ban.log
```

This helps confirm that suspicious login attempts are detected and blocked.

---

# Step 8: Test Manual IP Ban (Optional)

Ban an IP address manually:

```bash
sudo fail2ban-client set sshd banip 1.2.3.4
```

Check status again:

```bash
sudo fail2ban-client status sshd
```

---

# Step 9: Unban an IP Address

Remove a banned IP:

```bash
sudo fail2ban-client set sshd unbanip 1.2.3.4
```

---

# How This Protects Your Server

Fail2Ban monitors:

```
/var/log/auth.log
```

If someone fails SSH login **3 times within 10 minutes**, their IP gets banned for:

```
3600 seconds (1 hour)
```

Protection logic:

```
3 failed attempts → detected
within 10 minutes → trigger
IP banned → 1 hour block
```

This prevents brute-force attacks automatically.

---

# Recommended Production Improvements (Optional but Strongly Advised)

Increase ban time:

```bash
bantime = 86400
```

(24-hour ban)

Reduce allowed retries:

```bash
maxretry = 2
```

Restrict attack window:

```bash
findtime = 300
```

Final hardened example:

```bash
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 2
bantime = 86400
findtime = 300
```

---

# Verify Fail2Ban Starts Automatically After Reboot

Check:

```bash
sudo systemctl is-enabled fail2ban
```

Expected:

```
enabled
```

This confirms protection persists after server restart.

---

# Final Result

After completing this setup:

✅ SSH brute-force attacks automatically blocked
✅ Suspicious IPs banned temporarily
✅ Logs monitored continuously
✅ Protection active after reboot

Your VPS is now significantly more secure against automated login attacks 🔐
