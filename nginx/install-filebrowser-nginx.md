# How to Access Server Files in a Web Browser with FileBrowser

## Introduction

FileBrowser is an open-source file manager that allows you to browse and manage files on your server through a web interface. This guide will walk you through the installation, configuration, and setting up of FileBrowser to run persistently on your server.

---

## Prerequisites

- SSH access to your server
- Nginx installed
- `ufw` (Uncomplicated Firewall) configured properly

---

## Step 1: Install FileBrowser

First, fetch and install FileBrowser using the following command:

```bash
curl -fsSL https://raw.githubusercontent.com/filebrowser/get/master/get.sh | bash
```

Once installed, you can start FileBrowser using:

```bash
filebrowser -p 8080 -a <YOUR_SERVER_IP> -r /var/www

filebrowser -p 8080 -a 134.122.27.34 -r /var/www
```

- `-p 8080` specifies the port.
- `-a <YOUR_SERVER_IP>` specifies the server IP.
- `-r /var/www` defines the root directory for FileBrowser.

Now, open your browser and go to:
 
```
http://<YOUR_SERVER_IP>:8080

134.122.27.34:8080
```

**Default Credentials:**

* Username: `admin`
* Password: `admin`

You can now manage files in `/var/www` through the web interface. Your instance is now up and running. File Browser will automatically bootstrap a database, in which the configuration and the users are stored. You can find the address in which your instance is running, as well as the randomly generated password for the user `admin`, in the console logs.

⚠ **Important:**

* The automatically generated password for the `admin` user is only displayed **once** in the logs the first time you run File Browser.
* **Currently, this password is stored in the server logs**, so make sure to copy it and store it securely.
* If you lose it, you will need to manually delete the database and restart File Browser to regenerate credentials.



---

## When Port 8080 Is Already Taken

8080 is the most contested port on a server. Laravel **Reverb** defaults to it,
and so do Jenkins, Tomcat and most admin tools. If FileBrowser will not start, or
starts and the page never loads, work through this in order.

### 1. See what already holds it

```bash
ss -ltnp | grep :8080
```

If it names `php`, that is Reverb. Check which apps are running it:

```bash
sudo supervisorctl status
sudo ls -la /etc/supervisor/conf.d/
```

### 2. Pick a different port instead of fighting for 8080

Taking 8080 off a running app means stopping something that is serving users.
Give FileBrowser its own port:

```bash
sudo ufw allow 8081/tcp
sudo ufw status
filebrowser -p 8081 -a <YOUR_SERVER_IP> -r /var/www
```

Then browse to `http://<YOUR_SERVER_IP>:8081`.

To remove a port when you are done with it:

```bash
sudo ufw delete allow 8081/tcp
```

### 3. Page will not load, even though ufw allows the port

**Do not assume the port is the problem.** Test whether the port is reachable
from outside the server, from your own machine:

```powershell
Test-NetConnection 80.85.84.128 -Port 8081     # Windows
```

```bash
nc -zv 80.85.84.128 8081                        # macOS / Linux
```

If that fails while `ss -ltnp` on the server shows FileBrowser listening and
`ufw status` allows the port, then **something in front of the server is
blocking it** — a second firewall you did not configure.

### 4. The second firewall

Cloud providers put their own firewall in front of the machine, outside the OS
entirely, and `ufw` knows nothing about it. A port is only open when **both**
firewalls allow it.

#### Linode (Akamai)

`cloud.linode.com` → **Firewalls** → your firewall (e.g. `alpha-firewall`) →
**Rules** tab → **Inbound Rules** → **Add an Inbound Rule**.

Fill it in exactly like this:

| Field | Value |
|---|---|
| Preset | leave on `Select a rule preset...` — there is no preset for 8080 |
| Label | `accept-inbound-HTTP-8080` (letters, numbers and dashes only) |
| Description | optional |
| Protocol | `TCP` |
| Ports | `Custom` |
| Custom Port Range | `8080` |
| Sources | `All IPv4, All IPv6` |
| Action | `Accept` |

**Add Changes**, then **Save Changes** on the Rules page — the drawer only stages
the rule, and closing the page without saving loses it.

A working set of inbound rules ends up looking like:

```
accept-inbound-ssh         Accept  TCP   22     All IPv4, All IPv6
accept-inbound-icmp        Accept  ICMP  -      All IPv4, All IPv6
accept-inbound-HTTP        Accept  TCP   80     All IPv4, All IPv6
accept-inbound-HTTPS       Accept  TCP   443    All IPv4, All IPv6
accept-inbound-HTTP-8080   Accept  TCP   8080   All IPv4, All IPv6
```

Two things that catch people out:

- **The firewall must be attached to the machine.** Check the **Linodes** tab on
  the firewall — if your server is not listed there, the rules apply to nothing.
- **The default inbound policy is Drop**, so anything without an explicit Accept
  rule is blocked. That is why 22, 80 and 443 each need a rule of their own, and
  why a new port needs one too.

> Using a port other than 8080, say 8081? Make the rule say `8081` in both the
> label and the port range. I once added the rule for **8080** while FileBrowser
> was running on **8081**, and of course nothing changed.

#### Other providers

| Provider | Where |
|---|---|
| DigitalOcean | Networking → Firewalls → Inbound Rules |
| AWS | EC2 → Security Groups → Inbound rules |
| Azure | Network Security Group → Inbound security rules |
| Google Cloud | VPC network → Firewall → Create firewall rule |

> **What this cost me once.** FileBrowser would not load on 8081 and I assumed the
> port was at fault, so I stopped every Reverb app on the box trying to free 8080.
> Neither port was the problem: `ufw` allowed them and the **Linode** Cloud
> Firewall did not. The tell is that stopping apps changed nothing — when a port
> is genuinely in use you get an *address already in use* error, not a page that
> silently never loads.

### 5. If you must stop Reverb to reuse 8080

```bash
sudo supervisorctl stop alphapi.nwtdemos.com-reverb:*
sudo supervisorctl stop sdsbetapi.nwtdemos.com-reverb:*
ss -ltnp | grep :8080          # confirm nothing is left
```

Killing the process by PID does **not** work — Supervisor has `autorestart=true`
and puts it straight back. Use `supervisorctl stop`.

Start them again afterwards, and do not forget to, because a program left in
`STOPPED` stays down and does not come back on reboot:

```bash
sudo supervisorctl start alphapi.nwtdemos.com-reverb:*
sudo supervisorctl start sdsbetapi.nwtdemos.com-reverb:*
sudo supervisorctl status
```

---

## Step 2: Persistent FileBrowser Setup

To ensure FileBrowser starts on boot, create a configuration file:

```bash
nano /etc/filebrowser.json
```

Add the following content:

```json
{
  "port": 8080,
  "baseURL": "",
  "address": "<YOUR_SERVER_IP>",
  "log": "stdout",
  "database": "/etc/filebrowser.db",
  "root": "/var/www"
}


{
  "port": 8080,
  "baseURL": "",
  "address": "134.122.27.34",
  "log": "stdout",
  "database": "/etc/filebrowser.db",
  "root": "/var/www/"
}
```

**Note:** Remove any existing `filebrowser.db` file before restarting FileBrowser.
in the home directory or root the first tme we ran file browser manually , it created a file filebrowser.db we can remove it since now we are gona be storing it in /etc/filebrowser.db

Start FileBrowser using:
so now to start file browser we just need to specify the absolute file to our json file

```bash
filebrowser -c /etc/filebrowser.json
```

---

## Step 3: Configure Firewall for Port 8080

If the firewall is enabled, allow port 8080:

```bash
sudo ufw allow 8080/tcp
sudo ufw reload
sudo ufw status
```

> `ufw` is only the firewall **on** the machine. If the port still cannot be
> reached from outside, your cloud provider has a second one in front of it —
> see [When Port 8080 Is Already Taken](#when-port-8080-is-already-taken).

---

## Step 4: Run FileBrowser as a Service

lets make file browser run when the server runs
we shall use system demeans where we shall put our command that starts file server and it will be run whenever the system reboots again

Create a systemd service file:

```bash
nano /etc/systemd/system/filebrowser.service
```

Add the following content:

```ini
[Unit]
Description=File Browser
After=network.target

[Service]
ExecStart=/usr/local/bin/filebrowser -c /etc/filebrowser.json

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
systemctl enable filebrowser.service
systemctl start filebrowser.service
systemctl status filebrowser.service
```

Now, FileBrowser will start automatically on server reboot.

access it easily using your ip
http://172.105.24.194:8080

# How to Unzip Files in FileBrowser

first check if unzip is installed on ur system

```bash
unzip -v
```

if not installed then istall it using this comand

```
sudo apt install unzip -y
```

then after come back to file browser

## Step 1: Enable the `unzip` Command for Admin Users

https://github.com/filebrowser/filebrowser/discussions/1585

1. Navigate to **Settings > User Management**.
2. Select the **admin** user and click the **edit (pencil icon)**.
3. Scroll down to **Commands**.
4. Type `unzip` in the Commands field.
5. Click **Update** to save changes.

## Step 2: Unzip Files Using FileBrowser

1. Use FileBrowser to locate your **.zip** file.
2. Open the **console** (click the `< >` toolbar button).
3. Type the following command:
   ```sh
   unzip filename.zip
   ```
   _(Replace `filename.zip` with the actual file name)_
4. Watch the unzip progress – you're done!

## Step 3 (Optional): Set Commands for New Users

1. Go to **Settings > Global Settings**.
2. Scroll down to **User Default Settings**.
3. Under **Commands**, type `unzip`.
4. Click **Update** to apply this setting.
5. When you create a **new user**, the `unzip` command will be automatically assigned and available.

### Note:

- You are limited to commands available on the host system.
- If using the `hurlenko/filebrowser` Docker container, the host system is **BusyBox**, which includes a predefined list of commands.

💡 _It would be great to have an "Unzip" button in FileBrowser, but this would require changes to the application itself._

---

## Step 5: Install SSL Certificate for Secure Access

This stepp is optional

To enable HTTPS, install Certbot:

```bash
sudo apt install snapd
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
ln -s /snap/bin/certbot /usr/bin/certbot
```

Generate an SSL certificate:

```bash
certbot certonly --nginx
```

Enter your domain when prompted (e.g., `tonys.surf`). Certbot will generate certificates at:

```
/etc/letsencrypt/live/tonys.surf/fullchain.pem
/etc/letsencrypt/live/tonys.surf/privkey.pem
```

Modify `/etc/filebrowser.json` to include SSL:

```json
{
  "port": 8080,
  "baseURL": "",
  "address": "<YOUR_SERVER_IP>",
  "log": "stdout",
  "database": "/etc/filebrowser.db",
  "root": "/var/www",
  "cert": "/etc/letsencrypt/live/tonys.surf/fullchain.pem",
  "key": "/etc/letsencrypt/live/tonys.surf/privkey.pem"
}
```

Restart FileBrowser:

```bash
systemctl restart filebrowser.service
systemctl status filebrowser.service
```

Now you can access FileBrowser securely at:

```
https://tonys.surf:8080
```

---

## Troubleshooting: SSH Disconnected While FileBrowser Was Running

If your SSH session disconnected while FileBrowser was running manually (not as a service), you will get this error when you reconnect and try to start it again:

```
Error: timeout
```

This happens because the old FileBrowser process is still alive in the background holding the port or the database lock.

### Fix — Kill the old process first

Find the running FileBrowser process:

```bash
ps aux | grep filebrowser
```

Output will look like:

```
root  363307  0.0  0.6  1249356  25000  pts/3  Sl+  12:18  0:01  filebrowser -p 8080 -a 172.233.58.26 -r /var/www
root  366139  0.0  0.0  6676     2176   pts/0  S+   13:04  0:00  grep --color=auto filebrowser
```

Kill the FileBrowser process (use the PID from the first line, ignore the `grep` line):

```bash
kill -9 363307
```

Now start FileBrowser again:

```bash
filebrowser -p 8080 -a YOUR_SERVER_IP -r /var/www
# or with config file
filebrowser -c /etc/filebrowser.json
```

### Avoid this problem — use a systemd service

If FileBrowser is set up as a systemd service (Step 4), this problem never happens. The service survives SSH disconnects and restarts automatically on reboot. Always prefer the service over running it manually.

```bash
systemctl start filebrowser.service
systemctl status filebrowser.service
```

---

## Conclusion

You have successfully installed and configured FileBrowser with automatic startup and SSL. Now, you can manage server files through a secure web interface.
usefull links

```
https://tonyteaches.tech/filebrowser-tutorial/
https://youtu.be/92rzgw00YMo?si=TwypObrYUkRTUEpK
```
