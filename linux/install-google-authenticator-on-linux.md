# How to Enable SSH 2FA with Google Authenticator on Ubuntu

## Prerequisites

- A server running Ubuntu.
- OpenSSH server installed and enabled.
- Root or sudo privileges.

## Step 1: Install Google Authenticator

```sh
sudo apt install libpam-google-authenticator
```

## Step 2: Generate Google Authenticator Secret Key

Run the following command as the user you want to secure:

```sh
google-authenticator
```

You'll be prompted to answer several questions:

- **Do you want authentication tokens to be time-based?**
  Enter: `y`

- A QR code and secret key will be generated.

  - **Scan the QR code** using the Google Authenticator app.
  - **Or manually enter** the secret key:
    Example: `Y64KX7QUP55LALQM5FZ7RDORZU`

- Enter the code shown in the app when prompted.

- **Do you want me to update your "/root/.google_authenticator" file?**
  Enter: `y`

- **Disallow multiple uses of the same authentication token?**
  Enter: `y`

- **Increase time skew window to 4 minutes (17 permitted codes)?**
  Enter: `y`

- **Enable rate-limiting (3 login attempts every 30s)?**
  Enter: `y`

**Note:** Save your emergency scratch codes in a secure place. Example:

```
85500427
95647475
68147906
34286348
12299754
```

## Step 3: Configure PAM for SSH

Open the PAM SSH configuration file:

```sh
sudo nano /etc/pam.d/sshd
```

Add the following line at the end:

```
auth required pam_google_authenticator.so
```

## Step 4: Update SSH Configuration

Edit the SSH daemon config:

```sh
sudo nano /etc/ssh/sshd_config
```

Find and modify (or add) this line:
change no to yes

```
KbdInteractiveAuthentication yes
```

> If the line is commented (`#`), remove the `#`.

## Step 5: Restart SSH Service

Apply the changes:

```sh
sudo systemctl restart ssh
```

## Final Step: Logging In with 2FA

When connecting via SSH, you will now be prompted to enter:

1. Your **SSH password**
2. A **verification code** from the **Google Authenticator app**

👉 Simply open the Authenticator app on your Android (or iOS) device, and enter the 6-digit code shown for your server.

That's it — you're securely logged in using two-factor authentication! ✅

## References

- [YouTube Guide](https://youtu.be/8zPR-maW-9I?si=donplIafBY02Dpod)
