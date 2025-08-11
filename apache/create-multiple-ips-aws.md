Here are clean, structured notes from that second video you shared:

---

## **AWS EC2 — Multiple IPs and Multiple Webpages on Apache**

### **What You’ll Learn**

* Provision **secondary private IP addresses** and **Elastic IPs**.
* Make the OS recognize secondary IPs.
* Create **two web pages** on an Apache server.
* Serve each webpage on a different Elastic IP.

---

### **Key Concept**

* **Elastic IP** = a static IP in AWS that remains even after stopping/restarting the instance.
* EC2 can have **multiple private IPs** and **multiple public IPs**.

---

## **Steps from the Video**

### **1. Create an EC2 Instance with Apache Installed**

* OS: **Amazon Linux 2**
* Instance type: `t2.micro` (Free Tier)
* In **User Data**, paste the following script to auto-install Apache:

```bash
#!/bin/bash
sudo su
yum update -y
yum install -y httpd.x86_64
systemctl start httpd.service
systemctl enable httpd.service
echo "Hello World from $(hostname -f)" > /var/www/html/index.html
```

* Create a security group:

  * Allow ports **80** (HTTP), **443** (HTTPS), and **22** (SSH).

---

### **2. Add a Secondary Private IP**

* AWS Console → Select instance → **Actions → Networking → Manage IP Addresses**.
* Assign a **new private IP** (you can copy the first IP and just change the 4th octet).
* Example:

  ```
  172.31.29.219 (primary)
  172.31.29.220 (secondary)
  ```

---

### **3. Make OS Recognize the New Private IP**

* SSH into EC2:

```bash
ssh -i "key.pem" ec2-user@<public-ip>
sudo su
ip addr li  # Check assigned IPs
```

* If secondary IP isn’t showing:

```bash
sudo ip addr add 172.31.29.220 dev <network_interface>
```

*(Find interface name via `ip addr li`, e.g., `eth0` or `ens5`)*

---

### **4. Allocate Multiple Elastic IPs**

* AWS Console → **Network & Security → Elastic IPs → Allocate**.
* Create 2 Elastic IPs.
* Associate each Elastic IP with your instance, choosing the private IP it should map to.

---

### **5. Create Two Webpages**

```bash
sudo su
cd /var/www/html
mkdir WebPage1 WebPage2

cd WebPage1
nano index.html  # Add HTML content

cd ../WebPage2
nano index.html  # Add HTML content
```

---

### **6. Map Private IPs to Webpages (Apache Virtual Hosts)**

* Edit Apache config:

```bash
cd /etc/httpd/conf
nano httpd.conf
```

* Add inside the **Virtual Hosts** section:

```apache
<VirtualHost 172.31.29.219:80>
    ServerAdmin webmaster@WebPage1.com
    DocumentRoot "/var/www/html/WebPage1"
    ServerName 172.31.29.219
    ErrorLog "logs/WebPage1.com-error_log"
    CustomLog "logs/WebPage1.com-access_log" common
</VirtualHost>

<VirtualHost 172.31.29.220:80>
    ServerAdmin webmaster@WebPage2.com
    DocumentRoot "/var/www/html/WebPage2"
    ServerName 172.31.29.220
    ErrorLog "logs/WebPage2.com-error_log"
    CustomLog "logs/WebPage2.com-access_log" common
</VirtualHost>
```

---

### **7. Restart Apache**

```bash
systemctl stop httpd
systemctl start httpd
```

---

### **8. Test**

* Visit:

  ```
  http://<ElasticIP1>/index.html  → WebPage1
  http://<ElasticIP2>/index.html  → WebPage2
  ```

---

https://youtu.be/Wo0AlRNrFxY?si=uwN034FoG-EFx7ao

If you want, I can now merge **this video’s notes** with the **Laravel + MySQL AWS notes** you gave earlier so you have **one combined AWS setup guide**. That way, it’s all in one place instead of two separate lists.
