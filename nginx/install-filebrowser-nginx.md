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

## Conclusion

You have successfully installed and configured FileBrowser with automatic startup and SSL. Now, you can manage server files through a secure web interface.
usefull links

```
https://tonyteaches.tech/filebrowser-tutorial/
https://youtu.be/92rzgw00YMo?si=TwypObrYUkRTUEpK
```
